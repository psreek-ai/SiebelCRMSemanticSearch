# Testing Guide: AI-Powered Semantic Search Solution

**Version:** 2.2
**Date:** 2025-10-17
**Owner:** QA Team / Business Analysts

## 1. Introduction
This document outlines the comprehensive testing strategy for the AI-Powered Semantic Search solution. Testing is organized into multiple phases, from unit testing of individual components to user acceptance testing of the complete system.

## 2. Test Environment Setup

### 2.1. Prerequisites

Before beginning testing, ensure:
- [ ] Deployment Guide has been completed successfully
- [ ] Test environment matches production configuration
- [ ] Test user accounts created with appropriate privileges
- [ ] Test data loaded into databases
- [ ] Monitoring tools configured

### 2.2. Test Tools Required

| Tool | Purpose | Installation |
|------|---------|--------------|
| Postman | API testing | Download from postman.com |
| curl | Command-line API testing | Pre-installed on most systems |
| SQL Developer | Database testing | Download from Oracle |
| Browser DevTools | UI testing | Built into Chrome/Firefox (F12) |
| JMeter | Performance testing | Download from jmeter.apache.org |

### 2.3. Test Data Preparation

```sql
-- Connect to Oracle 23ai as SEMANTIC_SEARCH
sqlplus SEMANTIC_SEARCH/<password>@<service_name>

-- Verify test data exists
SELECT COUNT(*) AS VECTOR_COUNT
FROM SIEBEL_KNOWLEDGE_VECTORS
WHERE NARRATIVE_VECTOR IS NOT NULL;
-- Expected: > 1000

-- Create test queries table
CREATE TABLE TEST_QUERIES (
    query_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    query_text VARCHAR2(1000),
    expected_category VARCHAR2(200),
    expected_top_result VARCHAR2(100),
    test_type VARCHAR2(50),
    comments VARCHAR2(1000)
);

-- Insert test queries
INSERT ALL
    INTO TEST_QUERIES (query_text, expected_category, test_type, comments) 
        VALUES ('My laptop screen is broken', 'Hardware > Laptop', 'FUNCTIONAL', 'Basic hardware issue')
    INTO TEST_QUERIES (query_text, expected_category, test_type, comments) 
        VALUES ('I need to reset my password', 'IT Support > Account', 'FUNCTIONAL', 'Common IT request')
    INTO TEST_QUERIES (query_text, expected_category, test_type, comments) 
        VALUES ('Printer is not working', 'Hardware > Printer', 'FUNCTIONAL', 'Office equipment')
    INTO TEST_QUERIES (query_text, expected_category, test_type, comments) 
        VALUES ('Need access to finance system', 'IT Support > Access', 'FUNCTIONAL', 'Access request')
    INTO TEST_QUERIES (query_text, expected_category, test_type, comments) 
        VALUES ('Computer running very slow', 'IT Support > Performance', 'FUNCTIONAL', 'Performance issue')
    INTO TEST_QUERIES (query_text, expected_category, test_type, comments) 
        VALUES ('', 'N/A', 'NEGATIVE', 'Empty query test')
    INTO TEST_QUERIES (query_text, expected_category, test_type, comments) 
        VALUES ('xyz123!@#', 'N/A', 'EDGE_CASE', 'Random characters')
    INTO TEST_QUERIES (query_text, expected_category, test_type, comments) 
        VALUES (RPAD('a', 5000, 'a'), 'N/A', 'EDGE_CASE', 'Very long query')
SELECT * FROM dual;

COMMIT;
```

---

## 3. Testing Phases

## Phase 1: Unit Testing

**Duration:** 2-3 days  
**Executor:** Development Team / QA

### Test 1.1: Database Link Connectivity

**Objective:** Verify Oracle Database 23ai on Azure VM can access Oracle 19c via VNet peering

```sql
-- Test 1.1.1: Test database link
SELECT COUNT(*) AS sr_count
FROM SIEBEL.S_SRV_REQ@SIEBEL_19C_LINK;

-- Expected: Returns a number without errors
-- Pass Criteria: Query executes successfully
-- Fail Action: Review TNS configuration, check network connectivity
```

| Test ID | Description | Expected Result | Actual Result | Status |
|---------|-------------|-----------------|---------------|--------|
| UT-DB-001 | Database link accessible | COUNT > 0 | | ⬜ |
| UT-DB-002 | Query performance < 5sec | Response time OK | | ⬜ |

### Test 1.2: Data Extraction SQL

