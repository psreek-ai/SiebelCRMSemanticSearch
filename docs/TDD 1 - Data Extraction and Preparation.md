# Technical Design Document 1: Data Extraction and Preparation

**Version:** 2.0
**Date:** 2025-10-16
**Owner:** Database Team

## 1. Objective
This document provides the technical specification for extracting and preparing historical service request data from the Oracle 12c Siebel database. The output of this process will be a structured CSV file, ready for the vector indexing pipeline.

## 2. Source Data
The data will be extracted from the `SIEBEL` schema in the production Oracle 12c database.

### 2.1. Primary Tables
* `SIEBEL.S_SRV_REQ`: Master table for Service Requests.
* `SIEBEL.S_ORDER`: Table for Work Orders linked to Service Requests.
* `SIEBEL.S_EVT_ACT`: Table for Activities, including email communications.
* `SIEBEL.S_PROD_INT`: Table for the service catalog product hierarchy.

## 3. Data Aggregation Logic
The core of the extraction is a SQL query that consolidates all relevant text for each resolved service request into a single, chronologically ordered narrative. This narrative, along with the catalog path, forms the "document" that will be vectorized.

### 3.1. Final Extraction Script (Oracle 12c Compatible)
The following script should be used for the data export. It uses `XMLAGG` for compatibility with Oracle 12c and correctly builds the catalog hierarchy.

```sql
WITH Catalog_Hierarchy AS (
    -- This CTE traverses the S_PROD_INT table to build a full path for each item.
    SELECT
        ROW_ID,
        -- The path is built using the NAME column for readability.
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
)
SELECT
    SR_ID,
    FINAL_CATALOG_ITEM_ID,
    CATALOG_PATH,
    -- Prepend the catalog path to the narrative for full context.
    CATALOG_PATH || CHR(10) || '--------------------' || CHR(10) || AGGREGATED_NARRATIVE AS FULL_NARRATIVE_WITH_CONTEXT
FROM (
    SELECT
        SR_ID,
        FINAL_CATALOG_ITEM_ID,
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
            req.SR_TITLE || ' ' || req.DESC_TEXT AS EVENT_TEXT,
            req.SP_PRDINT_ID AS CATALOG_ITEM_ID
        FROM
            SIEBEL.S_SRV_REQ req
        WHERE
            req.SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')

        UNION ALL

        SELECT
            wo.SR_ID,
            wo.CREATED AS EVENT_TIMESTAMP,
            wo.DESC_TEXT AS EVENT_TEXT,
            NULL AS CATALOG_ITEM_ID
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
            act.COMMENTS AS EVENT_TEXT,
            NULL AS CATALOG_ITEM_ID
        FROM
            SIEBEL.S_EVT_ACT act
        INNER JOIN SIEBEL.S_SRV_REQ req ON COALESCE(act.SRA_SR_ID, (SELECT SR_ID FROM SIEBEL.S_ORDER WHERE ROW_ID = act.ORDER_ID)) = req.ROW_ID
        WHERE
            req.SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')
            AND COALESCE(act.SRA_SR_ID, act.ORDER_ID) IS NOT NULL
    ) All_Events
    LEFT JOIN (
        SELECT ROW_ID, SP_PRDINT_ID AS FINAL_CATALOG_ITEM_ID FROM SIEBEL.S_SRV_REQ
    ) sr_catalog ON All_Events.SR_ID = sr_catalog.ROW_ID
    WHERE
        SR_ID IS NOT NULL
        AND FINAL_CATALOG_ITEM_ID IS NOT NULL
    GROUP BY
        SR_ID, FINAL_CATALOG_ITEM_ID
) Final_Narrative
-- Join the final narrative with the catalog hierarchy path.
JOIN Catalog_Hierarchy ON Final_Narrative.FINAL_CATALOG_ITEM_ID = Catalog_Hierarchy.ROW_ID;
````

## 4\. Output Format

The output must be a CSV file with a header row and the following columns:

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `SR_ID` | VARCHAR2(15) | The unique identifier of the Service Request. |
| `FINAL_CATALOG_ITEM_ID` | VARCHAR2(15) | The `ROW_ID` of the final catalog item from `S_PROD_INT`. |
| `CATALOG_PATH` | VARCHAR2(4000) | The full human-readable path of the catalog item. |
| `FULL_NARRATIVE_WITH_CONTEXT`| CLOB | The complete aggregated text from all sources, including the catalog path. |

**CSV Format Details:**

  * **Delimiter:** Comma (`,`)
  * **Enclosure:** Double quotes (`"`) for all fields.
  * **Encoding:** UTF-8

## 5\. Execution Strategy

### 5.1. Initial Data Load

The initial export of millions of records must be performed in batches to avoid overwhelming the database and to create manageable file sizes.

**Recommendation:** Add a date filter to the final `WHERE` clause of the main query to process data by year.
*Example for the year 2024:*

```sql
...
JOIN Catalog_Hierarchy ON Final_Narrative.FINAL_CATALOG_ITEM_ID = Catalog_Hierarchy.ROW_ID
WHERE SR_ID IN (SELECT ROW_ID FROM SIEBEL.S_SRV_REQ WHERE CREATED >= TO_DATE('2024-01-01', 'YYYY-MM-DD') AND CREATED < TO_DATE('2025-01-01', 'YYYY-MM-DD'));
```

### 5.2. Incremental Updates

A nightly job should be scheduled to extract only the service requests that were closed or resolved in the last 24 hours.

#### 5.2.1. Incremental Update Script

This script is designed to be run daily. It efficiently finds SRs that were updated to a closed status within the last day and extracts their full history.

```sql
-- The Catalog_Hierarchy CTE remains the same as in the full extraction script.
WITH Catalog_Hierarchy AS (...)
SELECT ...
FROM ...
JOIN Catalog_Hierarchy ON Final_Narrative.FINAL_CATALOG_ITEM_ID = Catalog_Hierarchy.ROW_ID
-- The WHERE clause is modified to only select recently updated SRs.
WHERE SR_ID IN (
    SELECT ROW_ID FROM SIEBEL.S_SRV_REQ
    WHERE LAST_UPD > SYSDATE - 1
    AND SR_STAT_ID IN ('Closed', 'Resolved', 'Completed')
);
```

## 6\. Data Cleansing and Pre-processing

Before generating embeddings, the `FULL_NARRATIVE_WITH_CONTEXT` text should be cleaned to improve the quality of the vectors. This cleansing step will be part of the Python-based indexing pipeline.

  * **Remove Boilerplate:** Remove common, low-value text like "Dear Sir/Madam," "Thank you," and email signatures.
  * **HTML Stripping:** If emails contain HTML, strip all tags to retain only the plain text content.
  * **Normalize Whitespace:** Replace multiple spaces, tabs, and newlines with a single space.
  * **PII Masking (Optional but Recommended):** For compliance, consider implementing a step to find and mask Personally Identifiable Information (PII) like names, phone numbers, and email addresses, replacing them with tokens like `[NAME]` or `[EMAIL]`.

## 7\. Security and Performance

  * The extraction should be run during off-peak hours to minimize impact on the production Siebel application.
  * The SQL query should be executed with a read-only database user account that has the necessary privileges on the `SIEBEL` schema.
  * It is recommended to analyze the query's execution plan to ensure it is using appropriate indexes, particularly on `CREATED` and `LAST_UPD` columns.


