# Testing Guide: AI-Powered Semantic Search Solution

**Version:** 2.0
**Date:** 2025-10-16
**Owner:** QA Team

## 1. Introduction
This document outlines the testing strategy for the AI-Powered Semantic Search solution. The goal is to ensure the functional correctness, performance, relevance, and reliability of the end-to-end system.

## 2. Testing Phases

### 2.1. Unit Testing
* **Scope:** Individual components are tested in isolation.
* **Tools:** Postman (API), Pytest (Python), SQL Developer (SQL)

| Component | Test Cases |
| :--- | :--- | :--- |
| **Data Extraction SQL** | - Verify query runs without errors on a sample dataset.<br>- Check for correct aggregation of text from all sources.<br>- Validate that the catalog hierarchy path is generated correctly. |
| **Indexing Pipeline** | - Test ability to read the CSV format correctly.<br>- Mock OCI API calls to verify payload construction.<br>- Test database `MERGE` logic with new and existing SR_IDs. |
| **Semantic Search API** | - Test `POST /search` with valid queries.<br>- Test with missing or empty `query` field (expect HTTP 400).<br>- Test the RRF ranking logic with a controlled dataset.<br>- Test API key authentication (valid and invalid keys). |
| **Siebel eScript** | - Mock the API response to test PropertySet parsing.<br>- Verify correct construction of the `SearchSpec` and `SortSpec` strings.<br>- Test error handling and user-facing error messages. |

### 2.2. Integration Testing
* **Scope:** Testing the complete, end-to-end data flow.
* **Owner:** QA Team

| Scenario ID | Test Case | Expected Result |
| :--- | :--- | :--- |
| IT-001 | A user enters a valid search query in the Siebel UI. | 1. A loading indicator appears.<br>2. The API returns HTTP 200.<br>3. The results applet is refreshed with ranked results. |
| IT-002 | The Search API is temporarily down. | The Siebel UI displays a user-friendly error message ("Search is temporarily unavailable.") and does not crash. |
| IT-003 | A new SR is resolved, and the nightly indexing job runs. | A search query related to the new SR now returns the correct catalog item as a potential recommendation. |

### 2.3. Performance Testing
* **Scope:** Measuring the response time and load capacity of the API.
* **Tools:** JMeter, OCI Monitoring

| Scenario ID | Test Case | Success Criteria |
| :--- | :--- | :--- |
| PT-001 | **Baseline Response Time:** Single user search. | End-to-end response time (Siebel UI) < 3 seconds. |
| PT-002 | **Load Test:** 50 concurrent users making requests. | - API P95 latency < 1.5 seconds.<br>- API error rate < 0.1%.<br>- OCI Function CPU/Memory usage within limits. |
| PT-003 | **Cold Start Test:** First API call after a period of inactivity. | The first call latency should be acceptable (< 4-5 seconds). |

### 2.4. Relevance and User Acceptance Testing (UAT)
* **Scope:** Business users validate the quality and relevance of the search results. This is the most critical phase for an AI project.
* **Owner:** Business Analysts / Product Owner

#### 2.4.1. Methodology: Golden Set Testing
A "golden set" of 50-100 real-world, anonymized user queries will be created. For each query, a panel of business experts will manually determine the top 3 most relevant catalog items.

#### 2.4.2. UAT Scenarios
The test will involve executing each query from the golden set against the system and comparing the system's recommendations to the experts' choices.

| Scenario ID | Sample Query | Expected Top Recommendation(s) |
| :--- | :--- | :--- |
| UAT-001 | "I need to travel to Geneva next month for a conference." | `> Travel > International Travel Request` |
| UAT-002 | "My office chair is broken and I need a new one." | `> Facilities > Office Furniture > Chair Replacement` |
| UAT-003 | "I can't log in to the finance application" | `> IT Support > Account Management > Password Reset` |

#### 2.4.3. Success Metrics
* **Precision@3 (P@3):** Of the top 3 items recommended by the system, what percentage were also in the experts' top 3? **Target: > 75%**
* **Qualitative Feedback:** A survey will be provided to UAT testers to rate their satisfaction with the search relevance on a scale of 1-5. **Target: > 4.0 average score.**