**Objective:** Verify staging tables populated correctly

```sql
-- Test 1.2.1: Verify staging table row counts
SELECT 
    'STG_S_SRV_REQ' AS TABLE_NAME, COUNT(*) AS ROW_COUNT 
FROM STG_S_SRV_REQ
UNION ALL
SELECT 'STG_S_ORDER', COUNT(*) FROM STG_S_ORDER
UNION ALL
SELECT 'STG_S_EVT_ACT', COUNT(*) FROM STG_S_EVT_ACT
UNION ALL
SELECT 'STG_S_PROD_INT', COUNT(*) FROM STG_S_PROD_INT;

-- Pass Criteria: All tables have > 0 rows
-- Fail Action: Re-run data copy scripts from TDD 1

-- Test 1.2.2: Verify narrative aggregation
SELECT 
    SR_ID,
    LENGTH(FULL_NARRATIVE) AS NARRATIVE_LENGTH,
    CATALOG_PATH
FROM SIEBEL_NARRATIVES_STAGING
WHERE ROWNUM <= 10;

-- Pass Criteria: NARRATIVE_LENGTH > 50 characters
-- Fail Action: Check aggregation view logic
```

| Test ID | Description | Expected Result | Actual Result | Status |
|---------|-------------|-----------------|---------------|--------|
| UT-DATA-001 | Staging tables populated | All counts > 0 | | ⬜ |
| UT-DATA-002 | Narratives not NULL | No NULL values | | ⬜ |
| UT-DATA-003 | Catalog paths present | All have paths | | ⬜ |

### Test 1.3: Azure AI Foundry with OpenAI Service Embedding Generation

**Objective:** Verify embedding generation works

```sql
-- Test 1.3.1: Test Azure AI Foundry credential
DECLARE
    l_response CLOB;
    l_request_body CLOB;
    l_api_endpoint VARCHAR2(500) := 'https://<workspace-name>.<region>.api.azureml.ms/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-15-preview';
BEGIN
    l_request_body := '{"input": ["test"], "model": "text-embedding-3-small"}';
    
    l_response := DBMS_CLOUD.SEND_REQUEST(
        credential_name => 'AZURE_AI_FOUNDRY_CRED',
        uri => l_api_endpoint,
        method => 'POST',
        body => UTL_RAW.CAST_TO_RAW(l_request_body)
    );
    DBMS_OUTPUT.PUT_LINE('SUCCESS: ' || SUBSTR(l_response, 1, 100));
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('FAILED: ' || SQLERRM);
END;
/

-- Pass Criteria: No exceptions, response contains "data"
-- Fail Action: Verify Azure AI Foundry credentials, check network ACL

-- Test 1.3.2: Generate test embedding
SET SERVEROUTPUT ON;
BEGIN
    GENERATE_EMBEDDINGS_BATCH(
        p_batch_size => 1,
        p_workspace_endpoint => 'https://<workspace-name>.<region>.api.azureml.ms',
        p_deployment_name => 'text-embedding-3-small'
    );
END;
/

-- Pass Criteria: "Successfully processed: 1"
-- Fail Action: Check procedure logs, verify API limits
```

| Test ID | Description | Expected Result | Actual Result | Status |
|---------|-------------|-----------------|---------------|--------|
| UT-EMB-001 | Azure AI Foundry API accessible | HTTP 200 response | | ⬜ |
| UT-EMB-002 | Embedding dimensions | 1536 dimensions | | ⬜ |
| UT-EMB-003 | Batch processing | No errors | | ⬜ |

### Test 1.4: Vector Index Performance

**Objective:** Verify vector similarity search performs well

```sql
-- Test 1.4.1: Test vector search speed
SET TIMING ON;

DECLARE
    l_test_vector VECTOR(1536, FLOAT32);
BEGIN
    -- Get a sample vector
    SELECT NARRATIVE_VECTOR INTO l_test_vector
    FROM SIEBEL_KNOWLEDGE_VECTORS
    WHERE ROWNUM = 1;
    
    -- Perform similarity search
    FOR rec IN (
        SELECT 
            SR_ID,
            VECTOR_DISTANCE(NARRATIVE_VECTOR, l_test_vector, COSINE) AS SIMILARITY
        FROM SIEBEL_KNOWLEDGE_VECTORS
        ORDER BY VECTOR_DISTANCE(NARRATIVE_VECTOR, l_test_vector, COSINE) ASC
        FETCH FIRST 20 ROWS ONLY
    ) LOOP
        NULL; -- Just iterate
    END LOOP;
END;
/

SET TIMING OFF;

-- Pass Criteria: Query completes in < 100ms
-- Fail Action: Rebuild index, gather statistics
```

