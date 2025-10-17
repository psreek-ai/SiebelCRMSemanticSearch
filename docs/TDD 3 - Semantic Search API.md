# Technical Design Document 3: Semantic Search API

**Version:** 3.0  
**Date:** 2025-10-17  
**Owner:** Database Development Team

## 1. Objective
This document specifies the design and implementation of the RESTful API that provides semantic search functionality. The API is built and hosted directly within **Oracle Database 23ai on Azure VM** using **manually installed Oracle REST Data Services (ORDS)**, serving as a high-performance, secure endpoint that encapsulates all AI and vector database logic. This approach provides full control over ORDS configuration and deployment.

## 2. Technology Stack
* **Hosting Platform:** Oracle Database 23ai on Azure VM (Standard_D8s_v3)
* **API Engine:** Oracle REST Data Services (ORDS) 23.3.0 - Manually installed in standalone Jetty mode
* **Language:** PL/SQL, SQL with JSON extensions
* **Security:** API Keys via ORDS custom authentication
* **Network:** HTTP on port 8080 (internal VNet), HTTPS via Azure Application Gateway (production)

## 3. Prerequisites

Before implementing this TDD, ensure:
- ✅ TDD 1 is complete (data extraction from Oracle 19c Siebel database)
- ✅ TDD 2 is complete (vector generation via Azure AI Foundry and indexing)
- ✅ Oracle Database 23ai on Azure VM is installed and running
- ✅ **ORDS is installed and configured** (completed in Deployment Guide Phase 3)
- ✅ SEMANTIC_SEARCH schema is enabled for REST services

---

## 4. Implementation Steps

### Step 1: Verify ORDS Installation and Accessibility

**Executor:** Database Administrator  
**Duration:** 10 minutes

**Important:** Oracle Database 23ai on Azure VM requires manual ORDS installation (completed in Deployment Guide Phase 3). Verify it's properly configured before proceeding.

#### 1.1. Verify ORDS Service Status

```bash
# SSH to Oracle 23ai VM
ssh oracle@<oracle-23ai-vm-ip>

# Check ORDS service status
sudo systemctl status ords.service
# Expected: Active: active (running)

# Verify ORDS is listening on port 8080
sudo netstat -tuln | grep 8080
# Expected: tcp 0.0.0.0:8080 LISTEN

# Check ORDS logs for errors
sudo journalctl -u ords.service -n 50
# Expected: "ORDS is ready to accept requests"
```

#### 1.2. Test ORDS Accessibility

```bash
# Test ORDS from local VM
curl http://localhost:8080/ords/
# Expected: HTML response showing ORDS information page

# Test from another VM in same VNet (if available)
curl http://<oracle-23ai-vm-private-ip>:8080/ords/
# Expected: Same HTML response

# Test ORDS version endpoint
curl http://localhost:8080/ords/
# Look for ORDS version in response
```

#### 1.3. Verify ORDS Configuration in Database

```bash
# Connect to Oracle 23ai as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH
```

```sql
-- Check if schema is REST-enabled
SELECT name, enabled, url_mapping_type, url_mapping_pattern
FROM user_ords_schemas
WHERE name = 'SEMANTIC_SEARCH';

-- Expected output:
-- NAME               ENABLED  URL_MAPPING_TYPE  URL_MAPPING_PATTERN
-- SEMANTIC_SEARCH    TRUE     BASE_PATH         semantic_search

-- If schema is not enabled, enable it:
BEGIN
    ORDS.ENABLE_SCHEMA(
        p_enabled             => TRUE,
        p_schema              => 'SEMANTIC_SEARCH',
        p_url_mapping_type    => 'BASE_PATH',
        p_url_mapping_pattern => 'semantic_search',
        p_auto_rest_auth      => FALSE
    );
    COMMIT;
END;
/

-- Verify ORDS metadata objects exist
SELECT object_name, object_type 
FROM all_objects 
WHERE owner = 'ORDS_METADATA'
  AND object_type IN ('TABLE', 'VIEW')
  AND rownum <= 10
ORDER BY object_name;

EXIT;
```

