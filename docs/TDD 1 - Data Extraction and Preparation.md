# Technical Design Document 1: Data Extraction and Preparation

**Version:** 3.0  
**Date:** 2025-10-17  
**Owner:** Database Team

## 1. Objective
This document provides the technical specification for extracting and preparing historical service request data from the **Oracle 19c Siebel database** and loading it into **Oracle Database 23ai on Azure VM**. We use **direct database-to-database connectivity** via Oracle Database Links to copy relevant tables and then use SQL to create the consolidated dataset. This approach provides full control over the ETL process and network configuration.

## 2. Architecture Overview

### 2.1. Data Flow
```
Oracle 19c (Siebel database)  →  Azure Private Network  →  Oracle Database 23ai on Azure VM
      ↓                            (VNet connectivity)              ↓
   Source Data                                               Database Link
                                                                          ↓
                                              Copy relevant tables (S_SRV_REQ, S_ORDER, S_EVT_ACT, S_PROD_INT)
                                                                          ↓
                                              Run SQL aggregation query in Oracle 23ai
                                                                          ↓
                                              Populate staging table with narrative text
                                                                          ↓
                                              Generate embeddings via Azure AI Foundry (covered in TDD 2)
```

### 2.2. Primary Source Tables
* `SIEBEL.S_SRV_REQ`: Master table for Service Requests
* `SIEBEL.S_ORDER`: Work Orders linked to Service Requests
* `SIEBEL.S_EVT_ACT`: Activities including email communications
* `SIEBEL.S_PROD_INT`: Service catalog product hierarchy

## 3. Implementation Steps

### Step 1: Create Database Link from Oracle 23ai to Oracle 19c Siebel Database

**Executor:** Database Administrator on Oracle 23ai VM  
**Duration:** 30 minutes

#### 1.1. Configure Network Connectivity Prerequisites

**Important:** Ensure secure network connectivity between Oracle 23ai VM and Oracle 19c Siebel VM on Azure.

**Network Setup Requirements:**

1. **Azure VNet Configuration:**
   - Both VMs should be in the same VNet, or configure VNet peering if in different VNets
   - Verify VMs can reach each other via private IP addresses
   
2. **Network Security Group (NSG) Rules:**
   - Update NSG on Oracle 19c Siebel VM to allow inbound traffic:
     ```
     Source: Oracle 23ai VM private IP or subnet CIDR
     Destination: Oracle 19c VM private IP
     Destination Port: 1521
     Protocol: TCP
     Action: Allow
     Priority: 100
     Name: Allow-Oracle23ai-to-Siebel19c
     ```

3. **Oracle 19c Listener Configuration:**
   - Verify Oracle 19c listener is running and accepting connections
   ```bash
   # SSH to Oracle 19c Siebel VM
   ssh siebel@<siebel-vm-ip>
   
   # Check listener status
   lsnrctl status
   # Expected: Service "<service_name>" has 1+ instance(s)
   ```

4. **Firewall Configuration:**
   ```bash
   # On Oracle 19c Siebel VM, allow port 1521
   sudo firewall-cmd --permanent --add-port=1521/tcp
   sudo firewall-cmd --reload
   ```

#### 1.2. Test Network Connectivity

```bash
# From Oracle 23ai VM, test connectivity to Oracle 19c Siebel VM
ssh oracle@<oracle-23ai-vm-ip>

# Test network connectivity
telnet <siebel-vm-private-ip> 1521
# Expected: Connected to <siebel-vm-private-ip>

# Or using netcat
nc -zv <siebel-vm-private-ip> 1521
# Expected: Connection to <siebel-vm-private-ip> 1521 port [tcp/oracle] succeeded!

# Test with tnsping (if tnsnames.ora is configured)
tnsping <siebel-service-name>
# Expected: OK (time in ms)
```

#### 1.3. Create Database Link in Oracle 23ai

Connect to Oracle 23ai as SEMANTIC_SEARCH user:

```sql
-- SSH to Oracle 23ai VM
ssh oracle@<oracle-23ai-vm-ip>

-- Connect using SQL*Plus
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Create database link with inline connection descriptor to Oracle 19c Siebel database
CREATE DATABASE LINK SIEBEL_19C_LINK
  CONNECT TO siebel_readonly IDENTIFIED BY "<password>"
  USING '(DESCRIPTION =
           (ADDRESS = (PROTOCOL = TCP)(HOST = <siebel-vm-private-ip>)(PORT = 1521))
           (CONNECT_DATA =
             (SERVER = DEDICATED)
             (SERVICE_NAME = <SIEBEL_SERVICE_NAME>)
           )
         )';

-- Test the database link
SELECT COUNT(*) FROM SIEBEL.S_SRV_REQ@SIEBEL_19C_LINK;
-- Expected output: A number (e.g., 1500000+)

-- Verify all required tables are accessible
SELECT 'S_SRV_REQ' AS TABLE_NAME, COUNT(*) AS ROW_COUNT FROM SIEBEL.S_SRV_REQ@SIEBEL_19C_LINK
UNION ALL
SELECT 'S_ORDER', COUNT(*) FROM SIEBEL.S_ORDER@SIEBEL_19C_LINK
UNION ALL
SELECT 'S_EVT_ACT', COUNT(*) FROM SIEBEL.S_EVT_ACT@SIEBEL_19C_LINK
UNION ALL
SELECT 'S_PROD_INT', COUNT(*) FROM SIEBEL.S_PROD_INT@SIEBEL_19C_LINK;

-- Expected output: Row counts for all four tables

EXIT;
```

**Alternative: Add to tnsnames.ora (Optional)**

If you prefer using tnsnames.ora instead of inline descriptors:

```bash
# As oracle user on Oracle 23ai VM
vi $ORACLE_HOME/network/admin/tnsnames.ora

# Add entry:
SIEBEL_19C =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = <siebel-vm-private-ip>)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = <SIEBEL_SERVICE_NAME>)
    )
  )

# Then create database link with tnsnames reference:
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

CREATE DATABASE LINK SIEBEL_19C_LINK
  CONNECT TO siebel_readonly IDENTIFIED BY "<password>"
  USING 'SIEBEL_19C';
```

**Troubleshooting:**
- **ORA-12154**: TNS name not found → Check inline connection descriptor syntax or tnsnames.ora entry
- **ORA-12541**: No listener → Verify Oracle 19c listener is running on Siebel VM: `lsnrctl status`
- **ORA-01017**: Invalid username/password → Verify siebel_readonly user exists and credentials are correct
- **ORA-02019**: Connection description for remote database not found → Check USING clause format
- **ORA-12170**: Connect timeout → Verify:
  - Azure NSG rules allow traffic from Oracle 23ai VM to Siebel VM on port 1521
  - Both VMs are in same VNet or VNet peering is configured
  - Firewall on Siebel VM allows connections from Oracle 23ai VM
  - Oracle 19c listener is configured to accept connections from Oracle 23ai VM IP

---

### Step 2: Create Staging Schema and Tables in Oracle 23ai

**Executor:** Database Administrator on Oracle 23ai VM  
**Duration:** 10 minutes

#### 2.1. Connect to Oracle 23ai

```bash
# SSH to Oracle 23ai VM
ssh oracle@<oracle-23ai-vm-ip>

# Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH
```

#### 2.2. Create Staging Schema (Already Created in Deployment)

**Note:** The SEMANTIC_SEARCH schema was created during deployment (Phase 2, Step 2.4). If not already created, create it now:

```sql
-- Connect as SYS (if schema doesn't exist)
sqlplus / as sysdba

-- Create application user
CREATE USER SEMANTIC_SEARCH IDENTIFIED BY "<secure_password>"
  DEFAULT TABLESPACE USERS
  TEMPORARY TABLESPACE TEMP
  QUOTA UNLIMITED ON USERS;

-- Grant necessary privileges
GRANT CREATE SESSION TO SEMANTIC_SEARCH;
GRANT CREATE TABLE TO SEMANTIC_SEARCH;
GRANT CREATE VIEW TO SEMANTIC_SEARCH;
GRANT CREATE PROCEDURE TO SEMANTIC_SEARCH;
GRANT CREATE SEQUENCE TO SEMANTIC_SEARCH;
GRANT CREATE JOB TO SEMANTIC_SEARCH;
GRANT CREATE DATABASE LINK TO SEMANTIC_SEARCH;

-- Grant privileges for DBMS_CLOUD and network access
GRANT EXECUTE ON DBMS_CLOUD TO SEMANTIC_SEARCH;
GRANT EXECUTE ON UTL_HTTP TO SEMANTIC_SEARCH;

-- Network ACL for Azure AI Foundry was configured in Phase 2, Step 2.4
-- No additional configuration required here

COMMIT;

-- Verify user creation
SELECT username, account_status, default_tablespace
FROM dba_users
WHERE username = 'SEMANTIC_SEARCH';
-- Expected: SEMANTIC_SEARCH | OPEN | USERS

EXIT;
```