| Test ID | Description | Expected Result | Actual Result | Status |
|---------|-------------|-----------------|---------------|--------|
| UT-VEC-001 | Vector search speed | < 100ms | | ⬜ |
| UT-VEC-002 | Index used | INDEX SCAN in plan | | ⬜ |
| UT-VEC-003 | Results accurate | Top 20 returned | | ⬜ |

### Test 1.5: ORDS API Unit Tests

**Objective:** Test API endpoint in isolation

```bash
# Test 1.5.1: API accessibility
curl -X GET http://<oracle-23ai-vm-ip>:8080/ords/

# Expected: HTML welcome page or JSON response
# Pass Criteria: HTTP 200
# Fail Action: Verify ADB ORDS URL, check network connectivity

# Test 1.5.2: Search endpoint with valid query
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "X-API-Key: <API_KEY>" \
  -H "Top-K: 3" \
  -d "laptop screen broken" \
  http://<oracle-23ai-vm-ip>:8080/ords/semantic_search/siebel/search

# Expected: JSON with recommendations
# Pass Criteria: HTTP 200, valid JSON, 3 recommendations
# Fail Action: Check PL/SQL procedure, review API_SEARCH_LOG table

# Test 1.5.3: Error handling - empty query
curl -X POST \
  -H "Content-Type: text/plain" \
  -H "X-API-Key: <API_KEY>" \
  http://<oracle-23ai-vm-ip>:8080/ords/semantic_search/siebel/search

# Expected: Error message in JSON
# Pass Criteria: HTTP 200, error field present
# Fail Action: Update error handling in procedure

# Test 1.5.4: Error handling - missing API key
curl -X POST \
  -H "Content-Type: text/plain" \
  -d "test query" \
  http://<oracle-23ai-vm-ip>:8080/ords/semantic_search/siebel/search

# Expected: 401 Unauthorized
# Pass Criteria: HTTP 401
# Fail Action: Review security configuration
```

| Test ID | Description | Expected Result | Actual Result | Status |
|---------|-------------|-----------------|---------------|--------|
| UT-API-001 | ORDS accessible | HTTP 200 | | ⬜ |
| UT-API-002 | Valid query works | JSON returned | | ⬜ |
| UT-API-003 | Empty query error | Error message | | ⬜ |
| UT-API-004 | Auth enforced | HTTP 401 | | ⬜ |

### Test 1.6: Siebel Business Service

**Objective:** Test business service in isolation

1. Open Siebel Client as Administrator
2. Navigate to: **Administration - Business Service > Business Service Simulator**
3. Select Business Service: `SemanticSearchAPIService`
4. Select Method: `InvokeSearch`
5. Execute tests:

| Test ID | Input | Expected Output | Actual Output | Status |
|---------|-------|-----------------|---------------|--------|
| UT-BS-001 | QueryText="laptop broken"<br>TopK="5" | ResultSet with 5 recommendations | | ⬜ |
| UT-BS-002 | QueryText=""<br>TopK="5" | Error="Query text cannot be empty" | | ⬜ |
| UT-BS-003 | QueryText="test"<br>TopK="" | ResultSet with 5 recommendations (default) | | ⬜ |

---

## Phase 2: Integration Testing

**Duration:** 3-4 days  
**Executor:** QA Team

### Test 2.1: End-to-End Search Flow

**Objective:** Verify complete search flow from Siebel UI to API and back

**Test Procedure:**
1. Log in to Siebel application
2. Navigate to Service Request screen
3. Open browser DevTools (F12) → Network tab
4. Enter search query: "My computer is slow"
5. Click Search button
6. Monitor:
   - Loading indicator appears
   - API call in Network tab
   - Results display
   - Results are ranked

| Test ID | Query | Expected Behavior | Actual Behavior | Status |
|---------|-------|-------------------|-----------------|--------|
| IT-E2E-001 | "laptop broken" | Loading indicator → Results in < 3sec | | ⬜ |
| IT-E2E-002 | "password reset" | Top result related to passwords | | ⬜ |
| IT-E2E-003 | "" | Error message displayed | | ⬜ |
| IT-E2E-004 | Very long text (500+ chars) | Results or timeout error | | ⬜ |

### Test 2.2: Data Flow Integration

**Objective:** Verify data flows correctly through all layers