**ORDS Installation Verification Checklist:**
- ✅ ORDS service is running (`systemctl status ords.service`)
- ✅ Port 8080 is listening (`netstat -tuln | grep 8080`)
- ✅ ORDS responds to HTTP requests (`curl http://localhost:8080/ords/`)
- ✅ SEMANTIC_SEARCH schema is REST-enabled
- ✅ ORDS_METADATA and ORDS_PUBLIC_USER schemas exist
- ✅ No errors in ORDS logs (`/opt/oracle/ords/logs/ords.log`)

**If ORDS is Not Installed:**
- Refer to Deployment Guide, Phase 3: Install and Configure ORDS
- Complete ORDS installation before proceeding with this TDD

---

### Step 2: Enable ORDS for SEMANTIC_SEARCH Schema

**Executor:** Database Administrator  
**Duration:** 5 minutes

#### 2.1. Enable REST Services for Schema

```sql
-- Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@<service_name>_high

BEGIN
    ORDS.ENABLE_SCHEMA(
        p_enabled             => TRUE,
        p_schema              => 'SEMANTIC_SEARCH',
        p_url_mapping_type    => 'BASE_PATH',
        p_url_mapping_pattern => 'semantic_search',
        p_auto_rest_auth      => FALSE
    );
    COMMIT;
END;
/

-- Verify schema is enabled
SELECT name, enabled, url_mapping_type, url_mapping_pattern
FROM user_ords_schemas;

-- Expected output:
-- NAME               ENABLED  URL_MAPPING_TYPE  URL_MAPPING_PATTERN
-- SEMANTIC_SEARCH    TRUE     BASE_PATH         semantic_search
```

#### 2.2. Test Schema Accessibility

```bash
# Test the schema endpoint using the Oracle 23ai ORDS URL
curl http://localhost:8080/ords/semantic_search/

# Expected: 404 or empty JSON response (no modules defined yet - this is normal)
```

---

### Step 3: Create PL/SQL Search Procedure

**Executor:** PL/SQL Developer  
**Duration:** 30 minutes

#### 3.1. Create Main Search Procedure

