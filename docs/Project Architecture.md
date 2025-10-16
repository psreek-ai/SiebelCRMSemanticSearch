# Project Architecture Guide: AI-Powered Semantic Search for Siebel CRM

**Version:** 2.0
**Date:** 2025-10-16
**Author:** AI Assistant

## 1. Introduction

### 1.1. Business Problem
The current service catalog search in the Siebel CRM application relies on keyword matching against administrator-defined names and descriptions. With over 10 years of historical data and millions of service requests, this method is inefficient and fails to leverage the rich, contextual information embedded in past user interactions. Users struggle to find the correct catalog item, leading to miscategorized requests, delays in resolution, and a suboptimal user experience.

### 1.2. Proposed Solution
This document outlines the architecture for a modern, AI-powered semantic search engine that replaces the existing keyword search. The new solution will understand the *intent* behind a user's natural language query. It will analyze the user's problem description, compare it to the vast knowledge base of historical service requests, work orders, and email communications, and intelligently recommend the most relevant service catalog items.

## 2. High-Level Architecture

The solution is designed as a decoupled, microservices-oriented architecture that integrates with the existing Siebel CRM system without altering its core functionality. It consists of three main parts: an offline data pipeline for indexing, a real-time API for searching, and a lightweight integration layer within Siebel.



### 2.1. Component Breakdown

| Component | Technology Stack | Responsibility |
| :--- | :--- | :--- |
| **Data Source** | Oracle 12c (Siebel DB) | The existing system of record containing all historical CRM data. |
| **Data Extraction Pipeline** | SQL, Python (Optional) | Extracts, aggregates, and prepares historical data for indexing. Runs as a batch process. |
| **Vector Database** | Oracle Database 23ai | Stores the text narratives and their corresponding vector embeddings. Hosts the high-performance vector index. |
| **Embedding Service**| OCI Generative AI Service | A managed cloud service that converts text narratives and user queries into numerical vectors (embeddings). |
| **Semantic Search API** | OCI Functions (Python/FastAPI) | A RESTful API that encapsulates all AI search logic. It receives queries, generates vectors, queries the vector DB, and returns ranked results. |
| **Siebel CRM** | Siebel Open UI, eScript | The user-facing application. Modified to call the Semantic Search API and display the intelligent recommendations. |
| **Standalone Test App**| Streamlit / HTML+JS | A simple web application for testing and demonstrating the API's search capabilities independently of Siebel. |

### 2.2. Technology Stack Rationale

* **Oracle Database 23ai:** Chosen to leverage existing Oracle expertise, maintain data within a single secure ecosystem, and utilize its enterprise-grade AI Vector Search capabilities.
* **OCI Generative AI Service:** Provides access to state-of-the-art embedding models without the overhead of hosting and managing them.
* **OCI Functions:** A serverless approach for the API is cost-effective, scalable, and reduces infrastructure management.
* **Decoupled API:** The API-first approach ensures that Siebel is shielded from the complexity of the AI backend. This allows the AI components to be upgraded independently and reduces risk to the core CRM system.

## 3. Data Flow

### 3.1. Offline Indexing Flow (Batch Process)
1.  A scheduled job initiates the **Data Extraction Pipeline**.
2.  An optimized SQL script runs on the **Oracle 12c** database, aggregating text from `S_SRV_REQ`, `S_ORDER`, and `S_EVT_ACT` for millions of resolved requests.
3.  The pipeline processes the extracted data in batches. For each record, it calls the **OCI Embedding Service** to get a vector.
4.  The pipeline then inserts the original text, the vector, and relevant metadata (SR_ID, Catalog Path) into the **Oracle 23ai Vector Database**.
5.  This process runs periodically (e.g., nightly) to keep the search index up-to-date with newly resolved service requests.

### 3.2. Real-Time Search Flow (User Interaction)
1.  A user types a natural language query into the Siebel search UI (e.g., "The fan on my laptop is very loud and it's running hot").
2.  A Siebel eScript calls the **Semantic Search API** endpoint, passing the user's query text.
3.  The API calls the **OCI Embedding Service** to convert the user's query into a vector.
4.  The API executes a `VECTOR_DISTANCE` query against the **Oracle 23ai Vector Database** to find the most similar historical records.
5.  The API aggregates the results to find the top 5 most frequently used catalog items for those similar records.
6.  The API returns a ranked JSON list of the top 5 catalog items.
7.  The Siebel eScript parses the JSON and displays the recommended catalog items to the user.

## 4. Non-Functional Requirements

| Category | Requirement | Implementation Strategy |
| :--- | :--- | :--- |
| **Performance** | End-to-end search response time < 3 seconds (P95). | - HNSW vector index in Oracle 23ai.<br>- Optimized API logic.<br>- Caching of embedding models (managed by OCI). |
| **Scalability** | API must handle 50 concurrent users. Indexing must process 10,000+ new records nightly. | - OCI Functions for auto-scaling API layer.<br>- Batch processing for the indexing pipeline. |
| **Availability** | The search service should have 99.9% uptime. | - Deployment across multiple OCI Availability Domains.<br>- Graceful degradation in Siebel if the API is unreachable. |
| **Security** | All data in transit must be encrypted. Access to the API and database must be authenticated and authorized. | - TLS 1.2+ for all connections.<br>- API Gateway with API Keys.<br>- OCI IAM policies and Vault for credentials. |
| **Maintainability** | The solution must be modular and easy to update. | - Decoupled API architecture.<br>- Comprehensive logging and monitoring.<br>- Infrastructure as Code (Terraform) for deployments. |

## 5. Security Architecture
1.  **Network:** All components (OCI Functions, Oracle DB) will reside within a private Virtual Cloud Network (VCN). The API Gateway will be the only public-facing entry point. Network Security Groups (NSGs) will restrict traffic between components to only the necessary ports.
2.  **Authentication:** Siebel will authenticate to the API Gateway using a mandatory API Key. The API Function will authenticate to the Oracle 23ai database using credentials stored securely in OCI Vault.
3.  **Authorization:** OCI IAM policies will be configured to grant the API Function the minimum required permissions (e.g., read secrets from Vault, execute functions). The database user for the API will have read-only access to the vector table.
```eof