**Test Setup:**
```sql
-- Enable API logging
SELECT * FROM API_SEARCH_LOG
ORDER BY request_timestamp DESC
FETCH FIRST 1 ROWS ONLY;
-- Note the latest log_id
```

**Test Procedure:**
1. Perform search in Siebel: "printer not working"
2. Wait for results
3. Check API log:

```sql
SELECT 
    search_id,
    query_text,
    response_time_ms,
    result_count,
    error_message
FROM API_SEARCH_LOG
WHERE log_id > <noted_log_id>
ORDER BY request_timestamp DESC;
```

4. Verify in database:

```sql
-- Check which vectors were accessed
SELECT 
    v.SR_ID,
    v.CATALOG_PATH,
    VECTOR_DISTANCE(
        v.NARRATIVE_VECTOR,
        (SELECT NARRATIVE_VECTOR FROM SIEBEL_KNOWLEDGE_VECTORS WHERE ROWNUM = 1),
        COSINE
    ) AS DISTANCE
FROM SIEBEL_KNOWLEDGE_VECTORS v
WHERE CATALOG_PATH LIKE '%Printer%'
ORDER BY DISTANCE
FETCH FIRST 5 ROWS ONLY;
```

| Test ID | Check Point | Expected | Actual | Status |
|---------|-------------|----------|--------|--------|
| IT-FLOW-001 | API log entry created | 1 row | | ⬜ |
| IT-FLOW-002 | Response time logged | < 3000ms | | ⬜ |
| IT-FLOW-003 | Result count matches UI | Same count | | ⬜ |

### Test 2.3: Error Handling Integration

**Objective:** Verify graceful degradation when components fail

**Test 2.3.1: API Server Down**
```bash
# Stop ORDS
ps aux | grep ords
kill <ords_pid>
```

1. Try search in Siebel
2. Expected: "Search is temporarily unavailable" message
3. Verify: No application crash

**Test 2.3.2: Database Connection Loss**
```sql
-- Simulate connection issue (as ADMIN user in Oracle 23ai)
ALTER SYSTEM KILL SESSION '<sid>,<serial#>' IMMEDIATE;
```

1. Try search in Siebel
2. Expected: Error message, graceful handling

**Test 2.3.3: Invalid Response from API**
```sql
-- Temporarily break procedure to return invalid JSON
-- (Restore after test)
```

1. Try search
2. Expected: Error message, no JavaScript errors

| Test ID | Failure Scenario | Expected Behavior | Actual Behavior | Status |
|---------|-----------------|-------------------|-----------------|--------|
| IT-ERR-001 | ORDS down | User-friendly error | | ⬜ |
| IT-ERR-002 | DB connection lost | Graceful degradation | | ⬜ |
| IT-ERR-003 | Invalid JSON | Error message | | ⬜ |

---

## Phase 3: Performance Testing

**Duration:** 2-3 days  
**Executor:** Performance Testing Team

### Test 3.1: Single User Response Time

**Objective:** Measure baseline performance

**Test Procedure:**
1. Clear all caches
2. Execute 10 searches with different queries
3. Record response times

```sql
-- Query performance metrics
SELECT 
    query_text,
    response_time_ms,
    result_count
FROM API_SEARCH_LOG
WHERE request_timestamp >= SYSDATE - 1/24  -- Last hour
ORDER BY response_time_ms DESC;
```

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Average Response Time | < 2000ms | | ⬜ |
| P95 Response Time | < 3000ms | | ⬜ |
| P99 Response Time | < 5000ms | | ⬜ |
| Max Response Time | < 10000ms | | ⬜ |

### Test 3.2: Concurrent User Load Test

**Objective:** Test system under load

**Test Setup (JMeter):**
1. Install JMeter
2. Create test plan:

```xml
<!-- JMeter Test Plan -->
<TestPlan>
  <ThreadGroup>
    <numThreads>50</numThreads>
    <rampUp>60</rampUp>
    <loops>10</loops>
  </ThreadGroup>
  <HTTPSampler>
    <domain>${ORDS_HOST}</domain>
    <port>${ORDS_PORT}</port>
    <path>/ords/semantic_search/siebel/search</path>
    <method>POST</method>
    <headers>
      <Header name="Content-Type" value="text/plain"/>
      <Header name="X-API-Key" value="${API_KEY}"/>
    </headers>
    <body>${query}</body>
  </HTTPSampler>
</TestPlan>
```

3. Run test for 10 minutes
4. Collect results

