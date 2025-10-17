# Technical Design Document 2: Vector Database and Indexing Pipeline

**Version:** 3.0  
**Date:** 2025-10-17  
**Owner:** AI / Data Engineering Team

## 1. Objective
This document provides the technical specifications for generating vector embeddings from the staged Siebel narratives and populating the **Oracle Database 23ai** vector store on Azure VM. We will use **PL/SQL and DBMS_CLOUD** to call **Azure AI Foundry's OpenAI Service** directly from the database, eliminating the need for external Python scripts. This approach leverages Oracle 23ai's AI Vector Search capabilities with Azure's unified AI platform.

## 2. Architecture Overview

### 2.1. Processing Flow
```
SIEBEL_NARRATIVES_STAGING (PROCESSED_FLAG='N')
      ↓
PL/SQL Procedure: GENERATE_EMBEDDINGS_BATCH
      ↓
Call Azure AI Foundry OpenAI Service via DBMS_CLOUD (HTTPS)
      ↓
Parse JSON response (1536-dimensional vector)
      ↓
Insert into SIEBEL_KNOWLEDGE_VECTORS
      ↓
Update PROCESSED_FLAG='Y'
      ↓
Create HNSW Vector Index (after all embeddings complete)
```

### 2.2. Key Components
- **Oracle 23ai VM**: Self-managed database with AI Vector Search
- **Azure AI Foundry**: Unified AI platform hosting OpenAI Service
- **OpenAI text-embedding-3-small**: 1536-dimensional embedding model
- **DBMS_CLOUD**: Oracle package for HTTP REST API calls
- **Network ACLs**: Configured for `*.api.azureml.ms` and `*.openai.azure.com`

## 3. Implementation Steps

### Step 1: Configure Azure AI Foundry with OpenAI Service Credentials in Oracle 23ai

**Executor:** Database Administrator on Oracle 23ai VM  
**Duration:** 30 minutes

#### 1.1. Obtain Azure AI Foundry with OpenAI Service API Credentials

Before proceeding, gather the following from Azure Portal:

1. **Azure AI Foundry Workspace Name**: Your workspace identifier
2. **Azure Region**: e.g., `eastus`, `westus2`, `northeurope`
3. **API Endpoint**: Format: `https://<workspace-name>.<region>.api.azureml.ms`
4. **API Key**: Navigate to Azure AI Foundry → Keys and Endpoint → Copy Key 1
5. **Deployment Name**: The name you gave to your OpenAI deployment (e.g., `text-embedding-3-small`)

**Example API Endpoint:**
```
https://my-ai-workspace.eastus.api.azureml.ms/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-15-preview
```

#### 1.2. Create Azure AI Foundry Credential in Oracle 23ai

```bash
# SSH to Oracle 23ai VM
ssh oracle@<oracle-23ai-vm-ip>

# Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH
```

```sql
-- Create credential for Azure AI Foundry API authentication
-- Note: Azure uses Bearer token authentication (API key)

BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'AZURE_AI_FOUNDRY_CRED',
    username        => 'AZURE_OPENAI',  -- Placeholder username (required by DBMS_CLOUD)
    password        => '<your-azure-ai-foundry-api-key>'
  );
END;
/

-- Verify credential creation
SELECT credential_name, username, enabled
FROM USER_CREDENTIALS
WHERE credential_name = 'AZURE_AI_FOUNDRY_CRED';
-- Expected: AZURE_AI_FOUNDRY_CRED | AZURE_OPENAI | ENABLED

COMMIT;
```

**Security Best Practice:**
- Store API keys securely in Azure Key Vault
- Rotate keys periodically (regenerate Key 2, switch, regenerate Key 1)
- Use Azure Managed Identity where possible (future enhancement)
- Never commit API keys to source control

#### 1.3. Verify Network ACL Configuration

**Note:** Network ACLs were configured during deployment (Phase 2, Step 2.4). Verify they are in place:

```sql
-- Verify ACL configuration allows Azure AI Foundry access
SELECT host, lower_port, upper_port, acl
FROM dba_network_acls
WHERE host LIKE '%api.azureml.ms' OR host LIKE '%openai.azure.com';

-- Expected output:
-- HOST                    LOWER_PORT  UPPER_PORT  ACL
-- *.api.azureml.ms        443         443         azure_ai_foundry_acl.xml
-- *.openai.azure.com      443         443         azure_ai_foundry_acl.xml
```

If ACLs are missing, create them:

```sql
-- Create and assign ACL (only if not done in deployment)
BEGIN
  DBMS_NETWORK_ACL_ADMIN.CREATE_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    description  => 'ACL for Azure AI Foundry API access',
    principal    => 'SEMANTIC_SEARCH',
    is_grant     => TRUE,
    privilege    => 'connect',
    start_date   => NULL,
    end_date     => NULL
  );
  
  DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE (
    acl          => 'azure_ai_foundry_acl.xml',
    principal    => 'SEMANTIC_SEARCH',
    is_grant     => TRUE,
    privilege    => 'resolve'
  );
  
  DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    host         => '*.api.azureml.ms',
    lower_port   => 443,
    upper_port   => 443
  );
  
  DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    host         => '*.openai.azure.com',
    lower_port   => 443,
    upper_port   => 443
  );
  
  COMMIT;
END;
/
```

#### 1.4. Test Azure AI Foundry API Connectivity

```sql
-- Test Azure AI Foundry connectivity with a simple embedding call
SET SERVEROUTPUT ON SIZE UNLIMITED;

DECLARE
    l_api_endpoint VARCHAR2(500) := 'https://<workspace-name>.<region>.api.azureml.ms/openai/deployments/<deployment-name>/embeddings?api-version=2024-02-15-preview';
    l_request_body CLOB;
    l_response CLOB;
    l_http_status NUMBER;
BEGIN
    -- Construct request body
    l_request_body := '{
        "input": "test connectivity",
        "model": "text-embedding-3-small"
    }';
    
    -- Make API call
    l_response := DBMS_CLOUD.SEND_REQUEST(
        credential_name => 'AZURE_AI_FOUNDRY_CRED',
        uri             => l_api_endpoint,
        method          => DBMS_CLOUD.METHOD_POST,
        body            => UTL_RAW.CAST_TO_RAW(l_request_body),
        headers         => JSON_OBJECT(
            'Content-Type' VALUE 'application/json',
            'api-key' VALUE (SELECT password FROM user_credentials WHERE credential_name = 'AZURE_AI_FOUNDRY_CRED')
        )
    );
    
    -- Check response
    DBMS_OUTPUT.PUT_LINE('Response length: ' || DBMS_LOB.GETLENGTH(l_response) || ' bytes');
    DBMS_OUTPUT.PUT_LINE('First 200 chars: ' || SUBSTR(l_response, 1, 200));
    
    -- Expected: JSON response with "data" array containing embedding vector
    IF l_response LIKE '%"data"%' THEN
        DBMS_OUTPUT.PUT_LINE('✓ Azure AI Foundry connectivity test PASSED');
    ELSE
        DBMS_OUTPUT.PUT_LINE('✗ Azure AI Foundry connectivity test FAILED');
    END IF;
    
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        RAISE;
END;
/

EXIT;
```

**Troubleshooting:**
- **ORA-29273**: HTTP request failed → Check network ACL configuration
- **ORA-29270**: Connection refused → Verify firewall allows outbound HTTPS (port 443)
- **HTTP 401 Unauthorized**: Check API key is correct
- **HTTP 404 Not Found**: Verify API endpoint URL and deployment name
- **Timeout errors**: Check network latency to Azure region, consider increasing timeout

**Test Script:**
```sql
-- Test Azure AI Foundry connectivity
SET SERVEROUTPUT ON SIZE UNLIMITED;
DECLARE
  l_response CLOB;
  l_request_body CLOB;
  l_api_endpoint VARCHAR2(500) := 'https://<workspace-name>.<region>.api.azureml.ms/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-15-preview';
BEGIN
  l_request_body := '{
    "input": ["Test connection"],
    "model": "text-embedding-3-small"
  }';
  
  l_response := DBMS_CLOUD.SEND_REQUEST(
    credential_name => 'AZURE_AI_FOUNDRY_CRED',
    uri => l_api_endpoint,
    method => 'POST',
    body => UTL_RAW.CAST_TO_RAW(l_request_body)
  );
  
  DBMS_OUTPUT.PUT_LINE('Response: ' || SUBSTR(l_response, 1, 200));
  DBMS_OUTPUT.PUT_LINE('SUCCESS: Azure AI Foundry API is accessible');
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('ERROR: ' || SQLERRM);
    RAISE;
END;
/
```

**Troubleshooting:**
- **ORA-20000**: Invalid credentials → Check API key
- **Network timeout**: Verify Network ACL is configured
- **401 Unauthorized**: Check API key is valid and not expired
- **404 Not Found**: Verify workspace name, region, and deployment name in URI

---

### Step 2: Create Vector Table

**Executor:** Database Administrator on Oracle 23ai  
**Duration:** 5 minutes