```sql
-- Connect as SEMANTIC_SEARCH user

CREATE OR REPLACE PROCEDURE GET_SEMANTIC_RECOMMENDATIONS (
    p_query_text    IN  CLOB,
    p_top_k         IN  NUMBER DEFAULT 5,
    p_result_json   OUT CLOB
) AS
    -- Variables for query vector generation
    l_query_vector      VECTOR(1536, FLOAT32);
    l_api_response      CLOB;
    l_api_req_body      CLOB;
    l_embedding_url     VARCHAR2(512);
    l_search_id         VARCHAR2(64);
    
    -- Configuration variables for Azure AI Foundry (customize for your environment)
    l_workspace_endpoint VARCHAR2(500) := 'https://<workspace-name>.<region>.api.azureml.ms';
    l_deployment_name    VARCHAR2(100) := 'text-embedding-3-small';
    l_api_version        VARCHAR2(50) := '2024-02-15-preview';
    
    -- Exception handling
    l_error_message     VARCHAR2(4000);
    
BEGIN
    -- Generate unique search ID for tracking
    l_search_id := SYS_GUID();
    
    -- Validate input
    IF p_query_text IS NULL OR LENGTH(TRIM(p_query_text)) = 0 THEN
        p_result_json := JSON_OBJECT(
            'error' VALUE 'Query text cannot be empty',
            'search_id' VALUE l_search_id
        );
        RETURN;
    END IF;
    
    -- Build Azure AI Foundry with OpenAI Service embedding URL
    l_embedding_url := l_workspace_endpoint || '/openai/deployments/' || l_deployment_name || '/embeddings?api-version=' || l_api_version;
    
    -- Step 1: Generate vector for the user query using Azure AI Foundry with OpenAI Service
    BEGIN
        -- Clean the query text
        DECLARE
            l_clean_query CLOB;
        BEGIN
            l_clean_query := CLEAN_TEXT_FOR_EMBEDDING(p_query_text);
            
            -- Build API request body for Azure AI Foundry
            SELECT JSON_OBJECT(
                'input' VALUE l_clean_query,
                'model' VALUE l_deployment_name
            ) INTO l_api_req_body FROM DUAL;
            
            -- Call Azure AI Foundry OpenAI service
            l_api_response := DBMS_CLOUD.SEND_REQUEST(
                credential_name => 'AZURE_AI_FOUNDRY_CRED',
                uri             => l_embedding_url,
                method          => 'POST',
                body            => UTL_RAW.CAST_TO_RAW(l_api_req_body)
            );
            
            -- Extract the vector from the Azure OpenAI JSON response
            SELECT JSON_VALUE(l_api_response, '$.data[0].embedding' RETURNING VECTOR(1536, FLOAT32))
            INTO l_query_vector
            FROM DUAL;
        END;
    EXCEPTION
        WHEN OTHERS THEN
            l_error_message := 'Error generating query embedding: ' || SQLERRM;
            p_result_json := JSON_OBJECT(
                'error' VALUE l_error_message,
                'search_id' VALUE l_search_id
            );
            RETURN;
    END;
    
    -- Step 2: Perform vector similarity search and aggregate results
    BEGIN
        WITH similarity_scores AS (
            -- Find most similar vectors
            SELECT
                v.SR_ID,
                v.CATALOG_ITEM_ID,
                v.CATALOG_PATH,
                VECTOR_DISTANCE(v.NARRATIVE_VECTOR, l_query_vector, COSINE) AS DISTANCE_SCORE,
                (1 - VECTOR_DISTANCE(v.NARRATIVE_VECTOR, l_query_vector, COSINE)) AS SIMILARITY_SCORE
            FROM
                SIEBEL_KNOWLEDGE_VECTORS v
            WHERE
                v.NARRATIVE_VECTOR IS NOT NULL
            ORDER BY
                VECTOR_DISTANCE(v.NARRATIVE_VECTOR, l_query_vector, COSINE) ASC
            FETCH FIRST 100 ROWS ONLY  -- Get top 100 similar records
        ),
        catalog_aggregation AS (
            -- Aggregate by catalog item and rank
            SELECT
                CATALOG_ITEM_ID,
                CATALOG_PATH,
                COUNT(*) AS FREQUENCY,
                ROUND(AVG(SIMILARITY_SCORE), 4) AS AVG_RELEVANCE_SCORE,
                ROUND(MAX(SIMILARITY_SCORE), 4) AS MAX_RELEVANCE_SCORE
            FROM
                similarity_scores
            GROUP BY
                CATALOG_ITEM_ID,
                CATALOG_PATH
            ORDER BY
                FREQUENCY DESC,
                AVG_RELEVANCE_SCORE DESC
            FETCH FIRST p_top_k ROWS ONLY
        )
        -- Build final JSON response
        SELECT JSON_OBJECT(
            'search_id' VALUE l_search_id,
            'query' VALUE p_query_text,
            'timestamp' VALUE TO_CHAR(SYSTIMESTAMP, 'YYYY-MM-DD"T"HH24:MI:SS.FF3"Z"'),
            'recommendations' VALUE (
                SELECT JSON_ARRAYAGG(
                    JSON_OBJECT(
                        'rank' VALUE ROWNUM,
                        'catalog_item_id' VALUE CATALOG_ITEM_ID,
                        'catalog_path' VALUE CATALOG_PATH,
                        'relevance_score' VALUE AVG_RELEVANCE_SCORE,
                        'frequency' VALUE FREQUENCY,
                        'max_score' VALUE MAX_RELEVANCE_SCORE
                    )
                    ORDER BY ROWNUM
                )
                FROM catalog_aggregation
            )
        ) INTO p_result_json
        FROM DUAL;
        
    EXCEPTION
        WHEN OTHERS THEN
            l_error_message := 'Error performing vector search: ' || SQLERRM;
            p_result_json := JSON_OBJECT(
                'error' VALUE l_error_message,
                'search_id' VALUE l_search_id
            );
            RETURN;
    END;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Catch-all exception handler
        p_result_json := JSON_OBJECT(
            'error' VALUE 'Unexpected error: ' || SQLERRM,
            'search_id' VALUE l_search_id
        );
END GET_SEMANTIC_RECOMMENDATIONS;
/
```