| Metric | Target | 10 Users | 25 Users | 50 Users | Status |
|--------|--------|----------|----------|----------|--------|
| Avg Response Time | < 3000ms | | | | ⬜ |
| Throughput (req/sec) | > 10 | | | | ⬜ |
| Error Rate | < 1% | | | | ⬜ |
| CPU Usage | < 80% | | | | ⬜ |

### Test 3.3: Database Performance

**Objective:** Verify database handles load

```sql
-- Monitor during load test
SELECT 
    name,
    value
FROM v$sysstat
WHERE name IN (
    'physical reads',
    'logical reads',
    'execute count',
    'parse count (total)',
    'user commits'
)
ORDER BY name;

-- Monitor sessions
SELECT 
    COUNT(*) AS active_sessions,
    program,
    machine
FROM v$session
WHERE username = 'SEMANTIC_SEARCH'
  AND status = 'ACTIVE'
GROUP BY program, machine;

-- Check wait events
SELECT 
    event,
    total_waits,
    time_waited/100 AS time_waited_sec
FROM v$system_event
WHERE event NOT LIKE '%SQL*Net%'
  AND event NOT LIKE '%idle%'
ORDER BY time_waited DESC
FETCH FIRST 10 ROWS ONLY;
```

| Metric | Threshold | Measured | Status |
|--------|-----------|----------|--------|
| Active Sessions | < 100 | | ⬜ |
| DB CPU | < 80% | | ⬜ |
| I/O Wait | < 20% | | ⬜ |
| Locks/Blocks | 0 | | ⬜ |

---

## Phase 4: User Acceptance Testing (UAT)

**Duration:** 1-2 weeks  
**Executor:** Business Users / Subject Matter Experts

### Test 4.1: Golden Set Testing

**Objective:** Validate search relevance with business experts

**Test Setup:**
Create golden set of queries with expected results:

```sql
CREATE TABLE UAT_GOLDEN_SET (
    test_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    query_text VARCHAR2(1000),
    expected_rank1 VARCHAR2(200),
    expected_rank2 VARCHAR2(200),
    expected_rank3 VARCHAR2(200),
    tester_name VARCHAR2(100),
    test_date DATE,
    actual_rank1 VARCHAR2(200),
    actual_rank2 VARCHAR2(200),
    actual_rank3 VARCHAR2(200),
    pass_fail VARCHAR2(10),
    comments VARCHAR2(2000)
);

-- Insert test cases (created by business users)
INSERT INTO UAT_GOLDEN_SET (query_text, expected_rank1, expected_rank2, expected_rank3, tester_name)
VALUES (
    'I need to travel to Geneva for a conference',
    '> Travel > International Travel',
    '> Travel > Travel Request',
    '> Expense > Travel Expense',
    'John Smith'
);
-- Add 50-100 more test cases...
```

**Test Procedure:**
1. Business user executes each query
2. Records actual top 3 results
3. Marks as PASS if expected item is in top 3
4. Adds comments on relevance

**Success Criteria:**
- Precision@3 > 75% (75% of tests have expected item in top 3)
- User satisfaction score > 4.0/5.0

| Test ID | Query | Expected in Top 3? | Actual | P/F | Comments |
|---------|-------|-------------------|--------|-----|----------|
| UAT-001 | "laptop broken" | Hardware > Laptop | | | |
| UAT-002 | "password reset" | IT > Account Mgmt | | | |
| ... | ... | ... | | | |

### Test 4.2: Real-World Scenario Testing

**Scenarios:**

**Scenario 1: New Employee Onboarding**
- User: New employee Sarah
- Context: First day, needs various items
- Queries:
  1. "I need a laptop"
  2. "How do I get email access"
  3. "Need parking pass"
- Expected: Relevant HR/IT catalog items

**Scenario 2: IT Support**
- User: IT support agent Tom
- Context: Helping user with technical issue
- Queries:
  1. "User can't print"
  2. "Screen flickering"
  3. "Outlook not syncing"
- Expected: Relevant technical support items

**Scenario 3: Facilities Request**
- User: Office manager Lisa
- Context: Office maintenance
- Queries:
  1. "Conference room light broken"
  2. "Need office furniture"
  3. "Air conditioning not working"
- Expected: Relevant facilities items

| Scenario | User Feedback | Issues Found | P/F |
|----------|---------------|--------------|-----|
| Scenario 1 | | | ⬜ |
| Scenario 2 | | | ⬜ |
| Scenario 3 | | | ⬜ |

