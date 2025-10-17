# Project Architecture Guide: AI-Powered Semantic Search for Siebel CRM

**Version:** 2.2
**Date:** 2025-10-17
**Author:** Development Team

## 1. Introduction

### 1.1. Business Problem
The current service catalog search in the Siebel CRM application relies on keyword matching against administrator-defined names and descriptions. With over 10 years of historical data and millions of service requests, this method is inefficient and fails to leverage the rich, contextual information embedded in past user interactions. Users struggle to find the correct catalog item, leading to miscategorized requests, delays in resolution, and a suboptimal user experience.

### 1.2. Proposed Solution
This document outlines the architecture for a modern, AI-powered semantic search engine that replaces the existing keyword search. The new solution will understand the *intent* behind a user's natural language query. It will analyze the user's problem description, compare it to the vast knowledge base of historical service requests, work orders, and email communications, and intelligently recommend the most relevant service catalog items.

## 2. High-Level Architecture

The solution is designed with a simplified, highly performant architecture that integrates with the existing Siebel CRM system. It leverages Oracle REST Data Services (ORDS) to expose the AI search logic directly from the database, eliminating unnecessary network hops and simplifying the technology stack.



### 2.1. Component Breakdown

| Component | Technology Stack | Responsibility |
| :--- | :--- | :--- |
| **Data Source** | Oracle 12c (Siebel DB) | The existing system of record containing all historical CRM data. |
| **Data Extraction Pipeline** | Database Links, SQL | Extracts, aggregates, and prepares historical data for indexing using direct database-to-database connectivity. Runs as a batch process. |
| **Vector Database** | Oracle Database 23ai | Stores the text narratives and their corresponding vector embeddings. Hosts the high-performance HNSW vector index and the API logic. |
| **Embedding Service**| OCI Generative AI Service (Cohere Embed v3.0) | A managed cloud service that converts text narratives and user queries into numerical vectors (1024 dimensions), called from within the database via DBMS_CLOUD. |
| **Semantic Search API** | **Oracle REST Data Services (ORDS) & PL/SQL** | A high-performance RESTful API hosted directly by the Oracle database. The PL/SQL procedure encapsulates all AI search logic including vector similarity search and catalog aggregation. |
| **Siebel CRM** | Siebel Open UI, eScript, Custom Presentation Models | The user-facing application. Modified to call the ORDS endpoint via Named Subsystem and Business Service, displaying intelligent recommendations in the catalog search interface. |
| **Standalone Test App** | Python, Streamlit, Plotly | A web application for testing and demonstrating the semantic search API independently of Siebel. Provides side-by-side view of matching SRs and recommended catalog paths with analytics. See [TDD 5](TDD%205%20-%20Standalone%20Test%20Application.md) for complete specifications. |

### 2.2. Technology Stack Rationale

* **Oracle Database 23ai:** Chosen to leverage existing Oracle expertise, maintain data within a single secure ecosystem, and utilize its enterprise-grade AI Vector Search capabilities.
* **Oracle REST Data Services (ORDS):** This is the **preferred approach** for this project.
    * **Performance:** Eliminates the network hop between a separate API layer and the database, resulting in lower latency. The logic lives where the data lives.
    * **Simplicity:** Reduces the number of components to manage, secure, and deploy. Leverages existing PL/SQL skills.
    * **Security:** Enhances the security posture by removing the need to manage database credentials externally. Authentication is handled closer to the data.
* **OCI Generative AI Service:** Provides access to state-of-the-art embedding models without the overhead of hosting and managing them.

## 3. Data Flow

### 3.1. Offline Indexing Flow (Batch Process)
1.  A scheduled job initiates the **Data Extraction Pipeline**.
2.  An optimized SQL script runs on the **Oracle 12c** database, aggregating text for millions of resolved requests.
3.  The pipeline processes the extracted data, calls the **OCI Embedding Service** to get vectors, and inserts the data into the **Oracle 23ai Vector Database**.
4.  This process runs periodically (e.g., nightly) to keep the search index up-to-date.

### 3.2. Real-Time Search Flow (User Interaction)
1.  A user types a natural language query into the Siebel search UI.
2.  A Siebel eScript makes an HTTPS `POST` request to the **ORDS endpoint**, passing the user's query text.
3.  The ORDS listener routes the request to the `GET_SEMANTIC_RECOMMENDATIONS` PL/SQL procedure within the Oracle 23ai database.
4.  The PL/SQL procedure calls the **OCI Embedding Service** to convert the user's query into a vector.
5.  The procedure then executes a `VECTOR_DISTANCE` query against the local vector index to find the most similar historical records.
6.  The procedure aggregates the results to find the top 5 most frequently used catalog items.
7.  The procedure formats a JSON response and returns it directly to the Siebel application.
8.  The Siebel eScript parses the JSON and displays the recommended catalog items to the user.

## 4. Non-Functional Requirements

| Category | Requirement | Implementation Strategy |
| :--- | :--- | :--- |
| **Performance** | End-to-end search response time < 3 seconds (P95). | - HNSW vector index in Oracle 23ai.<br>- Co-location of API logic and data via ORDS. |
| **Scalability** | API must handle 50 concurrent users. Indexing must process 10,000+ new records nightly. | - ORDS and the Oracle Database are highly scalable.<br>- Batch processing for the indexing pipeline. |
| **Availability** | The search service should have 99.9% uptime. | - Oracle Database high-availability features (Data Guard, RAC).<br>- Graceful degradation in Siebel if the API is unreachable. |
| **Security** | All data in transit must be encrypted. Access to the API and database must be authenticated and authorized. | - TLS 1.2+ for all connections.<br>- ORDS endpoint protection (OAuth2 or Gateway API Keys).<br>- OCI Credentials stored securely within the database for callouts. |
| **Maintainability** | The solution must be modular and easy to update. | - Logic is encapsulated in a single, version-controlled PL/SQL package.<br>- Comprehensive logging and monitoring. |

