# Deployment Guide: AI-Powered Semantic Search Solution

**Version:** 2.2 (Complete Implementation)
**Date:** 2025-10-17
**Audience:** Deployment Team, Database Administrators, System Administrators, Siebel Administrators

## 1. Introduction
This guide provides a comprehensive, step-by-step process for deploying the AI-Powered Semantic Search solution into a target environment (DEV, UAT, or Production). This deployment uses Oracle Database 23ai, ORDS, and Siebel CRM. **Follow these steps in the exact sequence specified.**

## 2. Pre-Deployment Checklist

### 2.1. Infrastructure Requirements

**Oracle Database 23ai:**
- [ ] Oracle Database 23ai Enterprise Edition installed and running
- [ ] Minimum 100GB free space in USERS tablespace
- [ ] SGA minimum 8GB, PGA minimum 4GB
- [ ] Network connectivity to Oracle 12c Siebel database
- [ ] Network connectivity to OCI Generative AI endpoints
- [ ] ORDS 23.x installed and configured

**Oracle Database 12c (Siebel):**
- [ ] Read-only user account created with access to Siebel schema
- [ ] Listener configured and accessible from Oracle 23ai server
- [ ] Network firewall rules allow connections from Oracle 23ai

**OCI Account:**
- [ ] OCI account with access to Generative AI service
- [ ] Compartment OCID documented
- [ ] API keys generated (user OCID, tenancy OCID, private key, fingerprint)
- [ ] Region noted (e.g., us-ashburn-1)

**Siebel CRM:**
- [ ] Siebel Tools access with repository check-out privileges
- [ ] Siebel Web Server access for file deployment
- [ ] Siebel Application Server access for SRF deployment
- [ ] Administrative access to Siebel client

### 2.2. Configuration Values Document

Create a deployment configuration document with these values:

```
# Oracle 23ai Database
DB_23AI_HOST=<hostname>
DB_23AI_PORT=1521
DB_23AI_SERVICE=<service_name>

# Oracle 12c Siebel Database
DB_12C_HOST=<hostname>
DB_12C_PORT=1521
DB_12C_SERVICE=<service_name>
DB_12C_READONLY_USER=siebel_readonly
DB_12C_READONLY_PWD=<password>

# OCI Configuration
OCI_USER_OCID=ocid1.user.oc1..aaaaaaaa...
OCI_TENANCY_OCID=ocid1.tenancy.oc1..aaaaaaaa...
OCI_FINGERPRINT=a1:b2:c3:...
OCI_PRIVATE_KEY=<path_to_private_key>
OCI_COMPARTMENT_OCID=ocid1.compartment.oc1..aaaaaaaa...
OCI_REGION=us-ashburn-1

# ORDS Configuration
ORDS_PORT=8080
ORDS_PROTOCOL=http
ORDS_HOST=<hostname_or_ip>

# Siebel Configuration
SIEBEL_WEB_ROOT=<path_to_siebsrvr/public/enu/files>
SIEBEL_SRF_PATH=<path_to_siebsrvr/objects>
SIEBEL_SERVER_NAME=<siebel_server_name>

# Security
API_KEY=<generate_secure_api_key>
```

---

## 3. Deployment Phases

## Phase 1: Oracle 23ai Database Setup

**Duration:** 2-3 hours  
**Executor:** Database Administrator  
**Rollback Time:** < 30 minutes

### Step 1.1: Create Database User and Grant Privileges