#### 2.3. Create Staging Tables

Connect as SEMANTIC_SEARCH user:

```sql
-- Connect to Oracle 23ai
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Create staging tables to hold copies of Siebel data
-- These are materialized copies for better performance during aggregation

-- Service Requests staging table
CREATE TABLE STG_S_SRV_REQ AS
SELECT 
    ROW_ID,
    SR_NUM,
    SR_TITLE,
    DESC_TEXT,
    SR_STAT_ID,
    SP_PRDINT_ID,
    CREATED,
    LAST_UPD
FROM SIEBEL.S_SRV_REQ@SIEBEL_19C_LINK
WHERE 1=0; -- Create empty table with structure

-- Add primary key
ALTER TABLE STG_S_SRV_REQ ADD CONSTRAINT PK_STG_SRV_REQ PRIMARY KEY (ROW_ID);

-- Work Orders staging table
CREATE TABLE STG_S_ORDER AS
SELECT 
    ROW_ID,
    SR_ID,
    ORDER_NUM,
    DESC_TEXT,
    CREATED,
    LAST_UPD
FROM SIEBEL.S_ORDER@SIEBEL_19C_LINK
WHERE 1=0;

-- Add primary key and index
ALTER TABLE STG_S_ORDER ADD CONSTRAINT PK_STG_ORDER PRIMARY KEY (ROW_ID);
CREATE INDEX IDX_STG_ORDER_SR_ID ON STG_S_ORDER(SR_ID);

-- Activities staging table
CREATE TABLE STG_S_EVT_ACT AS
SELECT 
    ROW_ID,
    SRA_SR_ID,
    ORDER_ID,
    COMMENTS,
    CREATED,
    LAST_UPD
FROM SIEBEL.S_EVT_ACT@SIEBEL_19C_LINK
WHERE 1=0;

-- Add primary key and indexes
ALTER TABLE STG_S_EVT_ACT ADD CONSTRAINT PK_STG_EVT_ACT PRIMARY KEY (ROW_ID);
CREATE INDEX IDX_STG_EVT_SR_ID ON STG_S_EVT_ACT(SRA_SR_ID);
CREATE INDEX IDX_STG_EVT_ORDER_ID ON STG_S_EVT_ACT(ORDER_ID);

-- Product catalog staging table
CREATE TABLE STG_S_PROD_INT AS
SELECT 
    ROW_ID,
    NAME,
    PAR_PROD_INT_ID,
    PROD_CD,
    CREATED,
    LAST_UPD
FROM SIEBEL.S_PROD_INT@SIEBEL_19C_LINK
WHERE 1=0;

-- Add primary key and index
ALTER TABLE STG_S_PROD_INT ADD CONSTRAINT PK_STG_PROD_INT PRIMARY KEY (ROW_ID);
CREATE INDEX IDX_STG_PROD_PAR_ID ON STG_S_PROD_INT(PAR_PROD_INT_ID);

-- Gather statistics
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STG_S_SRV_REQ');
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STG_S_ORDER');
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STG_S_EVT_ACT');
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STG_S_PROD_INT');
END;
/
```

#### 2.5. Add Enhanced Columns for Improved Refresh Process

**Important:** These columns support the improved MERGE-based refresh logic and error tracking.

```sql
-- Add timestamp columns to staging table for tracking
ALTER TABLE SIEBEL_NARRATIVES_STAGING 
ADD (
    CREATED_DATE DATE DEFAULT SYSDATE,
    UPDATED_DATE DATE DEFAULT SYSDATE,
    ERROR_MESSAGE VARCHAR2(500)
);

-- Create index on updated date for performance
CREATE INDEX IDX_NARRATIVES_STG_UPD ON SIEBEL_NARRATIVES_STAGING(UPDATED_DATE);

-- Create refresh log table for auditing
CREATE TABLE STAGING_REFRESH_LOG (
    LOG_ID NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    REFRESH_DATE DATE,
    DAYS_LOOKBACK NUMBER,
    SRV_REQ_PROCESSED NUMBER,
    NARRATIVES_PROCESSED NUMBER,
    STATUS VARCHAR2(20),
    ERROR_MESSAGE VARCHAR2(500)
);

-- Add index for log queries
CREATE INDEX IDX_REFRESH_LOG_DATE ON STAGING_REFRESH_LOG(REFRESH_DATE);

-- Gather statistics
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'SIEBEL_NARRATIVES_STAGING');
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STAGING_REFRESH_LOG');
END;
/
```