#### 3.2. Create Wrapper Procedure for ORDS

ORDS requires a procedure that uses HTTP methods. Create a wrapper:

```sql
CREATE OR REPLACE PROCEDURE HANDLE_SEARCH_REQUEST AS
    l_query_text    CLOB;
    l_top_k         NUMBER;
    l_result_json   CLOB;
BEGIN
    -- Get query text from request body
    l_query_text := UTL_RAW.CAST_TO_VARCHAR2(OWA_UTIL.GET_CGI_ENV('wsgi.input'));
    
    -- Get Top-K from header (default to 5 if not provided)
    BEGIN
        l_top_k := TO_NUMBER(OWA_UTIL.GET_CGI_ENV('HTTP_TOP_K'));
    EXCEPTION
        WHEN OTHERS THEN
            l_top_k := 5;
    END;
    
    -- Call main search procedure
    GET_SEMANTIC_RECOMMENDATIONS(
        p_query_text => l_query_text,
        p_top_k => l_top_k,
        p_result_json => l_result_json
    );
    
    -- Set response headers
    OWA_UTIL.MIME_HEADER('application/json', FALSE);
    OWA_UTIL.HTTP_HEADER_CLOSE;
    
    -- Return JSON response
    HTP.PRN(l_result_json);
    
EXCEPTION
    WHEN OTHERS THEN
        OWA_UTIL.STATUS_LINE(500, 'Internal Server Error');
        OWA_UTIL.MIME_HEADER('application/json', FALSE);
        OWA_UTIL.HTTP_HEADER_CLOSE;
        HTP.PRN(JSON_OBJECT('error' VALUE SQLERRM));
END HANDLE_SEARCH_REQUEST;
/
```

#### 3.3. Test the Procedure Directly

```sql
-- Test the search procedure directly
DECLARE
    l_result CLOB;
BEGIN
    GET_SEMANTIC_RECOMMENDATIONS(
        p_query_text => 'My laptop screen is broken',
        p_top_k => 5,
        p_result_json => l_result
    );
    
    DBMS_OUTPUT.PUT_LINE(l_result);
END;
/
```

---

### Step 4: Register API Endpoint with ORDS

**Executor:** Database Administrator  
**Duration:** 15 minutes

#### 4.1. Create ORDS Module

```sql
-- Connect as SEMANTIC_SEARCH user

BEGIN
    -- Define the REST module
    ORDS.DEFINE_MODULE(
        p_module_name    => 'siebel_search',
        p_base_path      => '/siebel/',
        p_items_per_page => 0,
        p_status         => 'PUBLISHED',
        p_comments       => 'Semantic search API for Siebel CRM'
    );
    
    COMMIT;
END;
/

-- Verify module creation
SELECT id, name, base_path, status
FROM user_ords_modules;
```

#### 4.2. Create ORDS Template

```sql
BEGIN
    -- Define the URL template
    ORDS.DEFINE_TEMPLATE(
        p_module_name => 'siebel_search',
        p_pattern     => 'search',
        p_priority    => 0,
        p_etag_type   => 'HASH',
        p_comments    => 'Endpoint for semantic search queries'
    );
    
    COMMIT;
END;
/

-- Verify template creation
SELECT id, uri_template, priority
FROM user_ords_templates
WHERE module_id = (SELECT id FROM user_ords_modules WHERE name = 'siebel_search');
```

#### 4.3. Create ORDS Handler

```sql
BEGIN
    -- Define the POST handler
    ORDS.DEFINE_HANDLER(
        p_module_name    => 'siebel_search',
        p_pattern        => 'search',
        p_method         => 'POST',
        p_source_type    => ORDS.source_type_plsql,
        p_source         => 'BEGIN HANDLE_SEARCH_REQUEST; END;',
        p_items_per_page => 0,
        p_comments       => 'POST handler for search requests'
    );
    
    COMMIT;
END;
/

-- Verify handler creation
SELECT id, method, source_type
FROM user_ords_handlers
WHERE template_id = (
    SELECT id FROM user_ords_templates 
    WHERE module_id = (SELECT id FROM user_ords_modules WHERE name = 'siebel_search')
);
```

---

### Step 5: Test the API Endpoint