```sql
-- Connect as SYSDBA
sqlplus / as sysdba

-- Create user
CREATE USER SEMANTIC_SEARCH IDENTIFIED BY "<SecurePassword123!>"
  DEFAULT TABLESPACE USERS
  TEMPORARY TABLESPACE TEMP
  QUOTA UNLIMITED ON USERS;

-- Grant basic privileges
GRANT CREATE SESSION TO SEMANTIC_SEARCH;
GRANT CREATE TABLE TO SEMANTIC_SEARCH;
GRANT CREATE VIEW TO SEMANTIC_SEARCH;
GRANT CREATE PROCEDURE TO SEMANTIC_SEARCH;
GRANT CREATE SEQUENCE TO SEMANTIC_SEARCH;
GRANT CREATE JOB TO SEMANTIC_SEARCH;

-- Grant DBMS_CLOUD privileges for OCI API calls
GRANT EXECUTE ON DBMS_CLOUD TO SEMANTIC_SEARCH;
GRANT EXECUTE ON DBMS_CLOUD_AI TO SEMANTIC_SEARCH;

-- Configure Network ACL for OCI API access
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host => '*.oraclecloud.com',
    ace  => xs$ace_type(
      privilege_list => xs$name_list('http', 'connect', 'resolve'),
      principal_name => 'SEMANTIC_SEARCH',
      principal_type => xs_acl.ptype_db
    )
  );
END;
/

COMMIT;

-- Verify user creation
SELECT username, account_status, default_tablespace
FROM dba_users
WHERE username = 'SEMANTIC_SEARCH';
```

**Verification:**
```sql
-- Connect as SEMANTIC_SEARCH
connect SEMANTIC_SEARCH/<password>@<service_name>

-- Test privileges
SELECT * FROM session_privs;
-- Expected: CREATE SESSION, CREATE TABLE, CREATE PROCEDURE, etc.
```

### Step 1.2: Create Database Link to Oracle 12c

```bash
# First, configure TNS entry
vi $ORACLE_HOME/network/admin/tnsnames.ora

# Add entry:
SIEBEL_12C_DB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <DB_12C_HOST>)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = <DB_12C_SERVICE>)
    )
  )

# Test TNS connectivity
tnsping SIEBEL_12C_DB
```

```sql
-- Connect as SEMANTIC_SEARCH
sqlplus SEMANTIC_SEARCH/<password>@<service_name>

-- Create database link
CREATE DATABASE LINK SIEBEL_12C_LINK
  CONNECT TO siebel_readonly IDENTIFIED BY "<password>"
  USING 'SIEBEL_12C_DB';

-- Test database link
SELECT COUNT(*) FROM SIEBEL.S_SRV_REQ@SIEBEL_12C_LINK;
-- Expected: A number (e.g., 1500000)
```

**Troubleshooting:**
- If ORA-12154: Check tnsnames.ora syntax
- If ORA-01017: Verify username/password
- If ORA-12541: Check Oracle 12c listener status

### Step 1.3: Execute TDD 1 Scripts

Follow **TDD 1** completely:

```sql
-- Connect as SEMANTIC_SEARCH
-- Execute all scripts from TDD 1 in sequence:

-- 1. Create staging tables (Step 2.2)
@create_staging_tables.sql

-- 2. Copy data from 12c (Step 3.1)
@copy_data_from_12c.sql

-- 3. Create views (Step 4)
@create_aggregation_views.sql

-- 4. Create and populate staging table (Step 5)
@create_narrative_staging.sql

-- 5. Create refresh procedure (Step 6.1)
@create_refresh_procedure.sql

-- 6. Schedule refresh job (Step 6.2)
@schedule_refresh_job.sql

-- Verify completion
SELECT 
    'STG_S_SRV_REQ' AS TABLE_NAME, COUNT(*) FROM STG_S_SRV_REQ
UNION ALL
SELECT 'SIEBEL_NARRATIVES_STAGING', COUNT(*) FROM SIEBEL_NARRATIVES_STAGING;
```

**Expected Duration:** 30-60 minutes for initial data load

---

## Phase 2: Vector Generation and Indexing

**Duration:** Several hours (depends on data volume)  
**Executor:** Database Administrator  
**Rollback Time:** N/A (can re-run)

### Step 2.1: Create OCI Credentials