#### 2.1. Create the Vector Table DDL
```sql
-- Connect as SEMANTIC_SEARCH user

CREATE TABLE SIEBEL_KNOWLEDGE_VECTORS (
    SR_ID                   VARCHAR2(15) NOT NULL PRIMARY KEY,
    CATALOG_ITEM_ID         VARCHAR2(15) NOT NULL,
    CATALOG_PATH            VARCHAR2(4000),
    FULL_NARRATIVE          CLOB,
    NARRATIVE_VECTOR        VECTOR(1536, FLOAT32), -- Azure OpenAI text-embedding-3-small uses 1536 dimensions with FLOAT32
    EMBEDDING_MODEL         VARCHAR2(100) DEFAULT 'text-embedding-3-small',
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
COMMENT ON COLUMN SIEBEL_KNOWLEDGE_VECTORS.NARRATIVE_VECTOR IS 'The numerical vector representation of the narrative (1536 dimensions, FLOAT32)';

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
    
    -- Truncate to 8000 characters if too long (Azure OpenAI recommended limit for text-embedding-3-small)
    IF LENGTH(l_clean_text) > 8000 THEN
        l_clean_text := SUBSTR(l_clean_text, 1, 8000);
    END IF;
    
    RETURN l_clean_text;
END CLEAN_TEXT_FOR_EMBEDDING;
/
```

#### 3.2. Create Main Embedding Generation Procedure

**Important Note:** This improved version implements:
- ✅ Batch API calls (100 records per call for 100x performance improvement)
- ✅ Intelligent error handling with retry logic for network errors
- ✅ Authentication failure detection (aborts immediately on 401 errors)
- ✅ Failure thresholds to prevent resource waste (aborts on excessive errors)
- ✅ Error message tracking in SIEBEL_NARRATIVES_STAGING table