**Executor:** QA / Developer  
**Duration:** 15 minutes

#### 5.1. Test with curl

```bash
# Test the API endpoint using Oracle 23ai ORDS URL
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "Top-K: 5" \
  -d "My computer is running very slow" \
  http://localhost:8080/ords/semantic_search/siebel/search

# Expected response:
# {
#   "search_id": "ABC123...",
#   "query": "My computer is running very slow",
#   "timestamp": "2025-10-17T10:30:45.123Z",
#   "recommendations": [
#     {
#       "rank": 1,
#       "catalog_item_id": "1-XXXXX",
#       "catalog_path": " > IT Support > Hardware > Performance Troubleshooting",
#       "relevance_score": 0.8745,
#       "frequency": 23,
#       "max_score": 0.9123
#     },
#     ...
#   ]
# }
```

#### 5.2. Test with Postman

1. Open Postman
2. Create new POST request
3. URL: `http://localhost:8080/ords/semantic_search/siebel/search`
4. Headers:
   - `Content-Type`: `text/plain`
   - `Top-K`: `5`
5. Body (raw): `My laptop screen is flickering`
6. Send request
7. Verify JSON response

#### 5.3. Test Error Handling

```bash
# Test with empty query
curl -X POST \
  -H "Content-Type: text/plain" \
  http://localhost:8080/ords/semantic_search/siebel/search

# Expected: Error response with appropriate message

# Test with invalid Top-K
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "Top-K: abc" \
  -d "Test query" \
  http://localhost:8080/ords/semantic_search/siebel/search

# Expected: Should default to 5 and return results
```

---

### Step 6: Implement Security

**Executor:** Security Administrator  
**Duration:** 30 minutes

#### 6.1. Create ORDS Role and Privilege

```sql
-- Connect as SEMANTIC_SEARCH user

BEGIN
    -- Create a role for API access
    ORDS.CREATE_ROLE(
        p_role_name => 'siebel_search_role'
    );
    
    COMMIT;
END;
/

-- Create a privilege for the search endpoint
BEGIN
    ORDS.CREATE_PRIVILEGE(
        p_name        => 'siebel_search_privilege',
        p_role_name   => 'siebel_search_role',
        p_label       => 'Siebel Search API Access',
        p_description => 'Privilege to access semantic search API',
        p_comments    => 'Required for Siebel CRM integration'
    );
    
    COMMIT;
END;
/

-- Associate privilege with the module
BEGIN
    ORDS.CREATE_PRIVILEGE_MAPPING(
        p_privilege_name => 'siebel_search_privilege',
        p_pattern        => '/siebel/*'
    );
    
    COMMIT;
END;
/
```

#### 6.2. Create OAuth2 Client (Option 1 - Recommended)

```sql
BEGIN
    OAUTH.CREATE_CLIENT(
        p_name            => 'siebel_crm_client',
        p_grant_type      => 'CLIENT_CREDENTIALS',
        p_owner           => 'Siebel CRM Application',
        p_description     => 'OAuth2 client for Siebel CRM integration',
        p_support_email   => 'siebel-admin@example.com',
        p_privilege_names => 'siebel_search_privilege'
    );
    
    COMMIT;
END;
/

-- Retrieve client ID and secret
SELECT 
    name,
    client_id,
    client_secret,
    grant_type
FROM user_ords_clients
WHERE name = 'siebel_crm_client';

-- IMPORTANT: Save the client_id and client_secret securely!
```

#### 6.3. Or Use API Key (Option 2 - Simpler)

If you prefer a simpler approach with API keys:

```bash
# Generate a secure API key
openssl rand -base64 32

# Store this key in Siebel Named Subsystem configuration
# Validate the key in your PL/SQL wrapper procedure
```

Update the wrapper procedure to check for API key:

```sql
CREATE OR REPLACE PROCEDURE HANDLE_SEARCH_REQUEST AS
    l_query_text    CLOB;
    l_top_k         NUMBER;
    l_result_json   CLOB;
    l_api_key       VARCHAR2(100);
    l_expected_key  VARCHAR2(100) := 'YOUR_SECURE_API_KEY_HERE'; -- Store securely!
BEGIN
    -- Get API key from header
    l_api_key := OWA_UTIL.GET_CGI_ENV('HTTP_X_API_KEY');
    
    -- Validate API key
    IF l_api_key IS NULL OR l_api_key != l_expected_key THEN
        OWA_UTIL.STATUS_LINE(401, 'Unauthorized');
        OWA_UTIL.MIME_HEADER('application/json', FALSE);
        OWA_UTIL.HTTP_HEADER_CLOSE;
        HTP.PRN(JSON_OBJECT('error' VALUE 'Invalid or missing API key'));
        RETURN;
    END IF;
    
    -- Continue with normal processing...
    l_query_text := UTL_RAW.CAST_TO_VARCHAR2(OWA_UTIL.GET_CGI_ENV('wsgi.input'));
    
    BEGIN
        l_top_k := TO_NUMBER(OWA_UTIL.GET_CGI_ENV('HTTP_TOP_K'));
    EXCEPTION
        WHEN OTHERS THEN
            l_top_k := 5;
    END;
    
    GET_SEMANTIC_RECOMMENDATIONS(
        p_query_text => l_query_text,
        p_top_k => l_top_k,
        p_result_json => l_result_json
    );
    
    OWA_UTIL.MIME_HEADER('application/json', FALSE);
    OWA_UTIL.HTTP_HEADER_CLOSE;
    HTP.PRN(l_result_json);
    
EXCEPTION
    WHEN OTHERS THEN
        OWA_UTIL.STATUS_LINE(500, 'Internal Server Error');
        OWA_UTIL.MIME_HEADER('application/json', FALSE);
        OWA_UTIL.HTTP_HEADER_CLOSE;
        HTP.PRN(JSON_OBJECT('error' VALUE SQLERRM));
END HANDLE_SEARCH_REQUEST;
/
```

#### 6.4. Test Secured Endpoint

```bash
# Test with API key
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "X-API-Key: YOUR_SECURE_API_KEY_HERE" \
  -H "Top-K: 5" \
  -d "Test query" \
  http://localhost:8080/ords/semantic_search/siebel/search

# Test without API key (should fail)
curl -X POST \
  -H "Content-Type: text/plain" \
  -d "Test query" \
  http://localhost:8080/ords/semantic_search/siebel/search

# Expected: 401 Unauthorized
```

---

### Step 7: Enable Logging and Monitoring

**Executor:** Database Administrator  
**Duration:** 20 minutes

#### 7.1. Create Logging Table

```sql
-- Create table to log API requests
CREATE TABLE API_SEARCH_LOG (
    log_id              NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    search_id           VARCHAR2(64),
    query_text          CLOB,
    top_k               NUMBER,
    response_time_ms    NUMBER,
    result_count        NUMBER,
    error_message       VARCHAR2(4000),
    request_timestamp   TIMESTAMP DEFAULT SYSTIMESTAMP,
    client_ip           VARCHAR2(50)
);

-- Create index for performance
CREATE INDEX IDX_SEARCH_LOG_TS ON API_SEARCH_LOG(request_timestamp);
```

#### 7.2. Update Procedure to Include Logging