---

### Step 3: Copy Data from Oracle 19c to Oracle 23ai

**Executor:** Database Administrator on Oracle 23ai  
**Duration:** 30-60 minutes (depending on data volume)

#### 3.1. Initial Full Load
```sql
-- Connect as SEMANTIC_SEARCH user

-- Copy Product Catalog (usually small, copy all)
INSERT /*+ APPEND */ INTO STG_S_PROD_INT
SELECT 
    ROW_ID,
    NAME,
    PAR_PROD_INT_ID,
    PROD_CD,
    CREATED,
    LAST_UPD
FROM SIEBEL.S_PROD_INT@SIEBEL_19C_LINK;

COMMIT;

-- Copy resolved/closed Service Requests (filter for relevant records)
INSERT /*+ APPEND */ INTO STG_S_SRV_REQ
SELECT 
    ROW_ID,
    SR_NUM,
    SR_TITLE,
    DESC_TEXT,
    SR_STAT_ID,
    SP_PRDINT_ID,
    CREATED,
    LAST_UPD
FROM SIEBEL.S_SRV_REQ@SIEBEL_19C_LINK
WHERE SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')
  AND SP_PRDINT_ID IS NOT NULL
  AND DESC_TEXT IS NOT NULL;

COMMIT;

-- Copy related Work Orders
INSERT /*+ APPEND */ INTO STG_S_ORDER
SELECT 
    o.ROW_ID,
    o.SR_ID,
    o.ORDER_NUM,
    o.DESC_TEXT,
    o.CREATED,
    o.LAST_UPD
FROM SIEBEL.S_ORDER@SIEBEL_19C_LINK o
WHERE EXISTS (
    SELECT 1 FROM STG_S_SRV_REQ sr 
    WHERE sr.ROW_ID = o.SR_ID
);

COMMIT;

-- Copy related Activities
INSERT /*+ APPEND */ INTO STG_S_EVT_ACT
SELECT 
    a.ROW_ID,
    a.SRA_SR_ID,
    a.ORDER_ID,
    a.COMMENTS,
    a.CREATED,
    a.LAST_UPD
FROM SIEBEL.S_EVT_ACT@SIEBEL_19C_LINK a
WHERE (
    EXISTS (SELECT 1 FROM STG_S_SRV_REQ sr WHERE sr.ROW_ID = a.SRA_SR_ID)
    OR
    EXISTS (SELECT 1 FROM STG_S_ORDER o WHERE o.ROW_ID = a.ORDER_ID)
);

COMMIT;

-- Gather statistics after load
BEGIN
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STG_S_SRV_REQ');
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STG_S_ORDER');
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STG_S_EVT_ACT');
    DBMS_STATS.GATHER_TABLE_STATS('SEMANTIC_SEARCH', 'STG_S_PROD_INT');
END;
/

-- Verify row counts
SELECT 'STG_S_SRV_REQ' AS TABLE_NAME, COUNT(*) AS ROW_COUNT FROM STG_S_SRV_REQ
UNION ALL
SELECT 'STG_S_ORDER', COUNT(*) FROM STG_S_ORDER
UNION ALL
SELECT 'STG_S_EVT_ACT', COUNT(*) FROM STG_S_EVT_ACT
UNION ALL
SELECT 'STG_S_PROD_INT', COUNT(*) FROM STG_S_PROD_INT;
```

**Performance Tips:**
- Use `/*+ APPEND */` hint for direct-path inserts
- Run during off-peak hours
- Monitor tablespace usage
- Consider parallel processing for very large tables:
  ```sql
  ALTER SESSION ENABLE PARALLEL DML;
  INSERT /*+ APPEND PARALLEL(4) */ INTO STG_S_SRV_REQ ...
  ```

---

### Step 4: Create Data Aggregation View

**Executor:** Database Administrator on Oracle 23ai  
**Duration:** 30 minutes

