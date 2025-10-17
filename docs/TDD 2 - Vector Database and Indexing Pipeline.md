# Technical Design Document 2: Vector Database and Indexing Pipeline

**Version:** 2.1
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
* **Privileges required:** The `SEMANTIC_SEARCH` user will need `CREATE TABLE`, `CREATE SESSION`, `CREATE INDEX`, and `UNLIMITED TABLESPACE`.

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
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.CATALOG_ITEM_ID IS 'Foreign key to the S_PROD_INT table for the final catalog item.';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.CATALOG_PATH IS 'Full human-readable path of the catalog item.';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.FULL_NARRATIVE IS 'The complete aggregated text used to generate the vector.';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.NARRATIVE_VECTOR IS 'The numerical vector representation of the narrative.';
```

## 3. Embedding Model
* **Service:** OCI Generative AI Service
* **Model:** Cohere Embed English v3.0 (or latest equivalent)
* **Vector Dimension:** 1024

## 4. Indexing Pipeline
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
    # (Implementation details for calling OCI GenAI embed_text endpoint)
    # ...
    pass

def upsert_into_db(cursor, data_batch):
    """Uses MERGE for idempotent inserts/updates."""
    # Convert list of tuples to list of dictionaries for executemany
    data_dicts = [
        {"sr_id": r[0], "cat_id": r[1], "cat_path": r[2], "narrative": r[3], "vector": r[4]}
        for r in data_batch
    ]
    cursor.executemany("""
        MERGE INTO SIEBEL_KNOWLEDGE_VECTORS t
        USING (
            SELECT :sr_id AS SR_ID, :cat_id AS CATALOG_ITEM_ID, :cat_path AS CATALOG_PATH,
                   :narrative AS FULL_NARRATIVE, :vector AS NARRATIVE_VECTOR
            FROM DUAL
        ) s ON (t.SR_ID = s.SR_ID)
        WHEN MATCHED THEN
            UPDATE SET t.FULL_NARRATIVE = s.FULL_NARRATIVE,
                       t.NARRATIVE_VECTOR = s.NARRATIVE_VECTOR,
                       t.CATALOG_ITEM_ID = s.CATALOG_ITEM_ID,
                       t.CATALOG_PATH = s.CATALOG_PATH
        WHEN NOT MATCHED THEN
            INSERT (SR_ID, CATALOG_ITEM_ID, CATALOG_PATH, FULL_NARRATIVE, NARRATIVE_VECTOR)
            VALUES (s.SR_ID, s.CATALOG_ITEM_ID, s.CATALOG_PATH, s.FULL_NARRATIVE, s.NARRATIVE_VECTOR)
    """, data_dicts)

# --- Main Processing Logic ---
# 1. Read data from CSV into a pandas DataFrame.
# 2. Iterate through the DataFrame in batches.
# 3. For each batch:
#    a. Clean the 'FULL_NARRATIVE_WITH_CONTEXT' text.
#    b. Call get_embedding_with_retry() to get vectors.
#    c. Prepare the data batch for database insertion.
#    d. Call upsert_into_db() to load the data.
# 4. Commit the transaction.
```

## 5. Vector Index Creation
After the initial data load is complete, a vector index must be created to enable fast similarity searches. This is a critical step for performance.

### 5.1. Index DDL
The index should be created on the `NARRATIVE_VECTOR` column. We will use the `HNSW` (Hierarchical Navigable Small World) index type, which provides an excellent balance of search speed and accuracy.

```sql
CREATE VECTOR INDEX SIBL_KNOW_VEC_HNSW_IDX ON SEMANTIC_SEARCH.SIEBEL_KNOWLEDGE_VECTORS (NARRATIVE_VECTOR)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE;
```
* **`ORGANIZATION INMEMORY NEIGHBOR GRAPH`**: Specifies that this is a graph-based index that should be cached in memory for the best performance.
* **`DISTANCE COSINE`**: The distance metric must match the one used in the search query. Cosine similarity is recommended for document embeddings.

## 6. Continuous Learning Strategy
The indexing pipeline script will be scheduled to run nightly using a `cron` job or Oracle Scheduler. It will process the small CSV file from the incremental extraction job, using the `MERGE` statement to idempotently add new records or update existing ones if their narrative has changed.
