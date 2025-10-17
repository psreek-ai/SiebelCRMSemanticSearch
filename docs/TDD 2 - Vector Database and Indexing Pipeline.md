# Technical Design Document 2: Vector Database and Indexing Pipeline

**Version:** 2.2
**Date:** 2025-10-17
**Owner:** AI / Data Engineering Team

## 1. Objective
This document provides the technical specifications for generating vector embeddings from the staged Siebel narratives and populating the **Oracle Autonomous Database** vector store. We will use **PL/SQL and DBMS_CLOUD** to call the OCI Generative AI service directly from the database, eliminating the need for external Python scripts. The pre-configured DBMS_CLOUD package in Autonomous Database simplifies OCI service integration.

## 2. Architecture Overview

### 2.1. Processing Flow
```
SIEBEL_NARRATIVES_STAGING (PROCESSED_FLAG='N')
      ↓
PL/SQL Procedure: GENERATE_EMBEDDINGS_BATCH
      ↓
Call OCI GenAI Service via DBMS_CLOUD
      ↓
Insert into SIEBEL_KNOWLEDGE_VECTORS
      ↓
Update PROCESSED_FLAG='Y'
      ↓
Create HNSW Vector Index
```

## 3. Implementation Steps

### Step 1: Create OCI Credentials in Oracle Autonomous Database

**Executor:** Database Administrator on Oracle Autonomous Database  
**Duration:** 20 minutes

#### 1.1. Obtain OCI API Credentials

Before proceeding, gather the following from OCI Console:

1. **User OCID**: Navigate to Identity → Users → Your User → Copy OCID
2. **Tenancy OCID**: Profile menu → Tenancy → Copy OCID
3. **Fingerprint**: Generate an API key pair and note the fingerprint
4. **Private Key**: Download the private key file (`.pem` format)
5. **Region**: Note your OCI region (e.g., `us-ashburn-1`)
6. **Compartment OCID**: Navigate to Identity → Compartments → Your Compartment → Copy OCID

#### 1.2. Create OCI Credential in Database
```sql
-- Connect as SEMANTIC_SEARCH user

-- Read the private key content (you'll need to provide the actual key text)
-- Replace the placeholders with your actual OCI credentials

BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'OCI_GENAI_CREDENTIAL',
    user_ocid       => 'ocid1.user.oc1..aaaaaaaa...your_user_ocid',
    tenancy_ocid    => 'ocid1.tenancy.oc1..aaaaaaaa...your_tenancy_ocid',
    private_key     => '-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC...
...your_private_key_content_here...
...
-----END PRIVATE KEY-----',
    fingerprint     => 'a1:b2:c3:d4:e5:f6:...'
  );
END;
/

-- Verify credential creation
SELECT credential_name, username, enabled
FROM USER_CREDENTIALS
WHERE credential_name = 'OCI_GENAI_CREDENTIAL';
```

**Security Best Practice:**
- Store the private key securely
- Rotate keys periodically
- Use dedicated service accounts with minimal permissions

#### 1.3. Test OCI API Connectivity
```sql
-- Test a simple API call to verify connectivity
DECLARE
  l_response CLOB;
  l_request_body CLOB;
BEGIN
  l_request_body := '{
    "servingMode": {"servingType": "ON_DEMAND"},
    "compartmentId": "ocid1.compartment.oc1..aaaaaaaa...your_compartment_ocid",
    "inputs": ["Test connection"]
  }';
  
  l_response := DBMS_CLOUD.SEND_REQUEST(
    credential_name => 'OCI_GENAI_CREDENTIAL',
    uri => 'https://inference.generativeai.us-ashburn-1.oci.oraclecloud.com/20231130/actions/embedText',
    method => 'POST',
    body => UTL_RAW.CAST_TO_RAW(l_request_body)
  );
  
  DBMS_OUTPUT.PUT_LINE('Response: ' || SUBSTR(l_response, 1, 200));
  DBMS_OUTPUT.PUT_LINE('SUCCESS: OCI API is accessible');
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('ERROR: ' || SQLERRM);
    RAISE;
END;
/
```

**Troubleshooting:**
- **ORA-20000**: Invalid credentials → Check OCID values and fingerprint
- **Network timeout**: Verify Network ACL is configured (from TDD 1)
- **401 Unauthorized**: Check private key format and fingerprint
- **404 Not Found**: Verify region in the URI matches your OCI region