#### 4.1. Create Catalog Hierarchy View
```sql
-- Create view to build full catalog paths
CREATE OR REPLACE VIEW VW_CATALOG_HIERARCHY AS
SELECT
    ROW_ID,
    SYS_CONNECT_BY_PATH(NAME, ' > ') AS CATALOG_PATH,
    LEVEL AS HIERARCHY_LEVEL
FROM
    STG_S_PROD_INT
WHERE
    PROD_CD = 'Product' -- Filter for records of type 'Product'
    AND CONNECT_BY_ISLEAF = 1
START WITH
    PAR_PROD_INT_ID IS NULL
CONNECT BY
    PRIOR ROW_ID = PAR_PROD_INT_ID;

-- Test the view
SELECT * FROM VW_CATALOG_HIERARCHY
WHERE ROWNUM <= 10;
```

#### 4.2. Create Aggregated Narratives View
```sql
-- Create view that aggregates all text for each SR
CREATE OR REPLACE VIEW VW_AGGREGATED_NARRATIVES AS
WITH Aggregated_Events AS (
    SELECT
        SR_ID,
        -- Use XMLAGG for proper CLOB handling
        RTRIM(
            XMLAGG(
                XMLELEMENT(E, EVENT_TEXT || CHR(10) || '--------------------' || CHR(10))
                ORDER BY EVENT_TIMESTAMP
            ).EXTRACT('//text()').GetClobVal(),
            CHR(10) || '--------------------' || CHR(10)
        ) AS AGGREGATED_NARRATIVE
    FROM (
        -- Service Request initial description
        SELECT
            req.ROW_ID AS SR_ID,
            req.CREATED AS EVENT_TIMESTAMP,
            req.SR_TITLE || ' ' || req.DESC_TEXT AS EVENT_TEXT
        FROM
            STG_S_SRV_REQ req

        UNION ALL

        -- Work Order descriptions
        SELECT
            wo.SR_ID,
            wo.CREATED AS EVENT_TIMESTAMP,
            'Work Order: ' || wo.DESC_TEXT AS EVENT_TEXT
        FROM
            STG_S_ORDER wo
        WHERE
            wo.SR_ID IS NOT NULL
            AND wo.DESC_TEXT IS NOT NULL

        UNION ALL

        -- Activity comments (emails, notes, etc.)
        SELECT
            COALESCE(act.SRA_SR_ID, (SELECT SR_ID FROM STG_S_ORDER WHERE ROW_ID = act.ORDER_ID)) AS SR_ID,
            act.CREATED AS EVENT_TIMESTAMP,
            'Activity: ' || act.COMMENTS AS EVENT_TEXT
        FROM
            STG_S_EVT_ACT act
        WHERE
            COALESCE(act.SRA_SR_ID, act.ORDER_ID) IS NOT NULL
            AND act.COMMENTS IS NOT NULL
    ) All_Events
    WHERE SR_ID IS NOT NULL
    GROUP BY SR_ID
)
-- Final view combining everything
SELECT
    sr.ROW_ID AS SR_ID,
    sr.SR_NUM,
    sr.SP_PRDINT_ID AS FINAL_CATALOG_ITEM_ID,
    ch.CATALOG_PATH,
    -- Prepend the catalog path to the narrative for full context
    ch.CATALOG_PATH || CHR(10) || '--------------------' || CHR(10) || 
    ae.AGGREGATED_NARRATIVE AS FULL_NARRATIVE_WITH_CONTEXT
FROM
    STG_S_SRV_REQ sr
JOIN
    Aggregated_Events ae ON sr.ROW_ID = ae.SR_ID
JOIN
    VW_CATALOG_HIERARCHY ch ON sr.SP_PRDINT_ID = ch.ROW_ID
WHERE
    sr.SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')
    AND sr.SP_PRDINT_ID IS NOT NULL;

-- Test the view
SELECT 
    SR_ID, 
    SR_NUM,
    CATALOG_PATH,
    SUBSTR(FULL_NARRATIVE_WITH_CONTEXT, 1, 200) AS NARRATIVE_PREVIEW
FROM VW_AGGREGATED_NARRATIVES
WHERE ROWNUM <= 5;
```

---

### Step 5: Create Staging Table for Vector Generation

**Executor:** Database Administrator on Oracle 23ai  
**Duration:** 20-40 minutes (depending on data volume)

