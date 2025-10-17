# Technical Design Document 1: Data Extraction and Preparation

**Version:** 2.1
**Date:** 2025-10-16
**Owner:** Database Team

## 1. Objective
This document provides the technical specification for extracting and preparing historical service request data from the Oracle 12c Siebel database. The output of this process will be a structured dataset, ready for the vector indexing pipeline.

## 2. Source Data
The data will be extracted from the `SIEBEL` schema in the production Oracle 12c database.

### 2.1. Primary Tables
* `SIEBEL.S_SRV_REQ`: Master table for Service Requests.
* `SIEBEL.S_ORDER`: Table for Work Orders linked to Service Requests.
* `SIEBEL.S_EVT_ACT`: Table for Activities, including email communications.
* `SIEBEL.S_PROD_INT`: Table for the service catalog product hierarchy.

## 3. Data Aggregation Logic
The core of the extraction is a SQL query that consolidates all relevant text for each resolved service request into a single, chronologically ordered narrative. This narrative, along with the final, authoritative catalog path, forms the "document" that will be vectorized.

### 3.1. Final Extraction Script (Oracle 12c Compatible)
The following script should be used for the data export. It correctly identifies the final catalog item associated with each service request and builds the full narrative.

```sql
WITH Catalog_Hierarchy AS (
    -- This CTE traverses the S_PROD_INT table to build a full path for each item.
    SELECT
        ROW_ID,
        SYS_CONNECT_BY_PATH(NAME, ' > ') AS CATALOG_PATH
    FROM
        SIEBEL.S_PROD_INT
    WHERE
        PROD_CD = 'Product' -- Filter for records of type 'Product'
        AND CONNECT_BY_ISLEAF = 1
    START WITH
        PAR_PROD_INT_ID IS NULL
    CONNECT BY
        PRIOR ROW_ID = PAR_PROD_INT_ID
),
Aggregated_Events AS (
    -- This CTE gathers all text fragments and associates them with a Service Request ID.
    SELECT
        SR_ID,
        -- Use RTRIM to remove the final separator from the end of the aggregated CLOB
        RTRIM(
            XMLAGG(
                XMLELEMENT(E, EVENT_TEXT || CHR(10) || '--------------------' || CHR(10))
                ORDER BY EVENT_TIMESTAMP
            ).EXTRACT('//text()').GetClobVal(),
            CHR(10) || '--------------------' || CHR(10)
        ) AS AGGREGATED_NARRATIVE
    FROM (
        -- Inner query gathers all text fragments
        SELECT
            req.ROW_ID AS SR_ID,
            req.CREATED AS EVENT_TIMESTAMP,
            req.SR_TITLE || ' ' || req.DESC_TEXT AS EVENT_TEXT
        FROM
            SIEBEL.S_SRV_REQ req
        WHERE
            req.SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')

        UNION ALL

        SELECT
            wo.SR_ID,
            wo.CREATED AS EVENT_TIMESTAMP,
            wo.DESC_TEXT AS EVENT_TEXT
        FROM
            SIEBEL.S_ORDER wo
        INNER JOIN SIEBEL.S_SRV_REQ req ON wo.SR_ID = req.ROW_ID
        WHERE
            req.SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')
            AND wo.SR_ID IS NOT NULL

        UNION ALL

        SELECT
            COALESCE(act.SRA_SR_ID, (SELECT SR_ID FROM SIEBEL.S_ORDER WHERE ROW_ID = act.ORDER_ID)) AS SR_ID,
            act.CREATED AS EVENT_TIMESTAMP,
            act.COMMENTS AS EVENT_TEXT
        FROM
            SIEBEL.S_EVT_ACT act
        INNER JOIN SIEBEL.S_SRV_REQ req ON COALESCE(act.SRA_SR_ID, (SELECT SR_ID FROM SIEBEL.S_ORDER WHERE ROW_ID = act.ORDER_ID)) = req.ROW_ID
        WHERE
            req.SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')
            AND COALESCE(act.SRA_SR_ID, act.ORDER_ID) IS NOT NULL
    ) All_Events
    WHERE SR_ID IS NOT NULL
    GROUP BY SR_ID
)
-- Final SELECT statement to join the aggregated narrative with the final catalog item
SELECT
    sr.ROW_ID AS SR_ID,
    sr.SP_PRDINT_ID AS FINAL_CATALOG_ITEM_ID,
    ch.CATALOG_PATH,
    -- Prepend the catalog path to the narrative for full context.
    ch.CATALOG_PATH || CHR(10) || '--------------------' || CHR(10) || ae.AGGREGATED_NARRATIVE AS FULL_NARRATIVE_WITH_CONTEXT
FROM
    SIEBEL.S_SRV_REQ sr
JOIN
    Aggregated_Events ae ON sr.ROW_ID = ae.SR_ID
JOIN
    Catalog_Hierarchy ch ON sr.SP_PRDINT_ID = ch.ROW_ID
WHERE
    sr.SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')
    AND sr.SP_PRDINT_ID IS NOT NULL;
```

## 4. Output Format
The query will produce a dataset with the following columns, which should be exported to a CSV file for the next stage:
* `SR_ID`: The unique identifier of the service request.
* `FINAL_CATALOG_ITEM_ID`: The final catalog item ID linked to the service request.
* `CATALOG_PATH`: The full, human-readable path of the catalog item.
* `FULL_NARRATIVE_WITH_CONTEXT`: The complete, aggregated text narrative, prefixed with the catalog path.

## 5. Execution and Scheduling
- **Execution Method:** The SQL script will be executed via a standard database client (like SQL*Plus or SQLcl) or a scheduling tool capable of running SQL scripts.
- **Frequency:** The script should be run as a nightly batch job to capture newly resolved service requests.
- **Output:** The output must be saved as a CSV file (e.g., `siebel_narratives_YYYYMMDD.csv`) and placed in a pre-defined location accessible to the indexing pipeline.


