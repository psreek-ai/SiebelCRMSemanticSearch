# Email Activity Filtering Recommendations

## Overview
This document provides recommendations for filtering out redundant and system-generated email activities from Service Requests (SRs) and Work Orders (WOs) to focus only on meaningful communications for semantic search.

## Analysis of Email Activity Types

Based on the provided examples from `SR_EMAIL_COMMUNICATIONS.HTML` and `WORK_ORDER_EMAIL_COMMUNICATIONS.HTML`, the following patterns have been identified:

### Redundant Activities to EXCLUDE

#### 1. System Notification Activities
**Type:** `Notification`
**Examples:**
- "An e-mail has been sent to the Assigned To person"
- "This WO has been reassigned and an e-mail has been sent to..."

**Rationale:** These are purely system-generated status notifications with no substantive content.

#### 2. Automated Work Order Closures
**Type:** `WO Closure`
**Description:** "Work Order Closure Notification"
**Example Comments:** 
- "Work Order WO-RFS-1-12041310345 is closed and an email is sent To WO Owner"

**Rationale:** Closure notifications are procedural and don't add semantic value.

#### 3. Generic "Other" Activities with Minimal Content
**Type:** `Other`
**Examples:**
- Comments: "Resolved"
- Comments: "The issue is resolved."

**Rationale:** These are brief status updates without meaningful context.

#### 4. Standard System-Generated Outbound Emails
**Type:** `Email - Outbound`

**Specific Descriptions to Exclude:**
- "Service Request Submission Notification - SR ID: %"
- "Service Request has been assigned to you. SR ID: %"
- "Service Request Resolution Notification. SR ID: %"
- "Your Service Request has been updated - SR ID: %"

**Rationale:** These are templated system notifications sent for every SR. The boilerplate content doesn't add unique information for semantic matching.

### Meaningful Activities to INCLUDE

#### 1. Inbound Emails (Customer/Staff Communications)
**Type:** `Email - Inbound`
**Rationale:** These contain actual user communications, requirements, questions, and discussions that are valuable for semantic search.

#### 2. Substantive Outbound Emails
**Type:** `Email - Outbound`
**Examples:**
- "Follow-up regarding SR ID: %"
- Emails with specific work details, solutions, or technical content

**Rationale:** Follow-ups and detailed communications from support staff contain valuable context and solutions.

## SQL Implementation

### Updated Staging Table Structure

Add the `TYPE` and `DESC_TEXT` fields to the `STG_S_EVT_ACT` table:

```sql
-- Activities staging table (UPDATED)
CREATE TABLE STG_S_EVT_ACT AS
SELECT 
    ROW_ID,
    SRA_SR_ID,
    ORDER_ID,
    TYPE,           -- Activity type field
    DESC_TEXT,      -- Description field
    COMMENTS,
    CREATED,
    LAST_UPD
FROM SIEBEL.S_EVT_ACT@SIEBEL_19C_LINK
WHERE 1=0;

-- Add primary key and indexes
ALTER TABLE STG_S_EVT_ACT ADD CONSTRAINT PK_STG_EVT_ACT PRIMARY KEY (ROW_ID);
CREATE INDEX IDX_STG_EVT_SR_ID ON STG_S_EVT_ACT(SRA_SR_ID);
CREATE INDEX IDX_STG_EVT_ORDER_ID ON STG_S_EVT_ACT(ORDER_ID);
CREATE INDEX IDX_STG_EVT_TYPE ON STG_S_EVT_ACT(TYPE);
```

### Updated Data Extraction Query

```sql
-- Copy related Activities (UPDATED with filters)
INSERT /*+ APPEND */ INTO STG_S_EVT_ACT
SELECT 
    a.ROW_ID,
    a.SRA_SR_ID,
    a.ORDER_ID,
    a.TYPE,
    a.DESC_TEXT,
    a.COMMENTS,
    a.CREATED,
    a.LAST_UPD
FROM SIEBEL.S_EVT_ACT@SIEBEL_19C_LINK a
WHERE (
    a.SRA_SR_ID IN (SELECT ROW_ID FROM STG_S_SRV_REQ)
    OR a.ORDER_ID IN (SELECT ROW_ID FROM STG_S_ORDER)
)
AND a.COMMENTS IS NOT NULL
-- Filter out redundant activity types
AND a.TYPE NOT IN (
    'Notification',
    'WO Closure'
)
-- Filter out generic "Other" activities with minimal content
AND NOT (
    a.TYPE = 'Other' 
    AND (
        LENGTH(TRIM(a.COMMENTS)) < 50  -- Very short comments
        OR UPPER(a.COMMENTS) LIKE '%RESOLVED%'
        OR UPPER(a.COMMENTS) LIKE '%CLOSED%'
    )
)
-- Filter out standard system-generated outbound emails
AND NOT (
    a.TYPE = 'Email - Outbound'
    AND (
        a.DESC_TEXT LIKE 'Service Request Submission Notification%'
        OR a.DESC_TEXT LIKE 'Service Request has been assigned to you%'
        OR a.DESC_TEXT LIKE 'Service Request Resolution Notification%'
        OR a.DESC_TEXT LIKE 'Your Service Request has been updated%'
    )
);

COMMIT;
```

### Updated Aggregated Events Query