#### 5.1. Create the Narrative Staging Table
```sql
-- Create table to hold the prepared narratives before vector generation
CREATE TABLE SIEBEL_NARRATIVES_STAGING (
    SR_ID                   VARCHAR2(15) NOT NULL PRIMARY KEY,
    SR_NUM                  VARCHAR2(50),
    CATALOG_ITEM_ID         VARCHAR2(15) NOT NULL,
    CATALOG_PATH            VARCHAR2(4000),
    FULL_NARRATIVE          CLOB,
    PROCESSED_FLAG          CHAR(1) DEFAULT 'N',
    CREATED_DATE            DATE DEFAULT SYSDATE,
    PROCESSED_DATE          DATE
);

-- Create index for efficient processing
CREATE INDEX IDX_NARRATIVES_PROC_FLAG ON SIEBEL_NARRATIVES_STAGING(PROCESSED_FLAG);

COMMENT ON COLUMN SIEBEL_NARRATIVES_STAGING.SR_ID IS 'Primary key from S_SRV_REQ.ROW_ID';
COMMENT ON COLUMN SIEBEL_NARRATIVES_STAGING.CATALOG_ITEM_ID IS 'Foreign key to S_PROD_INT for the final catalog item';
COMMENT ON COLUMN SIEBEL_NARRATIVES_STAGING.CATALOG_PATH IS 'Full human-readable path of the catalog item';
COMMENT ON COLUMN SIEBEL_NARRATIVES_STAGING.FULL_NARRATIVE IS 'The complete aggregated text including catalog path';
COMMENT ON COLUMN SIEBEL_NARRATIVES_STAGING.PROCESSED_FLAG IS 'Y = vector generated, N = pending processing';
```

#### 5.2. Populate the Staging Table
```sql
-- Insert data from the aggregated view
INSERT INTO SIEBEL_NARRATIVES_STAGING (
    SR_ID,
    SR_NUM,
    CATALOG_ITEM_ID,
    CATALOG_PATH,
    FULL_NARRATIVE
)
SELECT
    SR_ID,
    SR_NUM,
    FINAL_CATALOG_ITEM_ID,
    CATALOG_PATH,
    FULL_NARRATIVE_WITH_CONTEXT
FROM
    VW_AGGREGATED_NARRATIVES;

COMMIT;

-- Verify the load
SELECT 
    COUNT(*) AS TOTAL_RECORDS,
    COUNT(CASE WHEN PROCESSED_FLAG = 'Y' THEN 1 END) AS PROCESSED,
    COUNT(CASE WHEN PROCESSED_FLAG = 'N' THEN 1 END) AS PENDING
FROM SIEBEL_NARRATIVES_STAGING;

-- Sample the data
SELECT 
    SR_ID,
    CATALOG_PATH,
    SUBSTR(FULL_NARRATIVE, 1, 150) AS NARRATIVE_SAMPLE
FROM SIEBEL_NARRATIVES_STAGING
WHERE ROWNUM <= 10;
```

---

### Step 6: Set Up Incremental Refresh Process

**Executor:** Database Administrator on Oracle 23ai  
**Duration:** 15 minutes

