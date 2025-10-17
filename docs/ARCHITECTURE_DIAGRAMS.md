# Architecture Diagrams - Semantic Search Solution

**Version:** 1.0  
**Date:** October 17, 2025  
**Purpose:** Comprehensive visual documentation of system architecture

---

## Important Architecture Note

**Oracle Database 23ai on Azure VM:**

This solution uses **Oracle Database 23ai installed on Azure Virtual Machines**, providing a self-managed database deployment with full control and flexibility. This is a **100% Azure-native solution**.

**Key Points:**
- ✅ **Oracle 23ai on Azure VM** - Self-managed installation on Azure infrastructure with full administrative control
- ✅ **Azure AI Foundry with OpenAI Service** - Unified AI development platform for embeddings and future AI capabilities
- ✅ **Private connectivity** - All traffic stays within Azure VNet with Private Endpoints to AI Foundry
- ✅ **Azure-native experience** - All resources (VMs, AI Foundry, Container Apps) in your Azure subscription
- ✅ **Unified management** - Managed through Azure Portal for complete stack
- ✅ **Low latency** - Direct in-region connectivity between all components
- ✅ **100% Azure** - No cross-cloud dependencies

**AI Platform:**
- **Azure AI Foundry** - Comprehensive AI development platform (formerly Azure AI Studio)
- **OpenAI Service**: `text-embedding-3-small` (1536 dimensions) or `text-embedding-3-large` (3072 dimensions)
- **Platform Benefits**: Prompt flow, model evaluation, responsible AI tools, unified governance
- **Cost**: Competitive pricing at $0.02 per million tokens for embeddings
- **Access**: From Oracle 23ai VM via DBMS_CLOUD to Azure AI Foundry endpoint with Private Endpoint

