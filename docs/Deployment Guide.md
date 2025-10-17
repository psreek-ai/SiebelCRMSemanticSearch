# Deployment Guide: AI-Powered Semantic Search Solution

**Version:** 2.2 (Complete Implementation)
**Date:** 2025-10-17
**Audience:** Deployment Team, Database Administrators, System Administrators, Siebel Administrators

## 1. Introduction
This guide provides a comprehensive, step-by-step process for deploying the AI-Powered Semantic Search solution into a target environment (DEV, UAT, or Production). This deployment uses **Oracle Autonomous Database on Azure** (provisioned through Oracle Database Service for Azure - ODSA), and Siebel CRM. The managed nature of Autonomous Database significantly simplifies deployment by eliminating manual database configuration and ORDS installation steps. **Follow these steps in the exact sequence specified.**

## 2. Pre-Deployment Checklist

### 2.1. Infrastructure Requirements

**Oracle Autonomous Database on Azure:**
- [ ] Azure subscription with appropriate permissions to provision ODSA resources
- [ ] Oracle Cloud Infrastructure (OCI) account linked to Azure subscription via ODSA
- [ ] Autonomous Database instance provisioned (Data Warehouse workload type recommended)
- [ ] Minimum 1 OCPU allocated (scales automatically based on workload)
- [ ] Minimum 1TB storage allocated (auto-scales as needed)
- [ ] Private endpoint configured for secure connectivity to Azure VNet
- [ ] Network connectivity from Autonomous Database to Oracle 12c Siebel database (via VNet peering or VPN)
- [ ] ORDS is pre-configured and enabled (no manual installation required)

**Oracle Database 12c (Siebel):**
- [ ] Read-only user account created with access to Siebel schema
- [ ] Listener configured and accessible from Autonomous Database (via VNet peering)
- [ ] Network firewall rules allow connections from Autonomous Database subnet

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
# Oracle Autonomous Database on Azure
ADB_INSTANCE_NAME=<autonomous_db_name>
ADB_ADMIN_PASSWORD=<secure_admin_password>
ADB_SERVICE_NAME=<service_name>_high  # Use _high for best performance
ADB_ORDS_URL=https://<unique_id>-<db_name>.adb.<region>.oraclecloudapps.com/ords/
ADB_WALLET_PATH=<path_to_downloaded_wallet>
ADB_CONNECTION_STRING=<from_wallet_tnsnames.ora>

# Azure Networking
AZURE_VNET_NAME=<vnet_name>
AZURE_SUBNET_NAME=<subnet_name>
ADB_PRIVATE_ENDPOINT=<private_endpoint_name>

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

# Siebel Configuration
SIEBEL_WEB_ROOT=<path_to_siebsrvr/public/enu/files>
SIEBEL_SRF_PATH=<path_to_siebsrvr/objects>
SIEBEL_SERVER_NAME=<siebel_server_name>

# Security
API_KEY=<generate_secure_api_key>
```

---

## 3. Deployment Phases

## Phase 1: Provision and Configure Oracle Autonomous Database

**Duration:** 1-2 hours (significantly reduced due to managed service)  
**Executor:** Cloud Administrator / Database Administrator  
**Rollback Time:** < 15 minutes

### Step 1.1: Provision Oracle Autonomous Database via Azure Portal

**Navigate to Oracle Database Service for Azure (ODSA) in Azure Portal:**

1. In Azure Portal, search for "Oracle Database@Azure"
2. Click **+ Create** to start provisioning a new Autonomous Database
3. Configure the instance with the following settings:

**Basic Configuration:**
```
Subscription: <your_azure_subscription>
Resource Group: <create_new_or_select_existing>
Region: <select_azure_region_with_odsa_support>
Database Name: semantic-search-adb
Display Name: Semantic Search Vector Database
Workload Type: Data Warehouse  (optimized for analytics and vector search)
Database Version: 19c or later (supports AI Vector Search)
Deployment Type: Serverless
```

**Compute and Storage:**
```
OCPU Count: 1 (auto-scales based on workload)
Storage (TB): 1 (auto-expands as needed)
Auto Scaling: Enabled
```

**Networking:**
```
Network Access Type: Private endpoint access
Virtual Network: <select_your_azure_vnet>
Subnet: <select_subnet_with_connectivity_to_siebel>
```

**Administrator Credentials:**
```
Administrator Username: ADMIN (default)
Administrator Password: <create_strong_password>
Confirm Password: <confirm_password>
```

4. Click **Review + Create**, then **Create**
5. Wait for deployment to complete (typically 5-10 minutes)

**Post-Provisioning:**
- Download the **Connection Wallet** from the Azure Portal (contains connection strings and certificates)
- Note the **ORDS URL** - it's pre-configured and displayed in the database details

### Step 1.2: Connect to Autonomous Database and Create Application Schema

```sql
-- Connect as ADMIN user using SQL*Plus or SQL Developer
-- Use the connection string from the wallet (service with _high suffix for best performance)
sqlplus ADMIN/<password>@<service_name>_high