#### 6.1. Create Incremental Refresh Procedure
```sql
CREATE OR REPLACE PROCEDURE REFRESH_STAGING_DATA (
    p_days_lookback IN NUMBER DEFAULT 1
) AS
    v_srv_req_count NUMBER;
    v_narratives_count NUMBER;
BEGIN
    DBMS_OUTPUT.PUT_LINE('Starting staging data refresh...');
    DBMS_OUTPUT.PUT_LINE('Lookback period: ' || p_days_lookback || ' days');
    
    -- Step 1: Refresh staging tables using MERGE
    MERGE INTO STG_S_SRV_REQ t
    USING (
        SELECT *
        FROM SIEBEL.S_SRV_REQ@SIEBEL_19C_LINK
        WHERE LAST_UPD >= SYSDATE - p_days_lookback
          AND SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')
          AND SP_PRDINT_ID IS NOT NULL
    ) s ON (t.ROW_ID = s.ROW_ID)
    WHEN MATCHED THEN
        UPDATE SET 
            t.SR_TITLE = s.SR_TITLE,
            t.DESC_TEXT = s.DESC_TEXT,
            t.SR_STAT_ID = s.SR_STAT_ID,
            t.SP_PRDINT_ID = s.SP_PRDINT_ID,
            t.LAST_UPD = s.LAST_UPD
    WHEN NOT MATCHED THEN
        INSERT (ROW_ID, SR_NUM, SR_TITLE, DESC_TEXT, SR_STAT_ID, SP_PRDINT_ID, CREATED, LAST_UPD)
        VALUES (s.ROW_ID, s.SR_NUM, s.SR_TITLE, s.DESC_TEXT, s.SR_STAT_ID, s.SP_PRDINT_ID, s.CREATED, s.LAST_UPD);
    
    v_srv_req_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Service Requests processed: ' || v_srv_req_count);
    COMMIT;
    
    -- Step 2: Refresh narratives using MERGE (more efficient than DELETE/INSERT)
    MERGE INTO SIEBEL_NARRATIVES_STAGING t
    USING (
        SELECT
            SR_ID,
            SR_NUM,
            FINAL_CATALOG_ITEM_ID AS CATALOG_ITEM_ID,
            CATALOG_PATH,
            FULL_NARRATIVE_WITH_CONTEXT AS FULL_NARRATIVE
        FROM
            VW_AGGREGATED_NARRATIVES
        WHERE
            SR_ID IN (
                SELECT ROW_ID 
                FROM SIEBEL.S_SRV_REQ@SIEBEL_19C_LINK
                WHERE LAST_UPD >= SYSDATE - p_days_lookback
            )
    ) s ON (t.SR_ID = s.SR_ID)
    WHEN MATCHED THEN
        UPDATE SET
            t.SR_NUM = s.SR_NUM,
            t.CATALOG_ITEM_ID = s.CATALOG_ITEM_ID,
            t.CATALOG_PATH = s.CATALOG_PATH,
            t.FULL_NARRATIVE = s.FULL_NARRATIVE,
            t.PROCESSED_FLAG = 'N',  -- Reset to reprocess
            t.PROCESSED_DATE = NULL,
            t.ERROR_MESSAGE = NULL,
            t.UPDATED_DATE = SYSDATE
    WHEN NOT MATCHED THEN
        INSERT (
            SR_ID, 
            SR_NUM, 
            CATALOG_ITEM_ID, 
            CATALOG_PATH, 
            FULL_NARRATIVE,
            PROCESSED_FLAG,
            CREATED_DATE,
            UPDATED_DATE
        )
        VALUES (
            s.SR_ID,
            s.SR_NUM,
            s.CATALOG_ITEM_ID,
            s.CATALOG_PATH,
            s.FULL_NARRATIVE,
            'N',
            SYSDATE,
            SYSDATE
        );
    
    v_narratives_count := SQL%ROWCOUNT;
    DBMS_OUTPUT.PUT_LINE('Narratives processed: ' || v_narratives_count);
    COMMIT;
    
    -- Log refresh activity
    INSERT INTO STAGING_REFRESH_LOG (
        REFRESH_DATE,
        DAYS_LOOKBACK,
        SRV_REQ_PROCESSED,
        NARRATIVES_PROCESSED,
        STATUS
    ) VALUES (
        SYSDATE,
        p_days_lookback,
        v_srv_req_count,
        v_narratives_count,
        'SUCCESS'
    );
    COMMIT;
    
    DBMS_OUTPUT.PUT_LINE('Staging refresh completed successfully');
    
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        
        -- Log failure
        BEGIN
            INSERT INTO STAGING_REFRESH_LOG (
                REFRESH_DATE,
                DAYS_LOOKBACK,
                STATUS,
                ERROR_MESSAGE
            ) VALUES (
                SYSDATE,
                p_days_lookback,
                'FAILED',
                SUBSTR(SQLERRM, 1, 500)
            );
            COMMIT;
        EXCEPTION
            WHEN OTHERS THEN
                NULL; -- Ignore logging errors
        END;
        
        DBMS_OUTPUT.PUT_LINE('ERROR: ' || SQLERRM);
        RAISE;
END REFRESH_STAGING_DATA;
/
```

#### 6.2. Schedule the Refresh Job
```sql
-- Create a scheduled job to run nightly at 2 AM
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
        job_name        => 'DAILY_STAGING_REFRESH',
        job_type        => 'STORED_PROCEDURE',
        job_action      => 'SEMANTIC_SEARCH.REFRESH_STAGING_DATA',
        number_of_arguments => 1,
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=0; BYSECOND=0',
        enabled         => TRUE,
        comments        => 'Daily refresh of staging data from Siebel 19c'
    );
    
    -- Set the parameter for lookback days
    DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE(
        job_name          => 'DAILY_STAGING_REFRESH',
        argument_position => 1,
        argument_value    => '1'
    );
END;
/

-- Verify job creation
SELECT job_name, enabled, state, next_run_date
FROM USER_SCHEDULER_JOBS
WHERE job_name = 'DAILY_STAGING_REFRESH';
```