```sql
-- Activity comments (emails, notes, etc.) - UPDATED
SELECT
    COALESCE(act.SRA_SR_ID, (SELECT SR_ID FROM STG_S_ORDER WHERE ROW_ID = act.ORDER_ID)) AS SR_ID,
    act.CREATED AS EVENT_TIMESTAMP,
    -- Include activity type for better context
    CASE 
        WHEN act.TYPE = 'Email - Inbound' THEN 'Inbound Email: '
        WHEN act.TYPE = 'Email - Outbound' THEN 'Outbound Email: '
        ELSE 'Activity: '
    END || act.COMMENTS AS EVENT_TEXT
FROM
    STG_S_EVT_ACT act
WHERE
    COALESCE(act.SRA_SR_ID, act.ORDER_ID) IS NOT NULL
    AND act.COMMENTS IS NOT NULL
    -- Additional runtime filter to ensure quality
    AND LENGTH(TRIM(act.COMMENTS)) >= 50  -- Minimum meaningful content
```

## Expected Impact

### Before Filtering
- **Total Activities:** ~13 per SR/WO (from examples)
- **Meaningful Activities:** ~3-4 per SR/WO

### After Filtering
- **Reduction:** ~65-70% of redundant activities excluded
- **Quality Improvement:** Focus on actual communications and substantive content
- **Vector Database Size:** Significantly reduced, leading to faster searches
- **Search Relevance:** Improved by eliminating noise from system notifications

## Testing Recommendations

### 1. Verify Filtering Logic
```sql
-- Test query to see what's being filtered out
SELECT 
    TYPE,
    DESC_TEXT,
    SUBSTR(COMMENTS, 1, 100) AS COMMENT_PREVIEW,
    COUNT(*) AS ACTIVITY_COUNT
FROM STG_S_EVT_ACT
GROUP BY TYPE, DESC_TEXT
ORDER BY COUNT(*) DESC;
```

### 2. Sample Check
```sql
-- Review a specific SR's activities before aggregation
SELECT 
    SRA_SR_ID,
    TYPE,
    DESC_TEXT,
    SUBSTR(COMMENTS, 1, 200) AS COMMENT_PREVIEW,
    CREATED
FROM STG_S_EVT_ACT
WHERE SRA_SR_ID = '1-5J532IP'  -- Replace with actual SR ID
ORDER BY CREATED;
```

### 3. Validate Exclusions
```sql
-- Check what would be excluded by the filters
SELECT 
    TYPE,
    DESC_TEXT,
    COUNT(*) AS EXCLUDED_COUNT
FROM SIEBEL.S_EVT_ACT@SIEBEL_19C_LINK
WHERE 
    (TYPE IN ('Notification', 'WO Closure')
    OR (TYPE = 'Other' AND LENGTH(TRIM(COMMENTS)) < 50)
    OR (TYPE = 'Email - Outbound' AND DESC_TEXT LIKE 'Service Request%Notification%'))
GROUP BY TYPE, DESC_TEXT
ORDER BY COUNT(*) DESC;
```

## Additional Considerations

### 1. Language Detection
Consider filtering activities that don't contain meaningful text (e.g., just URLs or boilerplate signatures):

```sql
-- Optional: Exclude activities that are mostly URLs or signatures
AND NOT REGEXP_LIKE(act.COMMENTS, '^https?://', 'i')
AND NOT (
    LENGTH(REGEXP_REPLACE(act.COMMENTS, 'https?://[^\s]+', '', 'g')) < 50
)
```

### 2. Duplicate Content Detection
Some emails might appear in both SR and WO activities. Consider deduplication in the aggregation:

```sql
-- In the aggregated events CTE, use DISTINCT
SELECT DISTINCT
    SR_ID,
    EVENT_TIMESTAMP,
    EVENT_TEXT
FROM (
    -- ... your activity queries
)
```

### 3. Private Flag
If the `PRIVATE` flag exists in S_EVT_ACT, consider whether to include/exclude private activities:

```sql
-- Optional: Exclude private activities
AND (a.PRIVATE_FLG = 'N' OR a.PRIVATE_FLG IS NULL)
```

## Maintenance

### Regular Review
Periodically review the activity types and descriptions in the system:

```sql
-- Audit query to identify new system notification patterns
SELECT 
    TYPE,
    DESC_TEXT,
    COUNT(*) AS OCCURRENCE_COUNT,
    MIN(CREATED) AS FIRST_SEEN,
    MAX(CREATED) AS LAST_SEEN
FROM SIEBEL.S_EVT_ACT@SIEBEL_19C_LINK
WHERE CREATED >= SYSDATE - 30  -- Last 30 days
GROUP BY TYPE, DESC_TEXT
HAVING COUNT(*) > 100  -- Frequently occurring patterns
ORDER BY COUNT(*) DESC;
```

## Conclusion

By implementing these filters, the semantic search system will:
1. **Focus on meaningful content** - Only customer communications and substantive support interactions
2. **Reduce noise** - Eliminate repetitive system notifications
3. **Improve search quality** - Better semantic matching with relevant content
4. **Optimize performance** - Smaller vector database with higher-quality data

---

**Document Version:** 1.0  
**Date:** October 17, 2025  
**Related TDD:** TDD 1 - Data Extraction and Preparation