-- Create application user
CREATE USER SEMANTIC_SEARCH IDENTIFIED BY "<SecurePassword123!>"
  DEFAULT TABLESPACE DATA
  TEMPORARY TABLESPACE TEMP
  QUOTA UNLIMITED ON DATA;

-- Grant basic privileges
GRANT CREATE SESSION TO SEMANTIC_SEARCH;
GRANT CREATE TABLE TO SEMANTIC_SEARCH;
GRANT CREATE VIEW TO SEMANTIC_SEARCH;
GRANT CREATE PROCEDURE TO SEMANTIC_SEARCH;
GRANT CREATE SEQUENCE TO SEMANTIC_SEARCH;
GRANT CREATE JOB TO SEMANTIC_SEARCH;

-- Grant DBMS_CLOUD privileges for OCI API calls (pre-configured in ADB)
GRANT EXECUTE ON DBMS_CLOUD TO SEMANTIC_SEARCH;
GRANT EXECUTE ON DBMS_CLOUD_AI TO SEMANTIC_SEARCH;

-- Note: Network ACLs for OCI services are pre-configured in Autonomous Database
-- No manual ACL configuration needed for *.oraclecloud.com

COMMIT;

-- Verify user creation
SELECT username, account_status, default_tablespace
FROM dba_users
WHERE username = 'SEMANTIC_SEARCH';
```

**Verification:**
```sql
-- Connect as SEMANTIC_SEARCH
connect SEMANTIC_SEARCH/<password>@<service_name>_high

-- Test privileges
SELECT * FROM session_privs;
-- Expected: CREATE SESSION, CREATE TABLE, CREATE PROCEDURE, etc.

-- Verify ORDS is enabled (pre-configured)
SELECT * FROM user_ords_schemas;
-- May return no rows initially, will enable in Phase 3
```

### Step 1.3: Configure Network Connectivity to Oracle 12c Siebel Database

**Important:** Autonomous Database requires secure network connectivity to access the Oracle 12c database on Azure VM.

**Option A: VNet Peering (Recommended for Azure-to-Azure connectivity)**

1. Configure Azure VNet peering between:
   - The VNet where Autonomous Database private endpoint resides
   - The VNet where Oracle 12c VM resides

2. Update Network Security Groups (NSGs) to allow traffic:
```
Source: Autonomous Database subnet CIDR
Destination: Oracle 12c VM IP address
Port: 1521
Protocol: TCP
```

**Option B: VPN Gateway (For secure connectivity across regions)**

1. Set up Azure VPN Gateway
2. Configure site-to-site VPN between VNets
3. Update route tables to direct traffic appropriately

### Step 1.4: Create Database Link to Oracle 12c

**Note:** In Autonomous Database, TNS entries are managed differently. Use a full connection string.

```sql
-- Connect as SEMANTIC_SEARCH
sqlplus SEMANTIC_SEARCH/<password>@<service_name>_high

-- Create database link using full connection descriptor
CREATE DATABASE LINK SIEBEL_12C_LINK
  CONNECT TO siebel_readonly IDENTIFIED BY "<password>"
  USING '(DESCRIPTION =
           (ADDRESS = (PROTOCOL = TCP)(HOST = <DB_12C_HOST>)(PORT = 1521))
           (CONNECT_DATA =
             (SERVER = DEDICATED)
             (SERVICE_NAME = <DB_12C_SERVICE>)
           )
         )';

-- Test database link
SELECT COUNT(*) FROM SIEBEL.S_SRV_REQ@SIEBEL_12C_LINK;
-- Expected: A number (e.g., 1500000)

-- Verify connectivity to key tables
SELECT 'S_SRV_REQ' AS TABLE_NAME, COUNT(*) AS ROW_COUNT FROM SIEBEL.S_SRV_REQ@SIEBEL_12C_LINK
UNION ALL
SELECT 'S_ORDER', COUNT(*) FROM SIEBEL.S_ORDER@SIEBEL_12C_LINK
UNION ALL
SELECT 'S_EVT_ACT', COUNT(*) FROM SIEBEL.S_EVT_ACT@SIEBEL_12C_LINK;
```

**Troubleshooting:**
- **ORA-12154**: Check connection string syntax in database link definition
- **ORA-01017**: Verify siebel_readonly username/password
- **ORA-12541**: Verify network connectivity between ADB and Oracle 12c VM
  - Check NSG rules on Oracle 12c VM
  - Verify VNet peering status
  - Confirm Oracle 12c listener is running: `lsnrctl status`
- **ORA-12170**: Check if firewall is blocking port 1521

### Step 1.5: Execute TDD 1 Scripts

Follow **TDD 1** completely (updated for Autonomous Database):

```sql
-- Connect as SEMANTIC_SEARCH using the _high service for best performance
sqlplus SEMANTIC_SEARCH/<password>@<service_name>_high