---

### Step 2: Create Vector Table

**Executor:** Database Administrator on Oracle Autonomous Database  
**Duration:** 5 minutes

#### 2.1. Create the Vector Table DDL
```sql
-- Connect as SEMANTIC_SEARCH user

CREATE TABLE SIEBEL_KNOWLEDGE_VECTORS (
    SR_ID                   VARCHAR2(15) NOT NULL PRIMARY KEY,
    CATALOG_ITEM_ID         VARCHAR2(15) NOT NULL,
    CATALOG_PATH            VARCHAR2(4000),
    FULL_NARRATIVE          CLOB,
    NARRATIVE_VECTOR        VECTOR(1024, FLOAT64), -- Cohere Embed v3.0 uses 1024 dimensions
    EMBEDDING_MODEL         VARCHAR2(100) DEFAULT 'cohere.embed-english-v3.0',
    CREATED_DATE            DATE DEFAULT SYSDATE,
    LAST_UPDATED            DATE DEFAULT SYSDATE
);

-- Create indexes
CREATE INDEX IDX_KNOWLEDGE_CATALOG_ID ON SIEBEL_KNOWLEDGE_VECTORS(CATALOG_ITEM_ID);

-- Add comments for clarity
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.SR_ID IS 'Primary key from SIEBEL.S_SRV_REQ.ROW_ID';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.CATALOG_ITEM_ID IS 'Foreign key to the S_PROD_INT table for the final catalog item';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.CATALOG_PATH IS 'Full human-readable path of the catalog item';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.FULL_NARRATIVE IS 'The complete aggregated text used to generate the vector';
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.NARRATIVE_VECTOR IS 'The numerical vector representation of the narrative (1024 dimensions)';

-- Verify table creation
DESC SIEBEL_KNOWLEDGE_VECTORS;
```

---

### Step 3: Create Embedding Generation Procedure

**Executor:** PL/SQL Developer  
**Duration:** 30 minutes

#### 3.1. Create Helper Function for Text Cleaning
```sql
-- Function to clean and prepare text for embedding
CREATE OR REPLACE FUNCTION CLEAN_TEXT_FOR_EMBEDDING(
    p_text IN CLOB
) RETURN CLOB
IS
    l_clean_text CLOB;
BEGIN
    l_clean_text := p_text;
    
    -- Remove HTML tags
    l_clean_text := REGEXP_REPLACE(l_clean_text, '<[^>]+>', ' ');
    
    -- Remove email signatures and common boilerplate
    l_clean_text := REGEXP_REPLACE(l_clean_text, '(Dear|Hi|Hello)[\s\w,]+', '', 1, 0, 'i');
    l_clean_text := REGEXP_REPLACE(l_clean_text, '(Regards|Best|Thanks|Thank you)[,\s\w]*$', '', 1, 0, 'i');
    
    -- Normalize whitespace
    l_clean_text := REGEXP_REPLACE(l_clean_text, '\s+', ' ');
    l_clean_text := TRIM(l_clean_text);
    
    -- Truncate to 5000 characters if too long (OCI API limit consideration)
    IF LENGTH(l_clean_text) > 5000 THEN
        l_clean_text := SUBSTR(l_clean_text, 1, 5000);
    END IF;
    
    RETURN l_clean_text;
END CLEAN_TEXT_FOR_EMBEDDING;
/
```