## 5. Security Architecture
1.  **Network:** The Oracle 23ai database and its ORDS listener will reside within a private Virtual Cloud Network (VCN). An OCI Load Balancer or API Gateway will be the only public-facing entry point, terminating TLS. Network Security Groups (NSGs) will restrict traffic to the database listener port.
2.  **Authentication:** Siebel will authenticate to the ORDS endpoint via the Load Balancer or API Gateway (e.g., using a mandatory API Key). The ORDS endpoint itself can be further secured using ORDS-native privileges or OAuth2.
3.  **Callout Security:** The PL/SQL procedure's callout to the OCI Embedding Service is secured using a `DBMS_CLOUD` credential, which stores OCI API keys encrypted within the database, eliminating the need for external credential files or vaults for the API layer.

## 6. Standalone Test Application

### 6.1. Purpose and Benefits

The Standalone Test Application is a Python-based web application built with Streamlit that provides an independent testing and demonstration platform for the semantic search functionality. This application is essential for:

- **Pre-Siebel Testing**: Validate search functionality before full Siebel integration
- **Rapid Development**: Test API changes without redeploying Siebel components
- **QA Validation**: Execute systematic test cases with golden set queries
- **Stakeholder Demos**: Showcase semantic search capabilities to business users
- **Performance Analysis**: Measure and visualize response times and search patterns
- **Training**: Help users understand semantic search behavior

### 6.2. Key Features

The test application provides a rich user interface with the following capabilities:

**Search Interface**:
- Natural language query input with configurable Top-K parameter
- Real-time search execution with response time display
- Search history with replay functionality
- Batch query testing from CSV files

**Results Visualization**:
- **Left Panel**: Matching service requests with similarity scores, narratives, and catalog paths
- **Right Panel**: Recommended catalog paths with frequency counts and percentages
- Interactive expandable sections for detailed information
- Visual progress bars for similarity scores and recommendation percentages

**Analytics Dashboard**:
- Similarity score distribution histogram
- Catalog path frequency bar chart
- Response time trend analysis over search history
- Aggregate statistics (average similarity, median, standard deviation)

**Export Capabilities**:
- Export results to CSV for spreadsheet analysis
- Export results to JSON for programmatic processing
- Export search history with timestamps
- Download golden set test results

### 6.3. Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Streamlit Web UI                          │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │Search Panel│  │Results Panel │  │Analytics Dashboard │  │
│  └────────────┘  └──────────────┘  └────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Python Application Logic                        │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ORDS API      │  │Data Processor│  │Cache Manager    │  │
│  │Client        │  │              │  │                 │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
┌─────────────────────┐         ┌──────────────────────┐
│  ORDS REST API      │         │  Oracle 23ai DB      │
│  (Primary Mode)     │         │  (Optional Direct)   │
└─────────────────────┘         └──────────────────────┘
```

### 6.4. Technology Stack

| Technology | Version | Purpose |
|-----------|---------|---------|
| Streamlit | 1.28+ | Web application framework for rapid UI development |
| requests | 2.31+ | HTTP client for ORDS REST API communication |
| pandas | 2.1+ | Data manipulation and analysis |
| plotly | 5.18+ | Interactive visualizations and charts |
| oracledb | 1.4+ | Optional direct database connectivity |
| python-dotenv | 1.0+ | Environment variable management |

### 6.5. Usage Scenarios

**Scenario 1: Developer Testing**
- Developers use the test app to validate new embedding models or algorithm changes
- Execute standard test queries and compare results with baseline
- Export results for detailed analysis in spreadsheets

**Scenario 2: QA Validation**
- QA loads golden set test queries from CSV file
- Executes batch testing to validate search relevance
- Calculates Precision@3 metric automatically
- Exports results for test report generation

**Scenario 3: Business Demonstration**
- Product owners demonstrate semantic search to stakeholders
- Use realistic business queries to show relevance
- Display analytics dashboard showing performance metrics
- Compare "before and after" scenarios

**Scenario 4: Performance Testing**
- Performance testers execute series of searches with timestamps
- Analytics panel shows response time trends
- Identify slow queries and performance bottlenecks
- Export timing data for performance reports

### 6.6. Configuration

The test application is configured via environment variables in a `.env` file:

```ini
# API Configuration
ORDS_BASE_URL=http://localhost:8080/ords
API_KEY=your_secure_api_key

# Application Settings
DEFAULT_TOP_K=5
CACHE_ENABLED=true
ENABLE_DATABASE_MODE=false  # Set to true for direct DB queries
LOG_LEVEL=INFO
```

### 6.7. Deployment Options

**Local Development**:
```bash
streamlit run app.py --server.port 8501
```

**Docker Deployment**:
```bash
docker build -t semantic-search-test-app .
docker run -p 8501:8501 --env-file .env semantic-search-test-app
```

**Cloud Deployment**:
- Deploy to Streamlit Cloud with one-click deployment
- Configure secrets via Streamlit Cloud interface
- Accessible via public URL for remote testing

### 6.8. Documentation

Complete technical specifications for the Standalone Test Application are available in [TDD 5: Standalone Test Application](TDD%205%20-%20Standalone%20Test%20Application.md), including:

- Detailed component specifications
- API client implementation
- UI component designs
- Error handling strategies
- Security considerations
- Troubleshooting guide
- Future enhancements