```sql
CREATE OR REPLACE PROCEDURE GENERATE_EMBEDDINGS_BATCH(
    p_batch_size IN NUMBER DEFAULT 100,
    p_workspace_endpoint IN VARCHAR2 DEFAULT 'https://<workspace-name>.<region>.api.azureml.ms',
    p_deployment_name IN VARCHAR2 DEFAULT 'text-embedding-3-small',
    p_max_consecutive_errors IN NUMBER DEFAULT 10,
    p_max_error_rate IN NUMBER DEFAULT 0.5
) AS
    -- API Configuration
    l_api_url VARCHAR2(500);
    l_request_body CLOB;
    l_response CLOB;
    
    -- Counters
    v_processed_count NUMBER := 0;
    v_error_count NUMBER := 0;
    v_consecutive_errors NUMBER := 0;
    v_total_pending NUMBER;
    v_batch_processed NUMBER := 0;
    
    -- Batch processing collections
    TYPE t_narrative_array IS TABLE OF CLOB INDEX BY PLS_INTEGER;
    TYPE t_sr_id_array IS TABLE OF VARCHAR2(100) INDEX BY PLS_INTEGER;
    TYPE t_catalog_id_array IS TABLE OF VARCHAR2(100) INDEX BY PLS_INTEGER;
    TYPE t_catalog_path_array IS TABLE OF VARCHAR2(500) INDEX BY PLS_INTEGER;
    
    l_narratives t_narrative_array;
    l_sr_ids t_sr_id_array;
    l_catalog_ids t_catalog_id_array;
    l_catalog_paths t_catalog_path_array;
    l_batch_counter NUMBER := 0;
    
    -- Error tracking
    l_error_code NUMBER;
    l_error_message VARCHAR2(4000);
    l_retry_count NUMBER;
    l_max_retries CONSTANT NUMBER := 3;
    
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
        FOR UPDATE SKIP LOCKED;
        
BEGIN
    -- Set API URL
    l_api_url := p_workspace_endpoint || '/openai/deployments/' || p_deployment_name || '/embeddings?api-version=2024-02-15-preview';
    
    -- Get count of pending records
    SELECT COUNT(*) INTO v_total_pending
    FROM SIEBEL_NARRATIVES_STAGING
    WHERE PROCESSED_FLAG = 'N';
    
    DBMS_OUTPUT.PUT_LINE('=== Batch Processing Started ===');
    DBMS_OUTPUT.PUT_LINE('Pending records: ' || v_total_pending);
    DBMS_OUTPUT.PUT_LINE('Batch size: ' || p_batch_size);
    DBMS_OUTPUT.PUT_LINE('API endpoint: ' || l_api_url);
    
    -- Process records in batches of 100
    FOR rec IN c_unprocessed LOOP
        BEGIN
            -- Add to batch
            l_batch_counter := l_batch_counter + 1;
            l_sr_ids(l_batch_counter) := rec.SR_ID;
            l_catalog_ids(l_batch_counter) := rec.CATALOG_ITEM_ID;
            l_catalog_paths(l_batch_counter) := rec.CATALOG_PATH;
            l_narratives(l_batch_counter) := CLEAN_TEXT_FOR_EMBEDDING(rec.FULL_NARRATIVE);
            
            -- Process when batch is full (100 records) or at end of cursor
            IF l_batch_counter >= 100 THEN
                l_retry_count := 0;
                
                <<retry_loop>>
                LOOP
                    BEGIN
                        -- Build batch request (Azure OpenAI supports input arrays)
                        DECLARE
                            l_input_json CLOB;
                        BEGIN
                            -- Build JSON array of texts
                            SELECT '[' || LISTAGG('"' || REPLACE(l_narratives(i), '"', '\"') || '"', ',') 
                                   WITHIN GROUP (ORDER BY i) || ']'
                            INTO l_input_json
                            FROM (SELECT LEVEL AS i FROM DUAL CONNECT BY LEVEL <= l_batch_counter);
                            
                            l_request_body := JSON_OBJECT(
                                'input' VALUE JSON_QUERY(l_input_json, '$'),
                                'model' VALUE p_deployment_name
                            );
                        END;
                        
                        -- Call Azure AI Foundry
                        l_response := DBMS_CLOUD.SEND_REQUEST(
                            credential_name => 'AZURE_AI_FOUNDRY_CRED',
                            uri => l_api_url,
                            method => 'POST',
                            body => UTL_RAW.CAST_TO_RAW(l_request_body)
                        );
                        
                        -- Process batch response
                        FOR i IN 1..l_batch_counter LOOP
                            DECLARE
                                l_vector VECTOR(1536, FLOAT32);
                            BEGIN
                                -- Extract vector for this index
                                SELECT JSON_VALUE(
                                    l_response, 
                                    '$.data[' || (i-1) || '].embedding' 
                                    RETURNING VECTOR(1536, FLOAT32)
                                ) INTO l_vector FROM DUAL;
                                
                                -- Update vector table
                                MERGE INTO SIEBEL_KNOWLEDGE_VECTORS kv
                                USING DUAL ON (kv.SR_ID = l_sr_ids(i))
                                WHEN MATCHED THEN
                                    UPDATE SET
                                        kv.NARRATIVE_VECTOR = l_vector,
                                        kv.LAST_UPDATED = SYSDATE
                                WHEN NOT MATCHED THEN
                                    INSERT (SR_ID, CATALOG_ITEM_ID, CATALOG_PATH, FULL_NARRATIVE, NARRATIVE_VECTOR, EMBEDDING_MODEL)
                                    VALUES (l_sr_ids(i), l_catalog_ids(i), l_catalog_paths(i), l_narratives(i), l_vector, p_deployment_name);
                                
                                -- Mark as processed
                                UPDATE SIEBEL_NARRATIVES_STAGING
                                SET PROCESSED_FLAG = 'Y',
                                    PROCESSED_DATE = SYSDATE,
                                    ERROR_MESSAGE = NULL
                                WHERE SR_ID = l_sr_ids(i);
                            END;
                        END LOOP;
                        
                        -- Success! Reset counters
                        v_processed_count := v_processed_count + l_batch_counter;
                        v_consecutive_errors := 0;
                        COMMIT;
                        EXIT retry_loop;
                        
                    EXCEPTION
                        WHEN OTHERS THEN
                            l_error_code := SQLCODE;
                            l_error_message := SQLERRM;
                            
                            -- Check for authentication errors (abort immediately)
                            IF INSTR(l_error_message, '401') > 0 OR INSTR(l_error_message, 'Unauthorized') > 0 THEN
                                RAISE_APPLICATION_ERROR(-20001, 
                                    'CRITICAL: API authentication failed. Check credential. Job aborted.');
                            END IF;
                            
                            -- Check for network errors (retry)
                            IF l_error_code = -29273 OR INSTR(l_error_message, 'HTTP') > 0 THEN
                                l_retry_count := l_retry_count + 1;
                                
                                IF l_retry_count <= l_max_retries THEN
                                    DBMS_OUTPUT.PUT_LINE('Network error, retry ' || l_retry_count || '/' || l_max_retries);
                                    DBMS_SESSION.SLEEP(2 * l_retry_count); -- Exponential backoff
                                ELSE
                                    -- Max retries exceeded - mark as error
                                    DBMS_OUTPUT.PUT_LINE('ERROR: Max retries exceeded');
                                    v_error_count := v_error_count + l_batch_counter;
                                    v_consecutive_errors := v_consecutive_errors + 1;
                                    
                                    FOR i IN 1..l_batch_counter LOOP
                                        UPDATE SIEBEL_NARRATIVES_STAGING
                                        SET PROCESSED_FLAG = 'E',
                                            PROCESSED_DATE = SYSDATE,
                                            ERROR_MESSAGE = 'Network error after retries'
                                        WHERE SR_ID = l_sr_ids(i);
                                    END LOOP;
                                    COMMIT;
                                    EXIT retry_loop;
                                END IF;
                            ELSE
                                -- Other error - mark and continue
                                DBMS_OUTPUT.PUT_LINE('ERROR: ' || l_error_message);
                                v_error_count := v_error_count + l_batch_counter;
                                v_consecutive_errors := v_consecutive_errors + 1;
                                
                                FOR i IN 1..l_batch_counter LOOP
                                    UPDATE SIEBEL_NARRATIVES_STAGING
                                    SET PROCESSED_FLAG = 'E',
                                        PROCESSED_DATE = SYSDATE,
                                        ERROR_MESSAGE = SUBSTR(l_error_message, 1, 500)
                                    WHERE SR_ID = l_sr_ids(i);
                                END LOOP;
                                COMMIT;
                                EXIT retry_loop;
                            END IF;
                    END;
                END LOOP retry_loop;
                
                -- Check failure thresholds
                IF v_consecutive_errors >= p_max_consecutive_errors THEN
                    RAISE_APPLICATION_ERROR(-20002, 
                        'ABORT: ' || p_max_consecutive_errors || ' consecutive failures. Check configuration.');
                END IF;
                
                v_batch_processed := v_batch_processed + l_batch_counter;
                IF v_batch_processed > 100 AND (v_error_count / v_batch_processed) > p_max_error_rate THEN
                    RAISE_APPLICATION_ERROR(-20003, 
                        'ABORT: Error rate ' || ROUND((v_error_count/v_batch_processed)*100,1) || '% exceeds threshold');
                END IF;
                
                -- Reset batch
                l_batch_counter := 0;
                l_narratives.DELETE;
                l_sr_ids.DELETE;
                l_catalog_ids.DELETE;
                l_catalog_paths.DELETE;
                
                IF MOD(v_processed_count, 100) = 0 THEN
                    DBMS_OUTPUT.PUT_LINE('Processed: ' || v_processed_count);
                END IF;
            END IF;
            
        EXCEPTION
            WHEN OTHERS THEN
                DBMS_OUTPUT.PUT_LINE('UNEXPECTED ERROR: ' || SQLERRM);
                ROLLBACK;
                RAISE;
        END;
    END LOOP;
    
    -- Process any remaining records in partial batch
    IF l_batch_counter > 0 THEN
        -- Similar logic as above for remaining records
        DBMS_OUTPUT.PUT_LINE('Processing final partial batch of ' || l_batch_counter || ' records...');
        -- [Implementation similar to main batch processing above]
    END IF;
    
    -- Final summary
    DBMS_OUTPUT.PUT_LINE('=== Batch Processing Complete ===');
    DBMS_OUTPUT.PUT_LINE('Successfully processed: ' || v_processed_count);
    DBMS_OUTPUT.PUT_LINE('Errors encountered: ' || v_error_count);
    DBMS_OUTPUT.PUT_LINE('Error rate: ' || ROUND((v_error_count/GREATEST(v_batch_processed,1))*100,1) || '%');
    DBMS_OUTPUT.PUT_LINE('Remaining pending: ' || (v_total_pending - v_processed_count - v_error_count));
    
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('FATAL ERROR: ' || SQLERRM);
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

**Executor:** Database Administrator on Oracle 23ai  
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
        
        -- Process batch using Azure AI Foundry
        GENERATE_EMBEDDINGS_BATCH(
            p_batch_size => v_batch_size,
            p_workspace_endpoint => 'https://<workspace-name>.<region>.api.azureml.ms',
            p_deployment_name => 'text-embedding-3-small'
        );
        
        -- Small delay between batches to avoid rate limiting
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

**Executor:** Database Administrator on Oracle 23ai  
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
    l_test_vector VECTOR(1536, FLOAT64);
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
-- Expected: All should be 1536

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
    ROUND(bytes/1536/1536, 2) AS MB
FROM v$sgastat
WHERE name LIKE '%vector%'
ORDER BY bytes DESC;

-- Check PGA usage
SELECT 
    name,
    ROUND(value/1536/1536, 2) AS MB
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