#### 3.2. Create Main Embedding Generation Procedure
```sql
CREATE OR REPLACE PROCEDURE GENERATE_EMBEDDINGS_BATCH(
    p_batch_size IN NUMBER DEFAULT 10,
    p_compartment_id IN VARCHAR2 DEFAULT 'ocid1.compartment.oc1..aaaaaaaa...your_compartment_ocid',
    p_region IN VARCHAR2 DEFAULT 'us-ashburn-1'
) AS
    -- Variables for API call
    l_api_url VARCHAR2(500);
    l_request_body CLOB;
    l_response CLOB;
    l_vector_json CLOB;
    l_vector VECTOR(1024, FLOAT64);
    
    -- Counters
    v_processed_count NUMBER := 0;
    v_error_count NUMBER := 0;
    v_total_pending NUMBER;
    
    -- Cursor for unprocessed narratives
    CURSOR c_unprocessed IS
        SELECT 
            n.SR_ID,
            n.SR_NUM,
            n.CATALOG_ITEM_ID,
            n.CATALOG_PATH,
            n.FULL_NARRATIVE
        FROM 
            SIEBEL_NARRATIVES_STAGING n
        WHERE 
            n.PROCESSED_FLAG = 'N'
            AND ROWNUM <= p_batch_size
        FOR UPDATE SKIP LOCKED; -- Prevent concurrent processing conflicts
        
BEGIN
    -- Set API URL based on region
    l_api_url := 'https://inference.generativeai.' || p_region || '.oci.oraclecloud.com/20231130/actions/embedText';
    
    -- Get count of pending records
    SELECT COUNT(*) INTO v_total_pending
    FROM SIEBEL_NARRATIVES_STAGING
    WHERE PROCESSED_FLAG = 'N';
    
    DBMS_OUTPUT.PUT_LINE('Starting batch processing...');
    DBMS_OUTPUT.PUT_LINE('Total pending records: ' || v_total_pending);
    DBMS_OUTPUT.PUT_LINE('Batch size: ' || p_batch_size);
    DBMS_OUTPUT.PUT_LINE('-----------------------------------');
    
    -- Process each record
    FOR rec IN c_unprocessed LOOP
        BEGIN
            -- Clean the text
            DECLARE
                l_clean_text CLOB;
            BEGIN
                l_clean_text := CLEAN_TEXT_FOR_EMBEDDING(rec.FULL_NARRATIVE);
                
                -- Build API request body
                SELECT JSON_OBJECT(
                    'servingMode' VALUE JSON_OBJECT('servingType' VALUE 'ON_DEMAND'),
                    'compartmentId' VALUE p_compartment_id,
                    'inputs' VALUE JSON_ARRAY(l_clean_text)
                ) INTO l_request_body FROM DUAL;
                
                -- Call OCI Generative AI service
                l_response := DBMS_CLOUD.SEND_REQUEST(
                    credential_name => 'OCI_GENAI_CREDENTIAL',
                    uri => l_api_url,
                    method => 'POST',
                    body => UTL_RAW.CAST_TO_RAW(l_request_body)
                );
                
                -- Parse the response to extract the vector
                SELECT JSON_VALUE(l_response, '$.embeddings[0]' RETURNING VECTOR(1024, FLOAT64))
                INTO l_vector
                FROM DUAL;
                
                -- Insert/Update in the vector table
                MERGE INTO SIEBEL_KNOWLEDGE_VECTORS kv
                USING (
                    SELECT 
                        rec.SR_ID AS SR_ID,
                        rec.CATALOG_ITEM_ID AS CATALOG_ITEM_ID,
                        rec.CATALOG_PATH AS CATALOG_PATH,
                        rec.FULL_NARRATIVE AS FULL_NARRATIVE,
                        l_vector AS NARRATIVE_VECTOR
                    FROM DUAL
                ) src ON (kv.SR_ID = src.SR_ID)
                WHEN MATCHED THEN
                    UPDATE SET
                        kv.NARRATIVE_VECTOR = src.NARRATIVE_VECTOR,
                        kv.FULL_NARRATIVE = src.FULL_NARRATIVE,
                        kv.CATALOG_PATH = src.CATALOG_PATH,
                        kv.LAST_UPDATED = SYSDATE
                WHEN NOT MATCHED THEN
                    INSERT (SR_ID, CATALOG_ITEM_ID, CATALOG_PATH, FULL_NARRATIVE, NARRATIVE_VECTOR)
                    VALUES (src.SR_ID, src.CATALOG_ITEM_ID, src.CATALOG_PATH, src.FULL_NARRATIVE, src.NARRATIVE_VECTOR);
                
                -- Mark as processed in staging table
                UPDATE SIEBEL_NARRATIVES_STAGING
                SET PROCESSED_FLAG = 'Y',
                    PROCESSED_DATE = SYSDATE
                WHERE SR_ID = rec.SR_ID;
                
                v_processed_count := v_processed_count + 1;
                
                -- Commit after each successful record to avoid losing progress
                COMMIT;
                
                -- Progress indicator (every 10 records)
                IF MOD(v_processed_count, 10) = 0 THEN
                    DBMS_OUTPUT.PUT_LINE('Processed: ' || v_processed_count || ' records');
                END IF;
                
            END;
            
        EXCEPTION
            WHEN OTHERS THEN
                -- Log error but continue processing
                v_error_count := v_error_count + 1;
                DBMS_OUTPUT.PUT_LINE('ERROR processing SR_ID ' || rec.SR_ID || ': ' || SQLERRM);
                
                -- Update staging with error flag
                UPDATE SIEBEL_NARRATIVES_STAGING
                SET PROCESSED_FLAG = 'E', -- E = Error
                    PROCESSED_DATE = SYSDATE
                WHERE SR_ID = rec.SR_ID;
                
                COMMIT;
        END;
        
        -- Rate limiting: Small delay to avoid overwhelming OCI API
        DBMS_SESSION.SLEEP(0.1); -- 100ms delay between calls
        
    END LOOP;
    
    -- Final summary
    DBMS_OUTPUT.PUT_LINE('-----------------------------------');
    DBMS_OUTPUT.PUT_LINE('Batch processing complete');
    DBMS_OUTPUT.PUT_LINE('Successfully processed: ' || v_processed_count);
    DBMS_OUTPUT.PUT_LINE('Errors encountered: ' || v_error_count);
    DBMS_OUTPUT.PUT_LINE('Remaining pending: ' || (v_total_pending - v_processed_count - v_error_count));
    
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('FATAL ERROR in batch processing: ' || SQLERRM);
        ROLLBACK;
        RAISE;
END GENERATE_EMBEDDINGS_BATCH;
/
```