### Test 4.3: User Satisfaction Survey

Distribute survey to UAT participants:

```
1. How would you rate the search relevance? (1-5)
2. How would you rate the search speed? (1-5)
3. How would you rate the ease of use? (1-5)
4. How does this compare to the old search? (Much Better/Better/Same/Worse)
5. What improvements would you suggest?
6. Would you recommend deploying this to production? (Yes/No)
```

**Success Criteria:**
- Average rating > 4.0/5.0
- "Better" or "Much Better" > 80%
- "Recommend" > 80%

---

## 5. Regression Testing

**Objective:** Ensure existing Siebel functionality not impacted

| Test Area | Test Case | Expected | Actual | Status |
|-----------|-----------|----------|--------|--------|
| Standard Search | Use old search method | Still works | | ⬜ |
| SR Creation | Create new SR | No issues | | ⬜ |
| SR Update | Update existing SR | No issues | | ⬜ |
| Workflows | SR workflow triggers | All fire correctly | | ⬜ |
| Reporting | Standard reports | No errors | | ⬜ |

---

## 6. Security Testing

### Test 6.1: Authentication

| Test ID | Test Case | Expected | Actual | Status |
|---------|-----------|----------|--------|--------|
| SEC-001 | API call without key | 401 Unauthorized | | ⬜ |
| SEC-002 | API call with invalid key | 401 Unauthorized | | ⬜ |
| SEC-003 | API call with valid key | 200 OK | | ⬜ |

### Test 6.2: Authorization

| Test ID | Test Case | Expected | Actual | Status |
|---------|-----------|----------|--------|--------|
| SEC-010 | User without permissions | Cannot access | | ⬜ |
| SEC-011 | User with permissions | Can access | | ⬜ |

### Test 6.3: Data Security

| Test ID | Test Case | Expected | Actual | Status |
|---------|-----------|----------|--------|--------|
| SEC-020 | SQL injection attempt | Blocked/Sanitized | | ⬜ |
| SEC-021 | XSS attempt | Sanitized | | ⬜ |
| SEC-022 | Sensitive data in logs | Not logged | | ⬜ |

---

## 7. Test Summary Report Template

```markdown
# Test Summary Report
**Date:** <date>
**Environment:** <DEV/UAT/PROD>
**Tester:** <name>

## Summary
- Total Tests: XX
- Passed: XX
- Failed: XX
- Blocked: XX
- Pass Rate: XX%

## Phase Results
| Phase | Total | Passed | Failed | Pass % |
|-------|-------|--------|--------|--------|
| Unit | | | | |
| Integration | | | | |
| Performance | | | | |
| UAT | | | | |

## Critical Issues
1. [Issue description]
2. [Issue description]

## Recommendations
- [Recommendation 1]
- [Recommendation 2]

## Sign-off
- QA Lead: _________________ Date: _______
- Business Owner: __________ Date: _______
- Technical Lead: __________ Date: _______
```

---

## 8. Go/No-Go Criteria

The solution is ready for production if:

- ✅ All critical unit tests pass (100%)
- ✅ All integration tests pass (> 95%)
- ✅ Performance targets met (P95 < 3sec)
- ✅ UAT Precision@3 > 75%
- ✅ User satisfaction > 4.0/5.0
- ✅ No critical security issues
- ✅ Rollback plan tested
- ✅ Production support team trained
- ✅ Monitoring in place
- ✅ Business stakeholder approval

---

## 9. Post-Production Monitoring

After go-live, monitor for 2 weeks:

```sql
-- Daily monitoring query
SELECT 
    TRUNC(request_timestamp) AS date,
    COUNT(*) AS search_count,
    ROUND(AVG(response_time_ms)) AS avg_response_ms,
    COUNT(CASE WHEN error_message IS NOT NULL THEN 1 END) AS error_count,
    ROUND(AVG(result_count), 1) AS avg_results
FROM API_SEARCH_LOG
WHERE request_timestamp >= SYSDATE - 14
GROUP BY TRUNC(request_timestamp)
ORDER BY date DESC;
```

Alert on:
- Error rate > 1%
- Avg response time > 3000ms
- Search count drops > 50%
- Zero results > 10%

---

## Appendix: Test Data Cleanup

```sql
-- Clean up test data after testing
DELETE FROM API_SEARCH_LOG WHERE query_text LIKE '%test%';
DELETE FROM TEST_QUERIES;
DELETE FROM UAT_GOLDEN_SET;
COMMIT;
```