---

## 4. Verification and Validation

### 4.1. Data Quality Checks
```sql
-- Check for NULL narratives
SELECT COUNT(*) AS NULL_NARRATIVES
FROM SIEBEL_NARRATIVES_STAGING
WHERE FULL_NARRATIVE IS NULL;
-- Expected: 0

-- Check for very short narratives (potential data quality issues)
SELECT COUNT(*) AS SHORT_NARRATIVES
FROM SIEBEL_NARRATIVES_STAGING
WHERE LENGTH(FULL_NARRATIVE) < 50;
-- Review these records manually

-- Check catalog path distribution
SELECT 
    SUBSTR(CATALOG_PATH, 1, 50) AS CATALOG_PREFIX,
    COUNT(*) AS COUNT
FROM SIEBEL_NARRATIVES_STAGING
GROUP BY SUBSTR(CATALOG_PATH, 1, 50)
ORDER BY COUNT DESC
FETCH FIRST 20 ROWS ONLY;

-- Check average narrative length
SELECT 
    ROUND(AVG(LENGTH(FULL_NARRATIVE))) AS AVG_LENGTH,
    MIN(LENGTH(FULL_NARRATIVE)) AS MIN_LENGTH,
    MAX(LENGTH(FULL_NARRATIVE)) AS MAX_LENGTH
FROM SIEBEL_NARRATIVES_STAGING;
```

### 4.2. Sample Data Review
```sql
-- Review a few complete records
SELECT 
    SR_ID,
    SR_NUM,
    CATALOG_PATH,
    FULL_NARRATIVE
FROM SIEBEL_NARRATIVES_STAGING
WHERE ROWNUM <= 3;
```

---

## 5. Troubleshooting Guide

### 5.1. Common Issues

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Database link test fails | Network/firewall issue | Check VNet peering, NSG rules, verify Oracle 19c listener is running |
| ORA-12154 TNS error | Connection descriptor syntax error | Verify inline connection string format in database link definition |
| Very slow data copy | Large data volume, network latency | Use APPEND hint, consider processing during off-peak hours |
| XMLAGG issues | Very long narratives | Increase PGA memory or process in batches |
| View returns no data | Staging tables empty | Verify Step 3 completed successfully, check database link connectivity |
| Job doesn't run | Scheduler not enabled | Check `USER_SCHEDULER_JOBS.STATE`, verify job is enabled in ADB |

### 5.2. Performance Monitoring
```sql
-- Monitor long-running queries
SELECT 
    sid, 
    serial#, 
    sql_id, 
    status, 
    last_call_et/60 AS minutes_running
FROM v$session
WHERE username = 'SEMANTIC_SEARCH'
  AND status = 'ACTIVE';

-- Check tablespace usage
SELECT 
    tablespace_name,
    ROUND(SUM(bytes)/1024/1024/1024, 2) AS GB_USED
FROM dba_segments
WHERE owner = 'SEMANTIC_SEARCH'
GROUP BY tablespace_name;
```

---

## 6. Next Steps

Once this TDD is complete, proceed to:
- **TDD 2**: Generate vector embeddings for the narratives in `SIEBEL_NARRATIVES_STAGING`
- The `PROCESSED_FLAG` will be updated to 'Y' as vectors are generated

---

## 7. Maintenance Procedures

### 7.1. Manual Refresh (if needed)
```sql
-- Run incremental refresh manually for last 7 days
EXEC REFRESH_STAGING_DATA(7);
```

### 7.2. Full Reload (disaster recovery)
```sql
-- Truncate all staging tables
TRUNCATE TABLE SIEBEL_NARRATIVES_STAGING;
TRUNCATE TABLE STG_S_SRV_REQ;
TRUNCATE TABLE STG_S_ORDER;
TRUNCATE TABLE STG_S_EVT_ACT;
TRUNCATE TABLE STG_S_PROD_INT;

-- Re-run Step 3 and Step 5 from this document
```