#### 3.3. Test the Procedure with Small Batch
```sql
-- Enable output
SET SERVEROUTPUT ON SIZE UNLIMITED;

-- Test with 5 records first
BEGIN
    GENERATE_EMBEDDINGS_BATCH(
        p_batch_size => 5,
        p_compartment_id => 'ocid1.compartment.oc1..aaaaaaaa...your_compartment_ocid',
        p_region => 'us-ashburn-1' -- Change to your region
    );
END;
/

-- Verify results
SELECT 
    COUNT(*) AS TOTAL_VECTORS,
    COUNT(CASE WHEN NARRATIVE_VECTOR IS NOT NULL THEN 1 END) AS VECTORS_GENERATED
FROM SIEBEL_KNOWLEDGE_VECTORS;

-- Check a sample vector
SELECT 
    SR_ID,
    CATALOG_PATH,
    VECTOR_DIMENSION_COUNT(NARRATIVE_VECTOR) AS DIMENSIONS,
    SUBSTR(TO_CHAR(NARRATIVE_VECTOR), 1, 100) AS VECTOR_PREVIEW
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE ROWNUM <= 3;
```

---

### Step 4: Process All Narratives in Batches

**Executor:** Database Administrator on Oracle Autonomous Database  
**Duration:** Several hours (depending on data volume)

#### 4.1. Create Processing Loop Script
```sql
-- Script to process all pending narratives
DECLARE
    v_remaining NUMBER;
    v_iteration NUMBER := 0;
    v_batch_size NUMBER := 50; -- Process 50 at a time
BEGIN
    LOOP
        -- Check how many are remaining
        SELECT COUNT(*) INTO v_remaining
        FROM SIEBEL_NARRATIVES_STAGING
        WHERE PROCESSED_FLAG = 'N';
        
        EXIT WHEN v_remaining = 0;
        
        v_iteration := v_iteration + 1;
        DBMS_OUTPUT.PUT_LINE('=== Iteration ' || v_iteration || ' ===');
        DBMS_OUTPUT.PUT_LINE('Remaining: ' || v_remaining);
        
        -- Process batch
        GENERATE_EMBEDDINGS_BATCH(
            p_batch_size => v_batch_size,
            p_compartment_id => 'ocid1.compartment.oc1..aaaaaaaa...your_compartment_ocid',
            p_region => 'us-ashburn-1'
        );
        
        -- Small delay between batches
        DBMS_SESSION.SLEEP(2); -- 2 second pause
        
    END LOOP;
    
    DBMS_OUTPUT.PUT_LINE('=== ALL PROCESSING COMPLETE ===');
END;
/
```

