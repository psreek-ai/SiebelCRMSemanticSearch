# Technical Design Document 2: Vector Database and Indexing Pipeline

**Version:** 2.0
**Date:** 2025-10-16
**Owner:** AI / Data Engineering Team

## 1. Objective
This document provides the technical specifications for setting up the Oracle 23ai database to function as a vector store and for building the pipeline that populates it with embeddings from the extracted Siebel data.

## 2. Vector Database Setup

### 2.1. Environment
* **Database:** Oracle Database 23ai (Enterprise Edition)
* **Deployment:** OCI Database Service or On-Premise

### 2.2. Schema and Tables
A dedicated schema (e.g., `SEMANTIC_SEARCH`) will be created to host the vector table and related objects.
* **Privileges required:** The `SEMANTIC_SEARCH` user will need `CREATE TABLE`, `CREATE SESSION`, and `UNLIMITED TABLESPACE`.

#### 2.2.1. Vector Table DDL
The following table will store the original text data, its vector embedding, and relevant metadata.

```sql
CREATE TABLE SEMANTIC_SEARCH.SIEBEL_KNOWLEDGE_VECTORS (
    SR_ID                   VARCHAR2(15) NOT NULL PRIMARY KEY,
    CATALOG_ITEM_ID         VARCHAR2(15) NOT NULL,
    CATALOG_PATH            VARCHAR2(4000),
    FULL_NARRATIVE          CLOB,
    NARRATIVE_VECTOR        VECTOR(1024, FLOAT64), -- Dimension depends on the chosen model
    CREATED_DATE            DATE DEFAULT SYSDATE
);

-- Add comments for clarity
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.SR_ID IS 'Primary key from SIEBEL.S_SRV_REQ.ROW_ID.';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.CATALOG_ITEM_ID IS 'Foreign key to the S_PROD_INT table.';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.CATALOG_PATH IS 'Full human-readable path of the catalog item.';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.FULL_NARRATIVE IS 'The complete aggregated text used to generate the vector.';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.NARRATIVE_VECTOR IS 'The numerical vector representation of the narrative.';
````

## 3\. Embedding Model

  * **Service:** OCI Generative AI Service
  * **Model:** Cohere Embed English v3.0 (or latest equivalent)
  * **Vector Dimension:** 1024
  * **Cost Model:** Priced per 1,000 characters of input text. The initial bulk load will be the primary cost driver. Incremental daily updates will have a minimal cost.

## 4\. Indexing Pipeline

The indexing pipeline is a process that reads the CSV files from the data extraction step, generates embeddings, and inserts the data into the vector table.

### 4.1. Pipeline Logic (Python Example)

This will be implemented as a Python script using the `oracledb`, `pandas`, and `oci` SDKs.

```python
# Conceptual Python script for the indexing pipeline
import pandas as pd
import oracledb
import oci
import json
import re
import time

# --- Configuration & Initialization ---
# (Initialize OCI GenAI client and Oracle DB connection securely)

def clean_text(text):
    """Applies pre-processing steps to the text."""
    if not isinstance(text, str):
        return ""
    # 1. Strip HTML tags
    text = re.sub('<[^<]+?>', '', text)
    # 2. Remove boilerplate signatures/greetings
    text = re.sub(r'(?i)dear.*?,|regards,|thank you,', '', text)
    # 3. Normalize whitespace
    text = re.sub(r'\s+', ' ', text).strip()
    return text

def get_embedding_with_retry(text_list, client, max_retries=3):
    """Calls the embedding service with exponential backoff."""
    for attempt in range(max_retries):
        try:
            # Logic to call OCI GenAI embed_text endpoint
            # Returns a list of vectors
            return embeddings
        except oci.exceptions.ServiceError as e:
            if e.status == 429: # Too Many Requests
                time.sleep(2 ** attempt) # Exponential backoff
            else:
                raise e
    raise Exception("Failed to get embeddings after multiple retries.")

def upsert_into_db(cursor, data_batch):
    """Uses MERGE for idempotent inserts/updates."""
    sql = """
    MERGE INTO SIEBEL_KNOWLEDGE_VECTORS t
    USING (SELECT :1 AS SR_ID, :2 AS CATALOG_ITEM_ID, :3 AS CATALOG_PATH, :4 AS FULL_NARRATIVE, :5 AS NARRATIVE_VECTOR FROM DUAL) s
    ON (t.SR_ID = s.SR_ID)
    WHEN MATCHED THEN
        UPDATE SET t.CATALOG_ITEM_ID = s.CATALOG_ITEM_ID, t.CATALOG_PATH = s.CATALOG_PATH, t.FULL_NARRATIVE = s.FULL_NARRATIVE, t.NARRATIVE_VECTOR = s.NARRATIVE_VECTOR
    WHEN NOT MATCHED THEN
        INSERT (SR_ID, CATALOG_ITEM_ID, CATALOG_PATH, FULL_NARRATIVE, NARRATIVE_VECTOR)
        VALUES (s.SR_ID, s.CATALOG_ITEM_ID, s.CATALOG_PATH, s.FULL_NARRATIVE, s.NARRATIVE_VECTOR)
    """
    cursor.executemany(sql, data_batch, batcherrors=True)
    # Handle batch errors if necessary

def process_csv_file(file_path):
    # Iterate through the DataFrame in chunks
    for chunk in pd.read_csv(file_path, chunksize=100):
        # 1. Clean the text data
        chunk['CLEANED_NARRATIVE'] = chunk['FULL_NARRATIVE_WITH_CONTEXT'].apply(clean_text)
        
        # 2. Get embeddings for the cleaned text
        embeddings = get_embedding_with_retry(chunk['CLEANED_NARRATIVE'].tolist(), oci_genai_client)

        # 3. Prepare data for batch upsert
        # ... logic to prepare data tuple ...

        # 4. Upsert the batch into the database
        upsert_into_db(db_cursor, data_to_insert)
        db_connection.commit()
```

## 5\. Vector Index Creation

After the data is loaded, a high-performance vector index must be created to enable fast similarity searches.

### 5.1. Index DDL

An `HNSW` index is recommended. The `efConstruction` and `M` parameters can be tuned to balance indexing time vs. search accuracy. Higher values lead to better accuracy but slower index builds.

```sql
CREATE VECTOR INDEX siebel_vector_idx ON SEMANTIC_SEARCH.SIEBEL_KNOWLEDGE_VECTORS (NARRATIVE_VECTOR)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE
WITH PARAMETERS ('efConstruction=400, M=64');
```

  * **`efConstruction`**: Size of the dynamic list for graph construction.
  * **`M`**: Maximum number of neighbors per node in the graph.

## 6\. Continuous Learning Strategy

The indexing pipeline script will be scheduled to run nightly using a `cron` job or Oracle Scheduler. It will process the small CSV file from the incremental extraction job, using the `MERGE` statement to idempotently add new records or update existing ones if their narrative has changed.