```sql
-- Connect as SEMANTIC_SEARCH
sqlplus SEMANTIC_SEARCH/<password>@<service_name>

-- Create OCI credential
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'OCI_GENAI_CREDENTIAL',
    user_ocid       => '<OCI_USER_OCID>',
    tenancy_ocid    => '<OCI_TENANCY_OCID>',
    private_key     => '-----BEGIN PRIVATE KEY-----
<paste_full_private_key_here>
-----END PRIVATE KEY-----',
    fingerprint     => '<OCI_FINGERPRINT>'
  );
END;
/

-- Verify credential
SELECT credential_name, username, enabled
FROM USER_CREDENTIALS
WHERE credential_name = 'OCI_GENAI_CREDENTIAL';
```

### Step 2.2: Execute TDD 2 Scripts

Follow **TDD 2** completely:

```sql
-- 1. Create vector table (Step 2.1)
@create_vector_table.sql

-- 2. Create helper functions (Step 3.1)
@create_clean_text_function.sql

-- 3. Create embedding procedure (Step 3.2)
@create_embedding_procedure.sql

-- 4. Test with small batch (Step 3.3)
SET SERVEROUTPUT ON SIZE UNLIMITED;
BEGIN
    GENERATE_EMBEDDINGS_BATCH(
        p_batch_size => 5,
        p_compartment_id => '<OCI_COMPARTMENT_OCID>',
        p_region => '<OCI_REGION>'
    );
END;
/

-- 5. Process all narratives (Step 4)
-- WARNING: This will take several hours for large datasets
-- Consider running in background or scheduling
@process_all_narratives.sql

-- Monitor progress
SELECT 
    COUNT(*) AS TOTAL,
    COUNT(CASE WHEN PROCESSED_FLAG = 'Y' THEN 1 END) AS PROCESSED,
    COUNT(CASE WHEN PROCESSED_FLAG = 'N' THEN 1 END) AS PENDING,
    ROUND(COUNT(CASE WHEN PROCESSED_FLAG = 'Y' THEN 1 END) * 100.0 / COUNT(*), 2) AS PCT_COMPLETE
FROM SIEBEL_NARRATIVES_STAGING;
```

### Step 2.3: Create Vector Index

**IMPORTANT:** Only create after embeddings are complete!

```sql
-- Verify embeddings are complete
SELECT COUNT(*) 
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE NARRATIVE_VECTOR IS NULL;
-- Expected: 0

-- Create HNSW vector index
CREATE VECTOR INDEX SIBL_KNOW_VEC_HNSW_IDX 
ON SIEBEL_KNOWLEDGE_VECTORS (NARRATIVE_VECTOR)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE
WITH TARGET ACCURACY 95;

-- Monitor index creation
SELECT index_name, status, last_analyzed
FROM user_indexes
WHERE index_name = 'SIBL_KNOW_VEC_HNSW_IDX';

-- Gather statistics
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS(
        ownname => 'SEMANTIC_SEARCH',
        tabname => 'SIEBEL_KNOWLEDGE_VECTORS',
        estimate_percent => 100,
        cascade => TRUE
    );
END;
/
```

**Expected Duration:** 30-60 minutes for index creation

---

## Phase 3: ORDS API Deployment

**Duration:** 1 hour  
**Executor:** Database Administrator  
**Rollback Time:** < 15 minutes

### Step 3.1: Install and Configure ORDS

```bash
# Download ORDS
cd /opt/oracle
wget https://download.oracle.com/otn_software/java/ords/ords-latest.zip
unzip ords-latest.zip -d ords

# Configure ORDS
cd /opt/oracle/ords
export ORDS_HOME=/opt/oracle/ords
export PATH=$ORDS_HOME/bin:$PATH

# Run installation
ords install

# Enter configuration values when prompted:
# Database hostname: <DB_23AI_HOST>
# Database port: 1521
# Database service: <DB_23AI_SERVICE>
# Administrator username: SYS
# Administrator password: <sys_password>

# Start ORDS
ords --config /opt/oracle/ords/config serve &

# Verify ORDS is running
curl http://localhost:8080/ords/
# Expected: HTML welcome page
```

### Step 3.2: Execute TDD 3 Scripts

Follow **TDD 3** completely:

```sql
-- Connect as SEMANTIC_SEARCH
sqlplus SEMANTIC_SEARCH/<password>@<service_name>

-- 1. Enable schema for ORDS (Step 2.1)
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

-- 2. Create search procedures (Step 3)
@create_search_procedures.sql

-- 3. Register ORDS endpoints (Step 4)
@register_ords_endpoints.sql

-- 4. Configure security (Step 6)
@configure_ords_security.sql

-- 5. Create logging (Step 7)
@create_api_logging.sql
```

### Step 3.3: Test API Endpoint

```bash
# Test API with curl
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "X-API-Key: <API_KEY>" \
  -H "Top-K: 5" \
  -d "My computer is running very slow" \
  http://<ORDS_HOST>:<ORDS_PORT>/ords/semantic_search/siebel/search

# Expected output: JSON with recommendations
{
  "search_id": "...",
  "query": "My computer is running very slow",
  "timestamp": "2025-10-17T...",
  "recommendations": [...]
}

# Test error handling
curl -X POST \
  -H "Content-Type: text/plain" \
  http://<ORDS_HOST>:<ORDS_PORT>/ords/semantic_search/siebel/search

# Expected: Error response
```

---

## Phase 4: Siebel CRM Integration

**Duration:** 2-3 hours  
**Executor:** Siebel Administrator / Developer  
**Rollback Time:** < 1 hour

### Step 4.1: Configure Named Subsystem

1. Log in to Siebel Client as Administrator
2. Navigate to: **Administration - Server Configuration > Profile Configuration**
3. Create Named Subsystem: **SemanticSearchORDS**
4. Add parameters:
   - URL: `http://<ORDS_HOST>:<ORDS_PORT>/ords/semantic_search/siebel/search`
   - APIKey: `<API_KEY>`
   - Timeout: `5000`
   - MaxRetries: `2`

**Verification SQL:**
```sql
SELECT 
    sp.NAME AS SUBSYSTEM,
    spp.NAME AS PARAMETER,
    spp.VAL AS VALUE
FROM SIEBEL.S_SYS_PROF sp
JOIN SIEBEL.S_SYS_PROF_PROP spp ON sp.ROW_ID = spp.PAR_ROW_ID
WHERE sp.NAME = 'SemanticSearchORDS';
```

### Step 4.2: Deploy Repository Changes

Follow **TDD 4** completely:

**In Siebel Tools:**

1. Create Business Service:
   - Name: `SemanticSearchAPIService`
   - Import eScript from TDD 4, Step 2.4

2. Compile objects:
   ```
   Tools > Compile > Selected Objects
   ```

3. Check in objects:
   ```
   Tools > Check In
   Select: SemanticSearchAPIService
   Comments: "Semantic search integration - v2.2"
   ```

4. Generate SRF:
   ```bash
   cd <SIEBEL_TOOLS_HOME>/bin
   ./srvrmgr -g <gateway> -e <enterprise> -u SADMIN -p <password>
   
   srvrmgr> list server
   srvrmgr> compile srf for server <SIEBEL_SERVER_NAME>
   srvrmgr> exit
   ```

### Step 4.3: Deploy Web Files

```bash
# Copy custom JavaScript files
cp CatalogSearchPM.js <SIEBEL_WEB_ROOT>/custom/
cp manifest_search.js <SIEBEL_WEB_ROOT>/custom/

# Set permissions
chmod 644 <SIEBEL_WEB_ROOT>/custom/*.js
chown siebel:siebel <SIEBEL_WEB_ROOT>/custom/*.js
```

### Step 4.4: Deploy SRF File

```bash
# Stop Siebel servers
cd <SIEBEL_ROOT>/siebsrvr/bin
./stop_ns

# Backup old SRF
cp ../objects/siebel.srf ../objects/siebel.srf.backup_$(date +%Y%m%d)

# Copy new SRF
cp /tmp/siebel.srf ../objects/

# Start Siebel servers
./start_ns

# Verify servers are running
./srvrmgr -g <gateway> -e <enterprise> -u SADMIN -p <password>
srvrmgr> list comp
# Expected: All components running
```