#### 4.2. Monitor Progress
```sql
-- Query to monitor progress
SELECT 
    COUNT(*) AS TOTAL,
    COUNT(CASE WHEN PROCESSED_FLAG = 'Y' THEN 1 END) AS PROCESSED,
    COUNT(CASE WHEN PROCESSED_FLAG = 'N' THEN 1 END) AS PENDING,
    COUNT(CASE WHEN PROCESSED_FLAG = 'E' THEN 1 END) AS ERRORS,
    ROUND(COUNT(CASE WHEN PROCESSED_FLAG = 'Y' THEN 1 END) * 100.0 / COUNT(*), 2) AS PCT_COMPLETE
FROM SIEBEL_NARRATIVES_STAGING;

-- Check for errors
SELECT SR_ID, SR_NUM, PROCESSED_DATE
FROM SIEBEL_NARRATIVES_STAGING
WHERE PROCESSED_FLAG = 'E'
ORDER BY PROCESSED_DATE DESC;
```

#### 4.3. Schedule for Automated Processing
```sql
-- Create a scheduled job to process new records daily
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
        job_name        => 'DAILY_EMBEDDING_GENERATION',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'BEGIN GENERATE_EMBEDDINGS_BATCH(p_batch_size => 100); END;',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=DAILY; BYHOUR=3; BYMINUTE=0; BYSECOND=0',
        enabled         => TRUE,
        comments        => 'Daily batch processing of new embeddings'
    );
END;
/

-- Verify job
SELECT job_name, enabled, state, next_run_date
FROM USER_SCHEDULER_JOBS
WHERE job_name = 'DAILY_EMBEDDING_GENERATION';
```

---

### Step 5: Create Vector Index

**Executor:** Database Administrator on Oracle Autonomous Database  
**Duration:** 30-60 minutes (depending on data volume)

#### 5.1. Create HNSW Vector Index

**Important:** Only create the index AFTER all (or most) embeddings are generated.

```sql
-- Connect as SEMANTIC_SEARCH user

-- Create the HNSW vector index for fast similarity search
CREATE VECTOR INDEX SIBL_KNOW_VEC_HNSW_IDX 
ON SIEBEL_KNOWLEDGE_VECTORS (NARRATIVE_VECTOR)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE
WITH TARGET ACCURACY 95;

-- Monitor index creation progress
SELECT INDEX_NAME, STATUS, LAST_ANALYZED
FROM USER_INDEXES
WHERE INDEX_NAME = 'SIBL_KNOW_VEC_HNSW_IDX';
```

**Index Parameters Explained:**
- **ORGANIZATION INMEMORY NEIGHBOR GRAPH**: Uses HNSW algorithm, optimized for in-memory processing
- **DISTANCE COSINE**: Cosine similarity metric (best for text embeddings)
- **TARGET ACCURACY 95**: Balance between speed and accuracy (higher = slower but more accurate)

#### 5.2. Gather Statistics
```sql
-- Gather statistics on the vector table and index
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname => 'SEMANTIC_SEARCH',
        tabname => 'SIEBEL_KNOWLEDGE_VECTORS',
        estimate_percent => 100,
        cascade => TRUE -- Also gather index stats
    );
END;
/

-- Verify statistics
SELECT 
    table_name, 
    num_rows, 
    last_analyzed
FROM USER_TABLES
WHERE table_name = 'SIEBEL_KNOWLEDGE_VECTORS';
```

#### 5.3. Test Vector Search Performance
```sql
-- Test a sample vector search
DECLARE
    l_test_vector VECTOR(1024, FLOAT64);
BEGIN
    -- Get a sample vector from the table
    SELECT NARRATIVE_VECTOR INTO l_test_vector
    FROM SIEBEL_KNOWLEDGE_VECTORS
    WHERE ROWNUM = 1;
    
    -- Perform a similarity search
    FOR rec IN (
        SELECT 
            SR_ID,
            CATALOG_PATH,
            VECTOR_DISTANCE(NARRATIVE_VECTOR, l_test_vector, COSINE) AS SIMILARITY_SCORE
        FROM 
            SIEBEL_KNOWLEDGE_VECTORS
        ORDER BY 
            VECTOR_DISTANCE(NARRATIVE_VECTOR, l_test_vector, COSINE) ASC
        FETCH FIRST 10 ROWS ONLY
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('SR: ' || rec.SR_ID || ' | Score: ' || rec.SIMILARITY_SCORE);
    END LOOP;
END;
/
```

---

## 4. Verification and Validation