-- 1. Create staging tables (Step 2.2)
@create_staging_tables.sql

-- 2. Copy data from 12c (Step 3.1)
-- Note: This may take longer due to cross-network data transfer
-- Consider running during off-peak hours
@copy_data_from_12c.sql

-- 3. Create views (Step 4)
@create_aggregation_views.sql

-- 4. Create and populate staging table (Step 5)
@create_narrative_staging.sql

-- 5. Create refresh procedure (Step 6.1)
@create_refresh_procedure.sql

-- 6. Schedule refresh job (Step 6.2)
-- Note: Autonomous Database automatically manages job scheduling
@schedule_refresh_job.sql

-- Verify completion
SELECT 
    'STG_S_SRV_REQ' AS TABLE_NAME, COUNT(*) FROM STG_S_SRV_REQ
UNION ALL
SELECT 'SIEBEL_NARRATIVES_STAGING', COUNT(*) FROM SIEBEL_NARRATIVES_STAGING;
```

**Expected Duration:** 30-60 minutes for initial data load

**Performance Note:** The private, high-speed interconnect between Azure and OCI via ODSA ensures optimal data transfer performance.

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

## Phase 3: ORDS API Configuration

**Duration:** 30 minutes (significantly reduced - ORDS is pre-installed!)  
**Executor:** Database Administrator  
**Rollback Time:** < 10 minutes

**Key Advantage:** Oracle Autonomous Database includes Oracle REST Data Services (ORDS) pre-installed, pre-configured, and ready to use. This eliminates the need for manual ORDS installation, configuration, and maintenance.

### Step 3.1: Verify ORDS is Enabled and Accessible

**No installation required!** ORDS is automatically configured when you provision Autonomous Database.

```sql
-- Connect as ADMIN to verify ORDS configuration
sqlplus ADMIN/<password>@<service_name>_high

-- Check ORDS status
SELECT * FROM DBA_ORDS_SCHEMAS;
-- This shows which schemas are REST-enabled

-- Verify ORDS URL (provided during provisioning)
-- Format: https://<unique_id>-<db_name>.adb.<region>.oraclecloudapps.com/ords/
```

**Test ORDS Accessibility:**
```bash
# From your local machine or Azure VM, test the ORDS URL
curl https://<your_adb_ords_url>/ords/

# Expected: HTML response showing ORDS is running
# OR JSON response with database information
```

### Step 3.2: Enable SEMANTIC_SEARCH Schema for REST Services

```sql
-- Connect as SEMANTIC_SEARCH
sqlplus SEMANTIC_SEARCH/<password>@<service_name>_high

-- Enable schema for ORDS REST services
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

-- Verify schema is REST-enabled
SELECT name, enabled, url_mapping_type, url_mapping_pattern
FROM user_ords_schemas;

-- Expected output:
-- NAME               ENABLED  URL_MAPPING_TYPE  URL_MAPPING_PATTERN
-- SEMANTIC_SEARCH    TRUE     BASE_PATH         semantic_search
```

### Step 3.3: Execute TDD 3 Scripts

Follow **TDD 3** (with modifications for Autonomous Database):

```sql
-- Connect as SEMANTIC_SEARCH
sqlplus SEMANTIC_SEARCH/<password>@<service_name>_high

-- 1. Schema is already REST-enabled (completed in Step 3.2)

-- 2. Create search procedures (Step 3 from TDD 3)
@create_search_procedures.sql

-- 3. Register ORDS endpoints (Step 4 from TDD 3)
@register_ords_endpoints.sql

-- 4. Configure security (Step 6 from TDD 3)
@configure_ords_security.sql

-- 5. Create logging (Step 7 from TDD 3)
@create_api_logging.sql
```

### Step 3.4: Test API Endpoint

**Note:** Use the HTTPS URL provided by Autonomous Database (ORDS is accessed via HTTPS, not HTTP).

```bash
# Test API with curl using the Autonomous Database ORDS URL
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "X-API-Key: <API_KEY>" \
  -H "Top-K: 5" \
  -d "My computer is running very slow" \
  https://<unique_id>-<db_name>.adb.<region>.oraclecloudapps.com/ords/semantic_search/siebel/search

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
  https://<unique_id>-<db_name>.adb.<region>.oraclecloudapps.com/ords/semantic_search/siebel/search

# Expected: Error response with appropriate message
```

**Troubleshooting:**
- **Connection refused**: Verify the ORDS URL is correct from Azure portal
- **SSL certificate errors**: Use `-k` flag with curl for testing (not recommended for production)
- **401 Unauthorized**: Check API key configuration
- **404 Not Found**: Verify schema is REST-enabled and endpoints are registered

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
-- Connect to Oracle Autonomous Database
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