### Step 4.5: Clear Caches

```bash
# Clear browser cache directories
rm -rf <SIEBEL_WEB_ROOT>/cache/*

# Restart Apache/IIS
systemctl restart httpd
# OR
iisreset
```

---

## 4. Post-Deployment Verification

### 4.1. Database Verification

```sql
-- Connect to Oracle 23ai
sqlplus SEMANTIC_SEARCH/<password>@<service_name>

-- Verify data load
SELECT 
    'Staging Records' AS METRIC, COUNT(*) AS COUNT
FROM SIEBEL_NARRATIVES_STAGING
UNION ALL
SELECT 'Vector Records', COUNT(*)
FROM SIEBEL_KNOWLEDGE_VECTORS
UNION ALL
SELECT 'Vectors Generated', COUNT(*)
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE NARRATIVE_VECTOR IS NOT NULL;

-- Verify index
SELECT index_name, status, last_analyzed
FROM user_indexes
WHERE table_name = 'SIEBEL_KNOWLEDGE_VECTORS';

-- Verify scheduled jobs
SELECT job_name, enabled, state, next_run_date
FROM user_scheduler_jobs
WHERE job_name IN ('DAILY_STAGING_REFRESH', 'DAILY_EMBEDDING_GENERATION');
```

### 4.2. API Verification

```bash
# Test API health
curl http://<ORDS_HOST>:<ORDS_PORT>/ords/

# Test search endpoint
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "X-API-Key: <API_KEY>" \
  -H "Top-K: 3" \
  -d "printer not working" \
  http://<ORDS_HOST>:<ORDS_PORT>/ords/semantic_search/siebel/search | jq .

# Verify response structure
# Expected fields: search_id, query, timestamp, recommendations
```

### 4.3. Siebel Verification

1. Log in to Siebel application
2. Navigate to Service Request screen
3. Enter test query: "laptop battery not charging"
4. Click Search
5. Verify:
   - Loading indicator appears
   - Results display in ranked order
   - No errors in browser console
   - Network tab shows successful API call

### 4.4. End-to-End Test

```bash
# Monitor API logs during Siebel search
tail -f /opt/oracle/ords/logs/ords.log

# Monitor database API logs
sqlplus SEMANTIC_SEARCH/<password>@<service_name>

SELECT 
    log_id,
    search_id,
    SUBSTR(query_text, 1, 50) AS query,
    response_time_ms,
    result_count,
    request_timestamp
FROM API_SEARCH_LOG
ORDER BY request_timestamp DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## 5. Rollback Procedures

### 5.1. Rollback Siebel Changes

```bash
# Stop Siebel servers
cd <SIEBEL_ROOT>/siebsrvr/bin
./stop_ns

# Restore old SRF
cp ../objects/siebel.srf.backup_<date> ../objects/siebel.srf

# Remove custom files
rm <SIEBEL_WEB_ROOT>/custom/CatalogSearchPM.js
rm <SIEBEL_WEB_ROOT>/custom/manifest_search.js

# Start Siebel servers
./start_ns