### 4.1. Data Quality Checks
```sql
-- Check vector completeness
SELECT 
    'Total Records' AS METRIC,
    COUNT(*) AS VALUE
FROM SIEBEL_KNOWLEDGE_VECTORS
UNION ALL
SELECT 
    'Records with Vectors',
    COUNT(*)
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE NARRATIVE_VECTOR IS NOT NULL
UNION ALL
SELECT 
    'Records without Vectors',
    COUNT(*)
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE NARRATIVE_VECTOR IS NULL;

-- Check vector dimensions
SELECT 
    SR_ID,
    VECTOR_DIMENSION_COUNT(NARRATIVE_VECTOR) AS DIMENSIONS
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE ROWNUM <= 10;
-- Expected: All should be 1024

-- Check for NULL or zero vectors
SELECT COUNT(*) AS NULL_VECTORS
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE NARRATIVE_VECTOR IS NULL;
-- Expected: 0
```

### 4.2. Performance Validation
```sql
-- Test query performance (should be < 100ms for indexed query)
SET TIMING ON;

SELECT 
    SR_ID,
    CATALOG_PATH,
    VECTOR_DISTANCE(
        NARRATIVE_VECTOR, 
        (SELECT NARRATIVE_VECTOR FROM SIEBEL_KNOWLEDGE_VECTORS WHERE ROWNUM = 1),
        COSINE
    ) AS SIMILARITY
FROM 
    SIEBEL_KNOWLEDGE_VECTORS
ORDER BY 
    SIMILARITY DESC
FETCH FIRST 20 ROWS ONLY;

SET TIMING OFF;
```

---

## 5. Troubleshooting Guide

### 5.1. Common Issues

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| ORA-29532 during API call | Invalid OCI credentials | Verify credentials with test in Step 1.3 |
| JSON parsing error | Unexpected API response format | Check API response structure, verify model version |
| Slow processing | Network latency, API throttling | Reduce batch size, add delays between calls |
| Index creation fails | Insufficient memory | Increase SGA/PGA, create index during off-peak hours |
| Vector dimensions mismatch | Wrong embedding model | Verify model in API call matches table definition |

### 5.2. Reprocessing Failed Records
```sql
-- Reset error records for reprocessing
UPDATE SIEBEL_NARRATIVES_STAGING
SET PROCESSED_FLAG = 'N',
    PROCESSED_DATE = NULL
WHERE PROCESSED_FLAG = 'E';

COMMIT;

-- Reprocess
BEGIN
    GENERATE_EMBEDDINGS_BATCH(p_batch_size => 20);
END;
/
```

### 5.3. Performance Tuning
```sql
-- Check SGA usage
SELECT 
    pool,
    name,
    ROUND(bytes/1024/1024, 2) AS MB
FROM v$sgastat
WHERE name LIKE '%vector%'
ORDER BY bytes DESC;

-- Check PGA usage
SELECT 
    name,
    ROUND(value/1024/1024, 2) AS MB
FROM v$pgastat
WHERE name LIKE '%PGA%';
```

---

## 6. Maintenance Procedures

### 6.1. Incremental Processing for New Records
```sql
-- This will be called by the scheduled job automatically
-- Or run manually for immediate processing:
BEGIN
    GENERATE_EMBEDDINGS_BATCH(
        p_batch_size => 100,
        p_compartment_id => 'ocid1.compartment.oc1..aaaaaaaa...',
        p_region => 'us-ashburn-1'
    );
END;
/
```

### 6.2. Rebuild Index (if needed)
```sql
-- Drop and recreate index if performance degrades
DROP INDEX SIBL_KNOW_VEC_HNSW_IDX;

CREATE VECTOR INDEX SIBL_KNOW_VEC_HNSW_IDX 
ON SIEBEL_KNOWLEDGE_VECTORS (NARRATIVE_VECTOR)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE
WITH TARGET ACCURACY 95;
```

### 6.3. Archive Old Vectors
```sql
-- Create archive table for old vectors
CREATE TABLE SIEBEL_KNOWLEDGE_VECTORS_ARCHIVE AS
SELECT * FROM SIEBEL_KNOWLEDGE_VECTORS WHERE 1=0;

-- Move vectors older than 2 years to archive
INSERT INTO SIEBEL_KNOWLEDGE_VECTORS_ARCHIVE
SELECT * FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE CREATED_DATE < ADD_MONTHS(SYSDATE, -24);

-- Delete from main table
DELETE FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE CREATED_DATE < ADD_MONTHS(SYSDATE, -24);

COMMIT;
```

---

## 7. Next Steps

Once this TDD is complete, proceed to:
- **TDD 3**: Create the ORDS-based Semantic Search API that queries these vectors
- The vector index created here will be used for high-performance similarity searches