**Database Architecture:**
- **Oracle Database 23ai** - Self-managed on Azure VM with native AI Vector Search (HNSW indexing)
- **Manual ORDS Installation** - Installed and configured on the same VM (http://localhost:8080/ords)
- **Full Control** - Complete control over configuration, tuning, patching, and resource allocation
- **Siebel Database** - Oracle 19c (corrected from 12c)

**Why This Architecture?**
- Complete control over database configuration and optimization
- Azure AI Foundry provides unified platform for current and future AI capabilities
- Simplifies networking (all components within Azure VNet)
- Flexible deployment and scaling based on specific requirements
- Lower costs with traditional Oracle licensing or BYOL options
- Single-pane-of-glass management through Azure Portal

---

## Table of Contents

1. [Overview Architecture](#1-overview-architecture)
2. [Component Architecture](#2-component-architecture)
3. [Data Flow Architecture](#3-data-flow-architecture)
4. [Network Architecture](#4-network-architecture)
5. [Security Architecture](#5-security-architecture)
6. [Deployment Architecture](#6-deployment-architecture)
7. [Vector Search Architecture](#7-vector-search-architecture)
8. [Integration Architecture](#8-integration-architecture)

---

## 1. Overview Architecture

### 1.1. System Context Diagram

```mermaid
C4Context
    title System Context Diagram - Semantic Search Solution

    Person(user, "Siebel User", "Service desk agent or customer")
    Person(admin, "Administrator", "System admin managing deployment")
    
    System_Boundary(azure, "Microsoft Azure") {
        System(siebel, "Siebel CRM", "Legacy CRM system with Oracle 19c")
        System(testapp, "Test Application", "Streamlit validation app")
        System(oracle23ai, "Oracle 23ai VM", "AI Vector Search + ORDS API")
        System(aifoundry, "Azure AI Foundry", "OpenAI Service for embeddings")
    }
    
    Rel(user, siebel, "Searches for catalog items", "HTTPS")
    Rel(admin, testapp, "Tests and validates", "HTTPS")
    Rel(siebel, oracle23ai, "API calls for semantic search", "HTTP/ORDS")
    Rel(testapp, oracle23ai, "API calls for testing", "HTTP/ORDS")
    Rel(oracle23ai, aifoundry, "Generate embeddings", "HTTPS")
    
    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="2")
```

### 1.2. High-Level Architecture

```mermaid
graph TB
    subgraph Users["Users"]
        EndUser["End Users<br/>(Natural Language Queries)"]
        TestUser["QA/Admins<br/>(Testing & Validation)"]
    end
    
    subgraph AzureCloud["Microsoft Azure Environment"]
        subgraph SiebelEnv["Siebel Environment (VM)"]
            SiebelUI["Siebel Open UI<br/>(Web Interface)"]
            SiebelDB["Oracle 19c Database<br/>(10 Years Historical Data)"]
        end
        
        subgraph TestEnv["Test Environment"]
            Streamlit["Streamlit Test App<br/>(Azure Container Apps)"]
        end
        
        subgraph VectorVM["Oracle 23ai VM (Self-Managed)"]
            ORDS["ORDS<br/>(Manually Installed)<br/>REST API Layer"]
            VectorEngine["Oracle 23ai Database<br/>AI Vector Search<br/>(HNSW Index)"]
            PLSQLLogic["PL/SQL Search Logic<br/>(Business Rules)"]
        end
        
        subgraph AzureNetwork["Azure Networking"]
            VNet["Virtual Network<br/>(Private Connectivity)"]
            NSG["Network Security Groups"]
            PrivateEndpoint["Private Endpoint<br/>(to AI Foundry)"]
        end
        
        subgraph AzureAI["Azure AI Services"]
            AzureOpenAI["Azure OpenAI Service<br/>(text-embedding-3-small)<br/>1536-dimensional vectors"]
        end
        
        subgraph AzureAI["Azure AI Foundry"]
            AIFoundry["AI Foundry Workspace<br/>(AI Development Platform)"]
            AzureOpenAI["OpenAI Service<br/>(text-embedding-3-small)<br/>1536 dimensions"]
        end
    end
    
    EndUser -->|1. Search Query| SiebelUI
    TestUser -->|1. Test Query| Streamlit
    
    SiebelUI -->|2. HTTP POST<br/>via VNet| ORDS
    Streamlit -->|2. HTTP POST<br/>via VNet| ORDS
    
    SiebelDB -.->|Batch: Database Link<br/>via VNet| VectorEngine
    
    ORDS -->|3. Execute| PLSQLLogic
    PLSQLLogic -->|4. Embed Query<br/>via Private Endpoint| AIFoundry
    AIFoundry -->|Route to| AzureOpenAI
    AzureOpenAI -->|5. Return Vector| PLSQLLogic
    PLSQLLogic -->|6. Vector Search| VectorEngine
    VectorEngine -->|7. Top Matches| PLSQLLogic
    PLSQLLogic -->|8. JSON Response| ORDS
    
    ORDS -->|9. Results| SiebelUI
    ORDS -->|9. Results| Streamlit
    SiebelUI -->|10. Display| EndUser
    Streamlit -->|10. Display| TestUser
    
    VNet -.->|Private Connectivity| VectorVM
    VNet -.->|Private Endpoint| AIFoundry
    
    classDef azure fill:#0078D4,stroke:#004578,color:#fff
    classDef vm fill:#8B4789,stroke:#5C2D5A,color:#fff
    classDef ai fill:#10A37F,stroke:#0D8A6A,color:#fff
    classDef user fill:#107C10,stroke:#094509,color:#fff
    
    class AzureCloud,SiebelEnv,TestEnv,AzureNetwork,VNet,NSG,PrivateEndpoint,AzureAI azure
    class VectorVM,ORDS,VectorEngine,PLSQLLogic vm
    class AIFoundry,AzureOpenAI ai
    class Users,EndUser,TestUser user
```

---

## 2. Component Architecture

### 2.1. Component Interaction Diagram

```mermaid
graph TB
    subgraph Presentation["Presentation Layer"]
        SiebelUI["Siebel Open UI<br/>Custom Presentation Model"]
        StreamlitUI["Streamlit Web UI<br/>Python + Plotly"]
    end
    
    subgraph Integration["Integration Layer"]
        SiebelBS["Siebel Business Service<br/>(Named Subsystem)"]
        StreamlitClient["Python API Client<br/>(requests library)"]
    end
    
    subgraph API["API Layer (ORDS)"]
        ORDSEndpoint["REST Endpoint<br/>/semantic_search/siebel/recommendations"]
        ORDSAuth["Authentication<br/>(OAuth2/API Key)"]
        ORDSRouter["Request Router<br/>(Handler Mapping)"]
    end
    
    subgraph Business["Business Logic Layer (PL/SQL)"]
        SearchProc["GET_SEMANTIC_RECOMMENDATIONS<br/>(Main Procedure)"]
        EmbedFunc["EMBED_TEXT<br/>(Embedding Function)"]
        VectorFunc["VECTOR_SEARCH<br/>(Similarity Search)"]
        AggFunc["AGGREGATE_CATALOG<br/>(Ranking Logic)"]
    end
    
    subgraph Data["Data Layer"]
        VectorTable["VECTOR_INDEX<br/>(Service Requests + Embeddings)"]
        CatalogTable["CATALOG_MAPPING<br/>(Category Hierarchy)"]
        AuditTable["SEARCH_AUDIT_LOG<br/>(Query History)"]
    end
    
    subgraph External["External Services"]
        AIFoundry["Azure AI Foundry<br/>(OpenAI Service via DBMS_CLOUD)"]
        SiebelDB["Siebel Database<br/>(Oracle 19c - Database Link)"]
    end
    
    SiebelUI -->|User Query| SiebelBS
    StreamlitUI -->|User Query| StreamlitClient
    
    SiebelBS -->|HTTPS POST| ORDSEndpoint
    StreamlitClient -->|HTTPS POST| ORDSEndpoint
    
    ORDSEndpoint --> ORDSAuth
    ORDSAuth --> ORDSRouter
    ORDSRouter --> SearchProc
    
    SearchProc --> EmbedFunc
    EmbedFunc --> AIFoundry
    AIFoundry --> EmbedFunc
    
    SearchProc --> VectorFunc
    VectorFunc --> VectorTable
    VectorTable --> VectorFunc
    
    SearchProc --> AggFunc
    AggFunc --> CatalogTable
    
    SearchProc --> AuditTable
    
    SiebelDB -.->|Nightly ETL| VectorTable
    
    SearchProc -->|JSON Response| ORDSRouter
    ORDSRouter -->|JSON| ORDSEndpoint
    ORDSEndpoint -->|JSON| SiebelBS
    ORDSEndpoint -->|JSON| StreamlitClient
    
    SiebelBS --> SiebelUI
    StreamlitClient --> StreamlitUI
    
    classDef presentation fill:#68B984,stroke:#3D7A52,color:#fff
    classDef integration fill:#FFB900,stroke:#C79400,color:#000
    classDef api fill:#0078D4,stroke:#004578,color:#fff
    classDef business fill:#107C10,stroke:#094509,color:#fff
    classDef data fill:#336791,stroke:#1E3A5F,color:#fff
    classDef external fill:#C74634,stroke:#8B2F23,color:#fff
    
    class Presentation,SiebelUI,StreamlitUI presentation
    class Integration,SiebelBS,StreamlitClient integration
    class API,ORDSEndpoint,ORDSAuth,ORDSRouter api
    class Business,SearchProc,EmbedFunc,VectorFunc,AggFunc business
    class Data,VectorTable,CatalogTable,AuditTable data
    class External,AIFoundry,SiebelDB external
```

### 2.2. Technology Stack Layers

```mermaid
graph LR
    subgraph Frontend["Frontend Technologies"]
        F1["Siebel Open UI<br/>(JavaScript/eScript)"]
        F2["Streamlit<br/>(Python)"]
        F3["Plotly<br/>(Visualization)"]
    end
    
    subgraph APILayer["API Technologies"]
        A1["Oracle REST Data Services<br/>(Manually Installed)"]
        A2["JSON over HTTP<br/>(RESTful)"]
        A3["OAuth2/API Keys<br/>(Security)"]
    end
    
    subgraph DatabaseLayer["Database Technologies"]
        D1["Oracle Database 23ai<br/>(Self-Managed on Azure VM)"]
        D2["PL/SQL<br/>(Business Logic)"]
        D3["AI Vector Search<br/>(HNSW Index)"]
        D4["DBMS_CLOUD<br/>(Azure Integration)"]
    end
    
    subgraph AILayer["AI/ML Technologies"]
        AI1["Azure AI Foundry<br/>(OpenAI Service - text-embedding-3-small)"]
        AI2["Vector Embeddings<br/>(1536 dimensions)"]
        AI3["Cosine Similarity<br/>(VECTOR_DISTANCE)"]
    end
    
    subgraph Infrastructure["Infrastructure"]
        I1["Azure VMs<br/>(Siebel + Oracle 23ai)"]
        I2["Azure Container Apps<br/>(Test App)"]
        I3["Azure AI Foundry<br/>(AI Platform)"]
        I4["Azure Networking<br/>(VNet, NSG, Private Endpoints)"]
    end
    
    Frontend --> APILayer
    APILayer --> DatabaseLayer
    DatabaseLayer --> AILayer
    Infrastructure -.->|Hosts| Frontend
    Infrastructure -.->|Hosts| APILayer
    Infrastructure -.->|Hosts| DatabaseLayer
    
    classDef frontend fill:#68B984,stroke:#3D7A52,color:#fff
    classDef api fill:#0078D4,stroke:#004578,color:#fff
    classDef db fill:#336791,stroke:#1E3A5F,color:#fff
    classDef ai fill:#C74634,stroke:#8B2F23,color:#fff
    classDef infra fill:#5E5E5E,stroke:#2E2E2E,color:#fff
    
    class Frontend,F1,F2,F3 frontend
    class APILayer,A1,A2,A3 api
    class DatabaseLayer,D1,D2,D3,D4 db
    class AILayer,AI1,AI2,AI3 ai
    class Infrastructure,I1,I2,I3,I4 infra
```

---

## 3. Data Flow Architecture

### 3.1. Real-Time Search Flow (Detailed)

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant S as Siebel UI
    participant BS as Business Service
    participant API as ORDS API
    participant PL as PL/SQL Logic
    participant CACHE as Query Cache
    participant AI as Azure AI Foundry
    participant VDB as Vector Index
    participant CAT as Catalog DB
    participant LOG as Audit Log
    
    Note over U,LOG: Real-Time Semantic Search Flow
    
    U->>S: Enter query: "laptop wifi problem"
    S->>BS: Invoke SearchRecommendations()
    BS->>API: POST /recommendations<br/>+ API Key Header
    
    API->>API: Validate API Key
    API->>PL: Execute GET_SEMANTIC_RECOMMENDATIONS
    
    PL->>CACHE: Check cache for query
    alt Cache Hit
        CACHE-->>PL: Return cached results
    else Cache Miss
        PL->>AI: DBMS_CLOUD.send_request<br/>(embed query text)
        AI-->>PL: Return embedding vector<br/>[1024 floats]
        
        PL->>VDB: SELECT ... WHERE<br/>VECTOR_DISTANCE(embedding, query_vec)<br/>ORDER BY similarity<br/>FETCH FIRST 100 ROWS
        VDB-->>PL: Top 100 matching SRs<br/>with catalog IDs
        
        PL->>CAT: Join to get catalog paths
        CAT-->>PL: Full catalog hierarchy
        
        PL->>PL: Aggregate by catalog<br/>Calculate frequencies<br/>Rank top 5
        
        PL->>CACHE: Store result in cache
    end
    
    PL->>LOG: Insert search audit record
    PL-->>API: Return JSON response
    API-->>BS: JSON with recommendations
    BS-->>S: Parse and format results
    S-->>U: Display top 5 catalog items<br/>with confidence scores
    
    Note over U,LOG: Total Time: < 3 seconds (P95)
```

### 3.2. Batch Indexing Flow (Detailed)

```mermaid
sequenceDiagram
    autonumber
    participant CRON as Scheduler<br/>(Nightly Job)
    participant ETL as ETL Process<br/>(PL/SQL)
    participant SIEBEL as Oracle 19c<br/>(Siebel database)
    participant STAGE as Staging Table
    participant AI as Azure AI Foundry<br/>(OpenAI)
    participant VDB as Vector Index
    participant STATS as Statistics
    
    Note over CRON,STATS: Nightly Batch Indexing (2:00 AM)
    
    CRON->>ETL: Trigger index_new_records()
    ETL->>SIEBEL: Query via Database Link:<br/>SELECT last 24h resolved SRs
    SIEBEL-->>ETL: ~500-2000 new records
    
    ETL->>ETL: Aggregate text:<br/>SR + Work Orders + Emails
    ETL->>STAGE: INSERT INTO staging_table
    
    loop Process in batches of 100
        ETL->>STAGE: SELECT 100 records
        
        ETL->>AI: DBMS_CLOUD.send_request<br/>(batch embed 100 texts)
        AI-->>ETL: Return 100 embeddings (1536-dim)
        
        ETL->>VDB: MERGE INTO vector_index<br/>(UPSERT records + embeddings)
        VDB-->>ETL: 100 rows merged
        
        ETL->>STATS: Update progress counter
    end
    
    ETL->>VDB: DBMS_VECTOR.rebuild_index()<br/>(if > 10% new data)
    VDB-->>ETL: Index rebuilt
    
    ETL->>STATS: Log job statistics:<br/>- Records processed<br/>- Errors<br/>- Duration
    
    ETL-->>CRON: Job completed successfully
    
    Note over CRON,STATS: Process Duration: 30-90 minutes
```

### 3.3. Data Transformation Pipeline

```mermaid
graph LR
    subgraph Source["Data Sources (Siebel Oracle 19c)"]
        SR[Service Request<br/>- Abstract<br/>- Detailed Description]
        WO[Work Orders<br/>- Action Log<br/>- Resolution Notes]
        EMAIL[Email Activities<br/>- Body Text<br/>- Attachments]
    end
    
    subgraph Extract["Extraction Layer"]
        DBLINK[Database Link<br/>SELECT with filters]
        AGG[Text Aggregation<br/>CONCAT fields]
    end
    
    subgraph Transform["Transformation Layer"]
        CLEAN[Text Cleaning<br/>- Remove HTML<br/>- Normalize whitespace]
        ENRICH[Enrichment<br/>- Add catalog path<br/>- Add metadata]
        VALIDATE[Validation<br/>- Check completeness<br/>- Filter invalid]
    end
    
    subgraph AI["AI Embedding"]
        BATCH[Batch Processing<br/>100 records/request]
        EMBED[Azure AI Foundry<br/>OpenAI text-embedding-3-small]
        VECTOR[Generate Vectors<br/>1536 dimensions]
    end
    
    subgraph Load["Load Layer"]
        STAGE[Staging Table<br/>Temporary storage]
        MERGE[MERGE Operation<br/>UPSERT logic]
        INDEX[Vector Index<br/>HNSW structure]
    end
    
    SR --> DBLINK
    WO --> DBLINK
    EMAIL --> DBLINK
    
    DBLINK --> AGG
    AGG --> CLEAN
    CLEAN --> ENRICH
    ENRICH --> VALIDATE
    
    VALIDATE --> BATCH
    BATCH --> EMBED
    EMBED --> VECTOR
    
    VECTOR --> STAGE
    STAGE --> MERGE
    MERGE --> INDEX
    
    classDef source fill:#FFB900,stroke:#C79400,color:#000
    classDef extract fill:#0078D4,stroke:#004578,color:#fff
    classDef transform fill:#68B984,stroke:#3D7A52,color:#fff
    classDef ai fill:#C74634,stroke:#8B2F23,color:#fff
    classDef load fill:#336791,stroke:#1E3A5F,color:#fff
    
    class Source,SR,WO,EMAIL source
    class Extract,DBLINK,AGG extract
    class Transform,CLEAN,ENRICH,VALIDATE transform
    class AI,BATCH,EMBED,VECTOR ai
    class Load,STAGE,MERGE,INDEX load
```

---

## 4. Network Architecture

### 4.1. Network Topology

```mermaid
graph TB
    subgraph Internet["Public Internet"]
        Users["End Users<br/>(HTTPS Only)"]
    end
    
    subgraph AzureRegion["Azure Region: East US"]
        subgraph VNET["Virtual Network (10.0.0.0/16)"]
            subgraph AppSubnet["Application Subnet (10.0.1.0/24)"]
                SiebelVM["Siebel VM<br/>10.0.1.10"]
                Oracle12c["Oracle 19c DB<br/>10.0.1.11"]
            end
            
            subgraph ContainerSubnet["Container Subnet (10.0.2.0/24)"]
                ContainerEnv["Container Apps Environment"]
                TestApp["Streamlit App<br/>10.0.2.20"]
            end
            
            subgraph PrivateSubnet["Private Link Subnet (10.0.3.0/24)"]
                PrivateEndpoint["Private Endpoint<br/>10.0.3.100"]
            end
            
            subgraph SecurityGroup["Network Security Groups"]
                NSG1["NSG-Siebel<br/>- Allow 443 inbound<br/>- Allow 1521 to ADB"]
                NSG2["NSG-Container<br/>- Allow 443 inbound<br/>- Allow HTTPS to ADB"]
                NSG3["NSG-PrivateLink<br/>- Allow Azure → OCI"]
            end
        end
        
        subgraph Management["Management Services"]
            Bastion["Azure Bastion<br/>(Secure SSH/RDP)"]
            Firewall["Azure Firewall<br/>(Optional)"]
        end
    end
    
    subgraph ODSAService["Oracle Database Service for Azure (within Azure)"]
        subgraph ADB["Oracle 23ai Database"]
            ORDS["ORDS Endpoint<br/>*.adb.oraclecloudapps.com"]
            VectorDB["Vector Database<br/>AI Vector Search"]
        end
    end
    
    subgraph OCIBackend["OCI Backend (via Azure-OCI Interconnect)"]
        GenAI["Azure AI Foundry<br/>(Regional Service)"]
    end
    
    Users -->|TLS 1.3| SiebelVM
    Users -->|TLS 1.3| TestApp
    
    SiebelVM -.->|NSG1| NSG1
    TestApp -.->|NSG2| NSG2
    
    SiebelVM -->|HTTPS:443| PrivateEndpoint
    TestApp -->|HTTPS:443| PrivateEndpoint
    
    PrivateEndpoint -.->|NSG3| NSG3
    PrivateEndpoint -->|Private Link<br/>within Azure| ADB
    
    ADB -->|Internal| ORDS
    ADB -->|HTTPS via<br/>Azure-OCI Interconnect| GenAI
    
    Bastion -.->|Secure Access| SiebelVM
    Firewall -.->|Optional Filter| VNET
    
    classDef internet fill:#5E5E5E,stroke:#2E2E2E,color:#fff
    classDef azure fill:#0078D4,stroke:#004578,color:#fff
    classDef security fill:#107C10,stroke:#094509,color:#fff
    classDef odsa fill:#8B4789,stroke:#5C2D5A,color:#fff
    classDef oci fill:#C74634,stroke:#8B2F23,color:#fff
    
    class Internet,Users internet
    class AzureRegion,VNET,AppSubnet,ContainerSubnet,PrivateSubnet,Management,SiebelVM,Oracle12c,ContainerEnv,TestApp,PrivateEndpoint,Bastion,Firewall azure
    class SecurityGroup,NSG1,NSG2,NSG3 security
    class ODSAService,ADB,ORDS,VectorDB odsa
    class OCIBackend,GenAI oci
```

### 4.2. Traffic Flow Patterns

```mermaid
graph LR
    subgraph UserTraffic["User Traffic"]
        Browser["Web Browser"]
    end
    
    subgraph EdgeSecurity["Edge Security"]
        WAF["Web Application Firewall<br/>(Optional)"]
        DDoS["DDoS Protection"]
    end
    
    subgraph LoadBalancing["Load Balancing"]
        ALB["Application Load Balancer<br/>(Built-in)"]
    end
    
    subgraph Application["Application Tier"]
        App1["Siebel/Streamlit<br/>Instance 1"]
        App2["Siebel/Streamlit<br/>Instance 2"]
    end
    
    subgraph Network["Network Layer"]
        VNet["Azure VNet<br/>(10.0.0.0/16)"]
        NSG["Network Security Groups"]
        Route["Route Tables"]
    end
    
    subgraph PrivateLink["Private Connectivity"]
        Endpoint["Private Endpoint"]
        DNS["Private DNS Zone"]
    end
    
    subgraph Backend["Backend Services"]
        ADB["Oracle 23ai Database<br/>(via ODSA)"]
    end
    
    Browser -->|HTTPS| WAF
    WAF --> DDoS
    DDoS --> ALB
    
    ALB -->|Round Robin| App1
    ALB -->|Round Robin| App2
    
    App1 --> VNet
    App2 --> VNet
    
    VNet --> NSG
    NSG --> Route
    
    Route --> Endpoint
    Endpoint --> DNS
    DNS --> ADB
    
    classDef user fill:#5E5E5E,stroke:#2E2E2E,color:#fff
    classDef security fill:#107C10,stroke:#094509,color:#fff
    classDef lb fill:#0078D4,stroke:#004578,color:#fff
    classDef app fill:#68B984,stroke:#3D7A52,color:#fff
    classDef network fill:#FFB900,stroke:#C79400,color:#000
    classDef private fill:#336791,stroke:#1E3A5F,color:#fff
    classDef backend fill:#C74634,stroke:#8B2F23,color:#fff
    
    class UserTraffic,Browser user
    class EdgeSecurity,WAF,DDoS security
    class LoadBalancing,ALB lb
    class Application,App1,App2 app
    class Network,VNet,NSG,Route network
    class PrivateLink,Endpoint,DNS private
    class Backend,ADB backend
```

---

## 5. Security Architecture

### 5.1. Defense in Depth

```mermaid
graph TB
    subgraph Layer7["Layer 7: Monitoring & Response"]
        SIEM["Azure Sentinel<br/>(Security Information)"]
        Threat["Threat Intelligence"]
        SOC["Security Operations"]
    end
    
    subgraph Layer6["Layer 6: Application Security"]
        AppAuth["Application Authentication"]
        WAF["Web Application Firewall"]
        InputVal["Input Validation"]
    end
    
    subgraph Layer5["Layer 5: Data Security"]
        TDE["Transparent Data Encryption"]
        Masking["Data Masking"]
        Backup["Encrypted Backups"]
    end
    
    subgraph Layer4["Layer 4: Identity & Access"]
        AAD["Azure Active Directory"]
        RBAC["Role-Based Access Control"]
        MFA["Multi-Factor Authentication"]
    end
    
    subgraph Layer3["Layer 3: Platform Security"]
        PatchMgmt["Automatic Patching<br/>(ADB Managed)"]
        Compliance["Compliance Standards<br/>(SOC 2, ISO 27001)"]
        KeyVault["Azure Key Vault"]
    end
    
    subgraph Layer2["Layer 2: Network Security"]
        NSG["Network Security Groups"]
        PrivateLink["Private Endpoints"]
        NoPublic["No Public Access"]
    end
    
    subgraph Layer1["Layer 1: Physical Security"]
        Datacenter["Azure/OCI Datacenters"]
        Physical["Physical Access Controls"]
        Environmental["Environmental Controls"]
    end
    
    Attack["Potential Attack"] --> Layer7
    Layer7 -->|Monitor| Layer6
    Layer6 -->|Protect| Layer5
    Layer5 -->|Secure| Layer4
    Layer4 -->|Control| Layer3
    Layer3 -->|Harden| Layer2
    Layer2 -->|Isolate| Layer1
    Layer1 -->|Foundation| Protected["Protected Assets"]
    
    classDef monitor fill:#5E5E5E,stroke:#2E2E2E,color:#fff
    classDef app fill:#68B984,stroke:#3D7A52,color:#fff
    classDef data fill:#C74634,stroke:#8B2F23,color:#fff
    classDef identity fill:#FFB900,stroke:#C79400,color:#000
    classDef platform fill:#107C10,stroke:#094509,color:#fff
    classDef network fill:#0078D4,stroke:#004578,color:#fff
    classDef physical fill:#336791,stroke:#1E3A5F,color:#fff
    
    class Layer7,SIEM,Threat,SOC monitor
    class Layer6,AppAuth,WAF,InputVal app
    class Layer5,TDE,Masking,Backup data
    class Layer4,AAD,RBAC,MFA identity
    class Layer3,PatchMgmt,Compliance,KeyVault platform
    class Layer2,NSG,PrivateLink,NoPublic network
    class Layer1,Datacenter,Physical,Environmental physical
```

### 5.2. Authentication & Authorization Flow

```mermaid
sequenceDiagram
    participant Client as Siebel/Streamlit Client
    participant Gateway as API Gateway<br/>(Optional)
    participant KeyVault as Azure Key Vault
    participant ORDS as ORDS Endpoint
    participant OAuth as OAuth2 Server<br/>(Optional)
    participant DB as Oracle 23ai Database
    participant Audit as Audit Log
    
    Note over Client,Audit: Option 1: API Key Authentication
    
    Client->>KeyVault: Retrieve API Key
    KeyVault-->>Client: Return encrypted key
    
    Client->>Gateway: HTTPS Request<br/>Header: X-API-Key
    Gateway->>Gateway: Validate API Key
    Gateway->>ORDS: Forward request
    ORDS->>DB: Execute query
    DB-->>ORDS: Return results
    ORDS-->>Gateway: JSON response
    Gateway-->>Client: JSON response
    
    Gateway->>Audit: Log request details
    
    Note over Client,Audit: Option 2: OAuth2 Token
    
    Client->>OAuth: Request token<br/>(client_id, client_secret)
    OAuth-->>Client: Return JWT token
    
    Client->>ORDS: HTTPS Request<br/>Header: Authorization Bearer
    ORDS->>ORDS: Validate JWT signature
    ORDS->>ORDS: Check token expiration
    ORDS->>ORDS: Verify scopes
    ORDS->>DB: Execute query
    DB-->>ORDS: Return results
    ORDS-->>Client: JSON response
    
    ORDS->>Audit: Log request details
```

---

## 6. Deployment Architecture

*(This section references the detailed deployment diagrams in the main Project Architecture document)*

### 6.1. CI/CD Pipeline

```mermaid
graph LR
    subgraph Source["Source Control"]
        Git["GitHub Repository<br/>Main Branch"]
        PR["Pull Request<br/>Feature Branch"]
    end
    
    subgraph Build["Build Stage"]
        Checkout["Checkout Code"]
        Lint["Linting<br/>(ESLint, Pylint)"]
        Test["Unit Tests<br/>(Jest, Pytest)"]
        Scan["Security Scan<br/>(Trivy, Snyk)"]
    end
    
    subgraph Artifact["Artifact Stage"]
        DockerBuild["Build Container<br/>(Docker)"]
        Push["Push to ACR"]
        Tag["Tag Version<br/>(Semantic)"]
    end
    
    subgraph Deploy["Deploy Stage"]
        Dev["Deploy to Dev<br/>(Auto)"]
        QA["Deploy to QA<br/>(Manual Approval)"]
        Prod["Deploy to Prod<br/>(Manual Approval)"]
    end
    
    subgraph Monitor["Monitoring"]
        AppInsights["Application Insights"]
        LogAnalytics["Log Analytics"]
        Alerts["Alert Rules"]
    end
    
    PR --> Git
    Git -->|Trigger| Checkout
    Checkout --> Lint
    Lint --> Test
    Test --> Scan
    
    Scan -->|Pass| DockerBuild
    DockerBuild --> Push
    Push --> Tag
    
    Tag --> Dev
    Dev -->|Tests Pass| QA
    QA -->|Approved| Prod
    
    Dev -.->|Metrics| AppInsights
    QA -.->|Metrics| AppInsights
    Prod -.->|Metrics| AppInsights
    
    AppInsights --> LogAnalytics
    LogAnalytics --> Alerts
    
    classDef source fill:#24292E,stroke:#1B1F23,color:#fff
    classDef build fill:#2088FF,stroke:#1366D6,color:#fff
    classDef artifact fill:#68B984,stroke:#3D7A52,color:#fff
    classDef deploy fill:#FFB900,stroke:#C79400,color:#000
    classDef monitor fill:#C74634,stroke:#8B2F23,color:#fff
    
    class Source,Git,PR source
    class Build,Checkout,Lint,Test,Scan build
    class Artifact,DockerBuild,Push,Tag artifact
    class Deploy,Dev,QA,Prod deploy
    class Monitor,AppInsights,LogAnalytics,Alerts monitor
```

---

## 7. Vector Search Architecture

### 7.1. Vector Embedding Process

```mermaid
graph TB
    subgraph Input["Text Input"]
        Query["User Query Text<br/>'My laptop WiFi is not working'"]
        Historical["Historical Service Request<br/>'SR-12345: Network connectivity issue...'"]
    end
    
    subgraph Preprocessing["Text Preprocessing"]
        Clean["Clean Text<br/>- Remove HTML<br/>- Normalize spaces<br/>- Lowercase"]
        Token["Tokenization<br/>- Word boundaries<br/>- Special chars"]
    end
    
    subgraph Embedding["Embedding Service"]
        API["Azure AI Foundry API<br/>(OpenAI text-embedding-3-small)"]
        Model["Neural Network Model<br/>- Input: Text tokens<br/>- Output: 1024-dim vector"]
    end
    
    subgraph Vector["Vector Representation"]
        Embed["Embedding Vector<br/>[0.123, -0.456, 0.789, ...]<br/>1024 floating-point numbers"]
        Norm["Normalization<br/>L2 norm = 1.0"]
    end
    
    subgraph Storage["Storage"]
        Column["VECTOR Data Type<br/>Oracle AI Vector Search"]
        Index["HNSW Index<br/>Fast similarity search"]
    end
    
    Query --> Clean
    Historical --> Clean
    
    Clean --> Token
    Token --> API
    API --> Model
    Model --> Embed
    Embed --> Norm
    Norm --> Column
    Column --> Index
    
    classDef input fill:#FFB900,stroke:#C79400,color:#000
    classDef prep fill:#68B984,stroke:#3D7A52,color:#fff
    classDef ai fill:#C74634,stroke:#8B2F23,color:#fff
    classDef vector fill:#0078D4,stroke:#004578,color:#fff
    classDef storage fill:#336791,stroke:#1E3A5F,color:#fff
    
    class Input,Query,Historical input
    class Preprocessing,Clean,Token prep
    class Embedding,API,Model ai
    class Vector,Embed,Norm vector
    class Storage,Column,Index storage
```

### 7.2. HNSW Index Structure

```mermaid
graph TB
    subgraph TopLayer["Layer 3 (Top Layer) - Sparse"]
        L3_1["Node A"]
        L3_2["Node B"]
        L3_3["Node C"]
    end
    
    subgraph MidLayer["Layer 2 - Medium Density"]
        L2_1["Node D"]
        L2_2["Node E"]
        L2_3["Node F"]
        L2_4["Node G"]
        L2_5["Node H"]
        L2_6["Node I"]
    end
    
    subgraph LowLayer["Layer 1 - Dense"]
        L1_1["Node J"]
        L1_2["Node K"]
        L1_3["Node L"]
        L1_4["Node M"]
        L1_5["Node N"]
        L1_6["Node O"]
        L1_7["Node P"]
        L1_8["Node Q"]
        L1_9["Node R"]
    end
    
    subgraph BaseLayer["Layer 0 (Base Layer) - All Vectors"]
        L0_1["Vector 1"]
        L0_2["Vector 2"]
        L0_3["Vector 3"]
        L0_4["Vector 4"]
        L0_5["Vector 5"]
        L0_6["Vector 6"]
        L0_7["..."]
        L0_N["Vector N<br/>(Millions)"]
    end
    
    Query["Query Vector<br/>(Entry Point)"] --> L3_1
    
    L3_1 -->|Long-range link| L3_2
    L3_2 -->|Long-range link| L3_3
    
    L3_1 --> L2_1
    L3_2 --> L2_3
    L3_3 --> L2_6
    
    L2_1 --> L2_2
    L2_2 --> L2_3
    L2_3 --> L2_4
    L2_4 --> L2_5
    L2_5 --> L2_6
    
    L2_1 --> L1_1
    L2_2 --> L1_3
    L2_3 --> L1_4
    L2_4 --> L1_6
    L2_5 --> L1_7
    L2_6 --> L1_9
    
    L1_1 --> L1_2
    L1_2 --> L1_3
    L1_3 --> L1_4
    L1_4 --> L1_5
    L1_5 --> L1_6
    L1_6 --> L1_7
    L1_7 --> L1_8
    L1_8 --> L1_9
    
    L1_1 --> L0_1
    L1_2 --> L0_2
    L1_3 --> L0_2
    L1_4 --> L0_3
    L1_5 --> L0_4
    L1_6 --> L0_4
    L1_7 --> L0_5
    L1_8 --> L0_6
    L1_9 --> L0_6
    
    L0_1 --> L0_2
    L0_2 --> L0_3
    L0_3 --> L0_4
    L0_4 --> L0_5
    L0_5 --> L0_6
    L0_6 --> L0_7
    L0_7 --> L0_N
    
    L0_3 -.->|Similar vectors| Result["Top 100<br/>Similar Vectors"]
    
    classDef layer3 fill:#C74634,stroke:#8B2F23,color:#fff
    classDef layer2 fill:#FFB900,stroke:#C79400,color:#000
    classDef layer1 fill:#68B984,stroke:#3D7A52,color:#fff
    classDef layer0 fill:#0078D4,stroke:#004578,color:#fff
    classDef query fill:#107C10,stroke:#094509,color:#fff
    
    class TopLayer,L3_1,L3_2,L3_3 layer3
    class MidLayer,L2_1,L2_2,L2_3,L2_4,L2_5,L2_6 layer2
    class LowLayer,L1_1,L1_2,L1_3,L1_4,L1_5,L1_6,L1_7,L1_8,L1_9 layer1
    class BaseLayer,L0_1,L0_2,L0_3,L0_4,L0_5,L0_6,L0_7,L0_N layer0
    class Query,Result query
```

### 7.3. Similarity Search Process

```mermaid
sequenceDiagram
    participant App as Application
    participant DB as Oracle 23ai Database
    participant HNSW as HNSW Index
    participant Calc as Distance Calculator
    participant Rank as Ranking Engine
    
    Note over App,Rank: Vector Similarity Search Flow
    
    App->>DB: Query with vector embedding<br/>[0.123, -0.456, ..., 0.789]
    DB->>HNSW: Enter at top layer
    
    loop Navigate layers (top to bottom)
        HNSW->>Calc: Calculate distance to neighbors
        Calc-->>HNSW: Return distances
        HNSW->>HNSW: Move to closest neighbor
        HNSW->>HNSW: Descend to next layer
    end
    
    HNSW->>Calc: Calculate exact distances<br/>at base layer
    Calc-->>HNSW: Return all distances
    
    HNSW->>Rank: Send candidates with distances
    Rank->>Rank: Sort by similarity score
    Rank->>Rank: Apply filters (if any)
    Rank->>Rank: Select top K results
    
    Rank-->>DB: Top K similar vectors<br/>with metadata
    DB-->>App: Query results with<br/>similarity scores
    
    Note over App,Rank: Search Time: ~10-50ms for millions of vectors
```

---

## 8. Integration Architecture

### 8.1. Siebel CRM Integration

```mermaid
graph TB
    subgraph SiebelUI["Siebel Open UI Layer"]
        PM["Custom Presentation Model<br/>(JavaScript)"]
        Applet["Search Catalog Applet<br/>(Modified)"]
    end
    
    subgraph SiebelBusiness["Siebel Business Layer"]
        BS["SemanticSearchAPIService<br/>(Business Service)"]
        NS["Named Subsystem<br/>(HTTP Handler)"]
        Workflow["Workflow Process<br/>(Optional)"]
    end
    
    subgraph SiebelData["Siebel Data Layer"]
        Catalog["S_SVC_PROD Table<br/>(Catalog Items)"]
        Config["S_APP_CFG Table<br/>(API Configuration)"]
    end
    
    subgraph Integration["Integration Middleware"]
        HTTPClient["HTTP Client<br/>(eScript)"]
        JSONParser["JSON Parser"]
        ErrorHandler["Error Handler"]
    end
    
    subgraph External["External Services"]
        ORDS["ORDS API<br/>(Autonomous DB)"]
    end
    
    User["User"] -->|1. Enter query| PM
    PM -->|2. Invoke| BS
    BS -->|3. Read config| Config
    BS -->|4. Call| NS
    NS -->|5. Execute| HTTPClient
    
    HTTPClient -->|6. HTTPS POST| ORDS
    ORDS -->|7. JSON response| HTTPClient
    
    HTTPClient -->|8. Parse| JSONParser
    JSONParser -->|9. Return| NS
    NS -->|10. Return| BS
    
    BS -->|11. Lookup| Catalog
    Catalog -->|12. Return details| BS
    
    BS -->|13. Format| PM
    PM -->|14. Display| Applet
    Applet -->|15. Render| User
    
    HTTPClient -.->|On error| ErrorHandler
    ErrorHandler -.->|Fallback| Workflow
    
    classDef ui fill:#68B984,stroke:#3D7A52,color:#fff
    classDef business fill:#FFB900,stroke:#C79400,color:#000
    classDef data fill:#336791,stroke:#1E3A5F,color:#fff
    classDef integration fill:#0078D4,stroke:#004578,color:#fff
    classDef external fill:#C74634,stroke:#8B2F23,color:#fff
    
    class SiebelUI,PM,Applet ui
    class SiebelBusiness,BS,NS,Workflow business
    class SiebelData,Catalog,Config data
    class Integration,HTTPClient,JSONParser,ErrorHandler integration
    class External,ORDS external
```

### 8.2. System Integration Points

```mermaid
graph TB
    subgraph Sources["Data Sources"]
        S1["Siebel Oracle 19c<br/>(Primary CRM Data)"]
        S2["Email Server<br/>(Activity History)"]
        S3["External Systems<br/>(Optional)"]
    end
    
    subgraph Core["Core Platform (Autonomous DB)"]
        ETL["ETL Pipeline<br/>(Database Links)"]
        Vector["Vector Store<br/>(AI Search)"]
        API["ORDS API<br/>(REST Endpoints)"]
    end
    
    subgraph Consumers["Data Consumers"]
        C1["Siebel CRM<br/>(Production UI)"]
        C2["Streamlit App<br/>(Test & QA)"]
        C3["BI Tools<br/>(Analytics)"]
        C4["Future Integrations<br/>(Chat, Mobile)"]
    end
    
    subgraph External["External AI Services"]
        E1["Azure AI Foundry<br/>(Embeddings)"]
        E2["Azure OpenAI<br/>(Future: LLM)"]
    end
    
    S1 -->|Database Link| ETL
    S2 -.->|Optional| ETL
    S3 -.->|Optional| ETL
    
    ETL -->|Load| Vector
    Vector -->|Expose| API
    
    Vector -->|Call| E1
    Vector -.->|Future| E2
    
    API -->|Serve| C1
    API -->|Serve| C2
    API -.->|Serve| C3
    API -.->|Future| C4
    
    classDef source fill:#FFB900,stroke:#C79400,color:#000
    classDef core fill:#0078D4,stroke:#004578,color:#fff
    classDef consumer fill:#68B984,stroke:#3D7A52,color:#fff
    classDef external fill:#C74634,stroke:#8B2F23,color:#fff
    
    class Sources,S1,S2,S3 source
    class Core,ETL,Vector,API core
    class Consumers,C1,C2,C3,C4 consumer
    class External,E1,E2 external
```

---

## Summary

This document provides comprehensive visual documentation of the semantic search solution architecture, covering:

- **System Context**: Overall system boundaries and major components
- **Component Architecture**: Detailed breakdown of all system components and their interactions
- **Data Flows**: Real-time search and batch indexing processes
- **Network Architecture**: Azure and OCI networking topology with private connectivity
- **Security Architecture**: Multi-layered defense-in-depth security model
- **Deployment Architecture**: CI/CD pipeline and container deployment patterns
- **Vector Search Architecture**: HNSW index structure and similarity search algorithms
- **Integration Architecture**: Siebel CRM integration and external system connections

All diagrams are created using Mermaid syntax and can be rendered in:
- GitHub (native support)
- VS Code (with Mermaid extension)
- Markdown viewers (with Mermaid plugin)
- Documentation platforms (Confluence, GitBook, etc.)

For implementation details, refer to the Technical Design Documents (TDD 1-5).