# Clear caches
rm -rf <SIEBEL_WEB_ROOT>/cache/*
systemctl restart httpd
```

### 5.2. Rollback ORDS API

```sql
-- Connect as SEMANTIC_SEARCH
sqlplus SEMANTIC_SEARCH/<password>@<service_name>

-- Delete ORDS handler
BEGIN
    ORDS.DELETE_HANDLER(
        p_module_name => 'siebel_search',
        p_pattern => 'search',
        p_method => 'POST'
    );
    COMMIT;
END;
/

-- Delete template
BEGIN
    ORDS.DELETE_TEMPLATE(
        p_module_name => 'siebel_search',
        p_pattern => 'search'
    );
    COMMIT;
END;
/

-- Delete module
BEGIN
    ORDS.DELETE_MODULE(
        p_module_name => 'siebel_search'
    );
    COMMIT;
END;
/

-- Disable schema
BEGIN
    ORDS.ENABLE_SCHEMA(
        p_enabled => FALSE,
        p_schema => 'SEMANTIC_SEARCH'
    );
    COMMIT;
END;
/
```

### 5.3. Rollback Database Changes

```sql
-- Connect as SYSDBA
sqlplus / as sysdba

-- Drop scheduler jobs
BEGIN
    DBMS_SCHEDULER.DROP_JOB('SEMANTIC_SEARCH.DAILY_STAGING_REFRESH');
    DBMS_SCHEDULER.DROP_JOB('SEMANTIC_SEARCH.DAILY_EMBEDDING_GENERATION');
END;
/

-- Drop vector index
DROP INDEX SEMANTIC_SEARCH.SIBL_KNOW_VEC_HNSW_IDX;

-- Drop tables (optional - only if full rollback needed)
DROP TABLE SEMANTIC_SEARCH.SIEBEL_KNOWLEDGE_VECTORS PURGE;
DROP TABLE SEMANTIC_SEARCH.SIEBEL_NARRATIVES_STAGING PURGE;
DROP TABLE SEMANTIC_SEARCH.STG_S_SRV_REQ PURGE;
DROP TABLE SEMANTIC_SEARCH.STG_S_ORDER PURGE;
DROP TABLE SEMANTIC_SEARCH.STG_S_EVT_ACT PURGE;
DROP TABLE SEMANTIC_SEARCH.STG_S_PROD_INT PURGE;

-- Drop user (only if complete removal needed)
DROP USER SEMANTIC_SEARCH CASCADE;
```

---

## 6. Troubleshooting Common Deployment Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| Database link fails | ORA-12154 or ORA-12541 | Verify tnsnames.ora, check Oracle 12c listener |
| OCI API calls fail | ORA-29532, HTTP 401 | Verify OCI credentials, check Network ACL |
| ORDS won't start | Port in use | Change ORDS port or kill existing process |
| API returns 404 | Endpoint not found | Verify ORDS module/template creation |
| Siebel search hangs | Timeout | Check API_SEARCH_LOG for errors, verify network |
| No search results | Empty response | Verify vector index exists, check data in tables |

---

## 7. Post-Deployment Tasks

### 7.1. Enable Monitoring

```sql
-- Create monitoring view
CREATE OR REPLACE VIEW V_SEARCH_METRICS AS
SELECT 
    TRUNC(request_timestamp, 'HH24') AS hour,
    COUNT(*) AS search_count,
    ROUND(AVG(response_time_ms), 2) AS avg_response_ms,
    ROUND(MAX(response_time_ms), 2) AS max_response_ms,
    COUNT(CASE WHEN error_message IS NOT NULL THEN 1 END) AS error_count
FROM API_SEARCH_LOG
WHERE request_timestamp >= SYSDATE - 7
GROUP BY TRUNC(request_timestamp, 'HH24')
ORDER BY hour DESC;

-- Query metrics
SELECT * FROM V_SEARCH_METRICS;
```

### 7.2. Schedule Maintenance

- Review **TDD 1, Step 6** for data refresh schedule
- Review **TDD 2, Step 4.3** for embedding generation schedule
- Set up alerting for failed jobs

### 7.3. Document Environment-Specific Information

Create a deployment record with:
- Deployment date/time
- Version numbers
- Configuration values used
- Test results
- Known issues
- Contact information

---

## 8. Success Criteria

Deployment is considered successful when:

- ✅ All database tables populated with data
- ✅ Vector index created successfully
- ✅ ORDS API responds with valid JSON
- ✅ Siebel search returns ranked results
- ✅ End-to-end search completes in < 3 seconds
- ✅ No errors in logs
- ✅ Scheduled jobs running successfully
- ✅ Rollback procedures tested and documented

---

## 9. Next Steps

After successful deployment:
1. Conduct User Acceptance Testing (see **Testing Guide**)
2. Train end users on new search functionality
3. Monitor performance and user feedback
4. Plan for production go-live
5. Schedule post-deployment review meeting