```sql
CREATE OR REPLACE PROCEDURE GET_SEMANTIC_RECOMMENDATIONS (
    p_query_text    IN  CLOB,
    p_top_k         IN  NUMBER DEFAULT 5,
    p_result_json   OUT CLOB
) AS
    -- All previous declarations...
    l_start_time        TIMESTAMP;
    l_end_time          TIMESTAMP;
    l_response_time_ms  NUMBER;
    l_result_count      NUMBER := 0;
    
BEGIN
    l_start_time := SYSTIMESTAMP;
    l_search_id := SYS_GUID();
    
    -- All previous logic...
    -- (Keep existing code)
    
    -- After successful execution, log the request
    l_end_time := SYSTIMESTAMP;
    l_response_time_ms := EXTRACT(SECOND FROM (l_end_time - l_start_time)) * 1000;
    
    -- Count results from JSON
    BEGIN
        SELECT JSON_VALUE(p_result_json, '$.recommendations.size()')
        INTO l_result_count
        FROM DUAL;
    EXCEPTION
        WHEN OTHERS THEN
            l_result_count := 0;
    END;
    
    -- Log the request (autonomous transaction)
    BEGIN
        INSERT INTO API_SEARCH_LOG (
            search_id, query_text, top_k, response_time_ms, 
            result_count, error_message
        ) VALUES (
            l_search_id, p_query_text, p_top_k, l_response_time_ms,
            l_result_count, NULL
        );
        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            NULL; -- Don't fail the request if logging fails
    END;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Log error
        BEGIN
            INSERT INTO API_SEARCH_LOG (
                search_id, query_text, top_k, error_message
            ) VALUES (
                l_search_id, p_query_text, p_top_k, SQLERRM
            );
            COMMIT;
        EXCEPTION
            WHEN OTHERS THEN
                NULL;
        END;
        
        p_result_json := JSON_OBJECT(
            'error' VALUE 'Unexpected error: ' || SQLERRM,
            'search_id' VALUE l_search_id
        );
END GET_SEMANTIC_RECOMMENDATIONS;
/
```

#### 7.3. Create Monitoring Queries

```sql
-- View recent search activity
SELECT 
    log_id,
    search_id,
    SUBSTR(query_text, 1, 50) AS query_preview,
    response_time_ms,
    result_count,
    request_timestamp
FROM API_SEARCH_LOG
ORDER BY request_timestamp DESC
FETCH FIRST 20 ROWS ONLY;

-- Performance statistics
SELECT 
    COUNT(*) AS total_searches,
    ROUND(AVG(response_time_ms), 2) AS avg_response_ms,
    ROUND(MAX(response_time_ms), 2) AS max_response_ms,
    ROUND(MIN(response_time_ms), 2) AS min_response_ms,
    COUNT(CASE WHEN error_message IS NOT NULL THEN 1 END) AS errors
FROM API_SEARCH_LOG
WHERE request_timestamp >= SYSTIMESTAMP - INTERVAL '24' HOUR;

-- Error analysis
SELECT 
    error_message,
    COUNT(*) AS occurrence_count,
    MAX(request_timestamp) AS last_occurrence
FROM API_SEARCH_LOG
WHERE error_message IS NOT NULL
GROUP BY error_message
ORDER BY occurrence_count DESC;
```

---

## 5. Troubleshooting Guide

### 5.1. Common Issues

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| ORDS won't start | Port already in use | Change port in ORDS config or kill existing process |
| 404 Not Found | Module/template not created | Verify ORDS configuration with queries in Step 4 |
| 500 Internal Error | PL/SQL procedure error | Check alert.log and trace files in database |
| Slow response times | Inefficient vector search | Verify HNSW index exists, gather statistics |
| Connection timeout | ORDS not accessible | Check firewall rules, verify ORDS is running |

### 5.2. Debugging Commands

```sql
-- Check ORDS schema status
SELECT * FROM user_ords_schemas;

-- Check all modules
SELECT * FROM user_ords_modules;

-- Check all templates
SELECT * FROM user_ords_templates;

-- Check all handlers
SELECT * FROM user_ords_handlers;

-- View ORDS privileges
SELECT * FROM user_ords_privileges;
```

### 5.3. ORDS Logs

```bash
# View ORDS logs
tail -f /opt/oracle/ords/logs/ords.log

# Check for errors
grep ERROR /opt/oracle/ords/logs/ords.log
```

---

## 6. Performance Tuning

### 6.1. Connection Pool Tuning

Edit `/opt/oracle/ords/config/databases/default/pool.xml`:

```xml
<pool>
  <min-limit>5</min-limit>
  <max-limit>50</max-limit>
  <max-statements>20</max-statements>
</pool>
```

### 6.2. Enable ORDS Caching

```sql
-- Enable result caching
ALTER PROCEDURE GET_SEMANTIC_RECOMMENDATIONS COMPILE;

-- Use result cache hint in queries (if appropriate)
```

---

## 7. Next Steps

Once this TDD is complete, proceed to:
- **TDD 4**: Integrate the API into Siebel CRM Open UI
- Use the API endpoint URL and authentication details from this setup
- The API is now ready to be called from Siebel eScript

