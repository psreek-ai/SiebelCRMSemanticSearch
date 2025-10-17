# Azure AI Foundry Integration Guide

**Version:** 3.0  
**Date:** October 17, 2025  
**Purpose:** Comprehensive guide for integrating Azure AI Foundry with Oracle Database 23ai on Azure VM

---

## Table of Contents

1. [Overview](#1-overview)
2. [Why Azure AI Foundry](#2-why-azure-ai-foundry)
3. [Architecture](#3-architecture)
4. [Azure AI Foundry Setup](#4-azure-ai-foundry-setup)
5. [Oracle 23ai VM Integration](#5-oracle-23ai-vm-integration)
6. [PL/SQL Implementation](#6-plsql-implementation)
7. [Prompt Flow Integration](#7-prompt-flow-integration)
8. [Performance Optimization](#8-performance-optimization)
9. [Cost Analysis](#9-cost-analysis)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Overview

This solution uses **Azure AI Foundry** (formerly Azure AI Studio) as a unified AI development platform for generating text embeddings with OpenAI Service. Azure AI Foundry provides a comprehensive environment that integrates seamlessly with **Oracle Database 23ai on Azure VM**, creating a 100% Azure-native architecture with advanced AI capabilities.

### Key Benefits

✅ **Unified AI Platform** - Single workspace for all AI operations (models, prompt flows, evaluations)  
✅ **Enterprise-Grade Governance** - Built-in responsible AI, content filtering, and compliance  
✅ **Advanced Prompt Engineering** - Visual prompt flow designer with debugging tools  
✅ **Model Flexibility** - Easy switching between OpenAI models (GPT-4, embeddings)  
✅ **Cost Optimization** - $0.02 per million tokens (text-embedding-3-small)  
✅ **High Performance** - 1536-dimensional embeddings with private endpoint connectivity  
✅ **Integrated Monitoring** - Built-in logging, metrics, and model performance tracking  
✅ **Azure-Native Security** - Managed identity, Key Vault, Private Link, VNet integration  

### What is Azure AI Foundry?

**Azure AI Foundry** is Microsoft's unified platform for building, deploying, and managing AI applications. It combines:

- **AI Model Catalog**: Access to Azure OpenAI, open-source models, and custom models
- **Prompt Flow**: Visual designer for LLM application orchestration
- **AI Search Integration**: Vector search and hybrid search capabilities
- **Content Safety**: Built-in content filtering and responsible AI guardrails
- **Model Evaluation**: A/B testing, benchmarking, and quality metrics
- **Deployment Options**: Managed endpoints, serverless, or real-time inference
- **MLOps Integration**: CI/CD pipelines, version control, model registry

---

## 2. Why Azure AI Foundry

### Azure AI Foundry vs. Direct Azure OpenAI Service

| Feature | Azure AI Foundry | Direct Azure OpenAI |
|---------|------------------|---------------------|
| **Deployment Model** | Workspace-based with projects | Resource-based |
| **Prompt Engineering** | Visual Prompt Flow designer | Manual code |
| **Model Evaluation** | Built-in A/B testing & metrics | Manual implementation |
| **Content Filtering** | Configurable per deployment | Basic global settings |
| **Monitoring** | Unified dashboard (tokens, latency, errors) | Separate Azure Monitor |
| **RAG Patterns** | Pre-built templates | Build from scratch |
| **Multi-Model Support** | OpenAI + OSS + Custom | OpenAI only |
| **Experimentation** | Built-in experiment tracking | External MLflow |
| **Cost Management** | Workspace-level quotas | Per-resource limits |
| **Team Collaboration** | Shared workspace with RBAC | Individual resource access |

### Key Advantages for Our Use Case

1. **Vector Search Optimization**
   - Pre-configured embedding pipelines
   - Automatic batching and retry logic
   - Performance profiling tools

2. **Responsible AI**
   - Content safety filters for queries
   - Bias detection in embeddings
   - Compliance reporting

3. **Operational Excellence**
   - Centralized logging of all API calls
   - Cost tracking per project
   - Alert configuration for anomalies

4. **Future-Proof Architecture**
   - Easy addition of GPT-4 for narrative generation
   - Integration with Azure AI Search for hybrid search
   - Support for custom fine-tuned models

### Cost Comparison (Annual)

**Scenario:** 1 million service requests indexed, 10,000 queries per month

```
Azure AI Foundry (text-embedding-3-small):
├── Initial Indexing: 1M records × 200 tokens avg = 200M tokens
│   Cost: 200M × $0.02 / 1M = $4.00
├── Monthly Queries: 10K queries × 50 tokens avg = 500K tokens
│   Cost: 0.5M × $0.02 / 1M = $0.01/month = $0.12/year
├── Annual Re-indexing (delta): ~50M tokens
│   Cost: 50M × $0.02 / 1M = $1.00/year
├── Workspace Costs: Included (no additional charge)
└── Total Annual Cost: $5.12

Additional Value at No Extra Cost:
✅ Prompt Flow designer
✅ Model evaluation tools
✅ Content safety filtering
✅ Unified monitoring dashboard
✅ Experiment tracking
```

**Note:** Azure AI Foundry workspace itself has no additional cost. You only pay for the underlying compute resources (OpenAI API calls, deployments, etc.).

---

## 3. Architecture

### 3.1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Azure Subscription                      │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Azure AI Foundry Workspace                 │ │
│  │                                                          │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │           Semantic Search Project             │  │ │
│  │  │                                                  │  │ │
│  │  │  ┌─────────────────────────────────────────┐   │  │ │
│  │  │  │   OpenAI Deployment                     │   │  │ │
│  │  │  │   • Model: text-embedding-3-small       │   │  │ │
│  │  │  │   • Dimensions: 1536                    │   │  │ │
│  │  │  │   • TPM: 120,000                        │   │  │ │
│  │  │  │   • Content Filter: Moderate            │   │  │ │
│  │  │  └─────────────────────────────────────────┘   │  │ │
│  │  │                                                  │  │ │
│  │  │  ┌─────────────────────────────────────────┐   │  │ │
│  │  │  │   Prompt Flow (Future)                  │   │  │ │
│  │  │  │   • Query preprocessing                 │   │  │ │
│  │  │  │   • Embedding generation                │   │  │ │
│  │  │  │   • Result ranking                      │   │  │ │
│  │  │  └─────────────────────────────────────────┘   │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  │                                                          │ │
│  │  Private Endpoint: workspace-pe.eastus.api.azureml.ms   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Azure Virtual Network                      │ │
│  │                                                          │ │
│  │  ┌────────────────────┐    ┌──────────────────────┐   │ │
│  │  │  Database Subnet   │    │  AI Foundry Subnet   │   │ │
│  │  │                    │────│                      │   │ │
│  │  │  Oracle 23ai VM    │    │  Private Endpoint    │   │ │
│  │  │  • VECSRCH DB      │    │  (AI Foundry)        │   │ │
│  │  │  • ORDS :8080      │    │                      │   │ │
│  │  │  • Semantic Search │    │                      │   │ │
│  │  └────────────────────┘    └──────────────────────┘   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Azure Key Vault                            │ │
│  │  • AI Foundry API Key                                   │ │
│  │  • Database credentials                                 │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 3.2. Data Flow

```
1. Oracle 23ai VM (PL/SQL Procedure)
   ↓
2. HTTPS Request to Azure AI Foundry Private Endpoint
   • URL: https://<workspace-name>.<region>.api.azureml.ms/
   • Method: POST
   • Headers: api-key, Content-Type
   • Body: {"input": "narrative text", "model": "text-embedding-3-small"}
   ↓
3. Azure AI Foundry Workspace
   • Routes to correct project
   • Applies content filtering
   • Logs request metadata
   ↓
4. OpenAI Deployment
   • Generates 1536-dimensional embedding
   • Returns JSON response
   ↓
5. Oracle 23ai (PL/SQL)
   • Parses JSON response
   • Extracts embedding vector
   • Inserts into SIEBEL_KNOWLEDGE_VECTORS
   ↓
6. Vector Index (HNSW)
   • Updates index automatically
   • Optimizes for cosine similarity search
```

### 3.3. Network Architecture

```
Oracle 23ai VM → Private Endpoint → AI Foundry Workspace → OpenAI Service
                    (VNet)              (Azure Backbone)
```

**Security Layers:**
1. **Network ACLs**: Oracle 23ai configured to allow HTTPS to `*.api.azureml.ms`
2. **Private Endpoint**: All traffic stays within Azure VNet
3. **Managed Identity**: (Future) No API keys in database
4. **Key Vault Integration**: API keys stored securely
5. **Content Filtering**: Configurable safety policies

---

## 4. Azure AI Foundry Setup

### 4.1. Prerequisites

- Azure Subscription with Owner or Contributor role
- Azure CLI installed (`az --version`)
- Resource Group for AI resources
- Virtual Network with subnets for private endpoints

### 4.2. Create Azure AI Foundry Workspace

**Option 1: Azure Portal (Recommended for First-Time)**

1. Navigate to Azure Portal → Search "Azure AI Foundry" (or "Azure AI Studio")
2. Click **+ Create**
3. Fill in workspace details:
   ```
   Subscription: <your-subscription>
   Resource Group: semantic-search-rg
   Workspace Name: siebel-semantic-search-ws
   Region: East US (or your preferred region)
   Storage Account: Create new (auto-generated)
   Key Vault: Create new (auto-generated)
   Application Insights: Create new (optional but recommended)
   ```
4. Click **Review + Create** → **Create**
5. Wait 3-5 minutes for deployment

**Option 2: Azure CLI**

```bash
# Set variables
RESOURCE_GROUP="semantic-search-rg"
LOCATION="eastus"
WORKSPACE_NAME="siebel-semantic-search-ws"
STORAGE_ACCOUNT="semanticsearchsa$(date +%s)"
KEY_VAULT_NAME="semantic-search-kv"

# Create AI Foundry workspace
az ml workspace create \
  --name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --storage-account $STORAGE_ACCOUNT \
  --key-vault $KEY_VAULT_NAME \
  --application-insights semantic-search-appinsights \
  --container-registry semantic-search-acr

# Verify workspace
az ml workspace show \
  --name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP
```

### 4.3. Create AI Foundry Project

Projects organize work within a workspace (models, datasets, experiments).

```bash
# Create project
az ml project create \
  --name siebel-semantic-search-project \
  --workspace-name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --description "Semantic search for Siebel CRM service requests"

# Or via Portal:
# AI Foundry → Workspaces → siebel-semantic-search-ws → + New Project
```

### 4.4. Deploy OpenAI Model

**Via Azure Portal:**

1. Open AI Foundry workspace
2. Navigate to **Deployments** → **+ Create Deployment**
3. Select model:
   ```
   Model Type: Azure OpenAI
   Model: text-embedding-3-small
   Deployment Name: text-embedding-3-small
   Version: Latest
   ```
4. Configure deployment:
   ```
   Tokens Per Minute (TPM) Rate Limit: 120,000
   Content Filter: Standard (Moderate)
   Enable Dynamic Quota: Yes
   ```
5. Click **Deploy**

**Via Azure CLI:**

```bash
# Create Azure OpenAI resource (if not exists)
az cognitiveservices account create \
  --name siebel-openai-service \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --kind OpenAI \
  --sku S0

# Deploy embedding model
az cognitiveservices account deployment create \
  --name siebel-openai-service \
  --resource-group $RESOURCE_GROUP \
  --deployment-name text-embedding-3-small \
  --model-name text-embedding-3-small \
  --model-version "1" \
  --model-format OpenAI \
  --sku-capacity 120 \
  --sku-name "Standard"

# Verify deployment
az cognitiveservices account deployment show \
  --name siebel-openai-service \
  --resource-group $RESOURCE_GROUP \
  --deployment-name text-embedding-3-small
```

### 4.5. Configure Private Endpoint

**Why Private Endpoint?**
- All traffic between Oracle 23ai VM and AI Foundry stays within Azure VNet
- No exposure to public internet
- Lower latency (<10ms typically)
- Enhanced security and compliance

**Create Private Endpoint:**

```bash
# Get AI Foundry workspace resource ID
WORKSPACE_ID=$(az ml workspace show \
  --name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Create private endpoint subnet (if not exists)
az network vnet subnet create \
  --name ai-foundry-pe-subnet \
  --resource-group $RESOURCE_GROUP \
  --vnet-name semantic-search-vnet \
  --address-prefixes 10.0.3.0/24 \
  --disable-private-endpoint-network-policies

# Create private endpoint
az network private-endpoint create \
  --name ai-foundry-private-endpoint \
  --resource-group $RESOURCE_GROUP \
  --vnet-name semantic-search-vnet \
  --subnet ai-foundry-pe-subnet \
  --private-connection-resource-id $WORKSPACE_ID \
  --group-id amlworkspace \
  --connection-name ai-foundry-connection

# Create private DNS zone
az network private-dns zone create \
  --name privatelink.api.azureml.ms \
  --resource-group $RESOURCE_GROUP

# Link DNS zone to VNet
az network private-dns link vnet create \
  --name ai-foundry-dns-link \
  --resource-group $RESOURCE_GROUP \
  --zone-name privatelink.api.azureml.ms \
  --virtual-network semantic-search-vnet \
  --registration-enabled false

# Create DNS zone group
az network private-endpoint dns-zone-group create \
  --name ai-foundry-dns-group \
  --resource-group $RESOURCE_GROUP \
  --endpoint-name ai-foundry-private-endpoint \
  --private-dns-zone privatelink.api.azureml.ms \
  --zone-name privatelink.api.azureml.ms
```

### 4.6. Get API Credentials

```bash
# Get workspace endpoint
WORKSPACE_ENDPOINT=$(az ml workspace show \
  --name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query discovery_url -o tsv)

echo "Workspace Endpoint: $WORKSPACE_ENDPOINT"

# Get API key (for workspace authentication)
# Navigate to Azure Portal → AI Foundry → Keys and Endpoint
# Or use Azure OpenAI keys:

OPENAI_KEY=$(az cognitiveservices account keys list \
  --name siebel-openai-service \
  --resource-group $RESOURCE_GROUP \
  --query key1 -o tsv)

# Store in Key Vault
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name ai-foundry-api-key \
  --value $OPENAI_KEY

echo "API Key stored in Key Vault: $KEY_VAULT_NAME/ai-foundry-api-key"
```

### 4.7. Configure Content Filtering

Azure AI Foundry includes responsible AI features:

```bash
# Configure content filter (via Azure Portal recommended)
# AI Foundry → Deployments → text-embedding-3-small → Content filters

# Recommended settings for our use case:
# - Hate: Medium (block)
# - Sexual: Medium (block)
# - Violence: Medium (block)
# - Self-harm: Medium (block)
```

**Content Filter Levels:**
- **Low**: Very permissive (production risk)
- **Medium**: Balanced (recommended)
- **High**: Very restrictive (may block legitimate queries)

---

## 5. Oracle 23ai VM Integration

### 5.1. Network Connectivity Setup

**Ensure Oracle 23ai VM can reach AI Foundry private endpoint:**

```bash
# SSH to Oracle 23ai VM
ssh oracle@<oracle-23ai-vm-ip>

# Test connectivity to AI Foundry (use private endpoint IP or DNS)
curl -v https://<workspace-name>.eastus.api.azureml.ms

# Expected: Connection successful (even if 401 Unauthorized - means network is OK)

# Test DNS resolution
nslookup <workspace-name>.eastus.api.azureml.ms

# Expected: Resolves to private IP (10.0.3.x)
```

### 5.2. Configure Network ACLs (Already Done in Deployment)

**Verify ACLs are configured:**

```sql
-- Connect to Oracle 23ai
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Check ACLs
SELECT host, lower_port, upper_port, acl
FROM dba_network_acls
WHERE host LIKE '%api.azureml.ms' OR host LIKE '%openai.azure.com';

-- Expected: ACLs for *.api.azureml.ms and *.openai.azure.com on port 443
```

If missing, create them (already in Deployment Guide Phase 2):

```sql
BEGIN
  -- Create ACL
  DBMS_NETWORK_ACL_ADMIN.CREATE_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    description  => 'ACL for Azure AI Foundry API access',
    principal    => 'SEMANTIC_SEARCH',
    is_grant     => TRUE,
    privilege    => 'connect'
  );
  
  -- Add resolve privilege
  DBMS_NETWORK_ACL_ADMIN.ADD_PRIVILEGE (
    acl          => 'azure_ai_foundry_acl.xml',
    principal    => 'SEMANTIC_SEARCH',
    is_grant     => TRUE,
    privilege    => 'resolve'
  );
  
  -- Assign to AI Foundry hosts
  DBMS_NETWORK_ACL_ADMIN.ASSIGN_ACL (
    acl          => 'azure_ai_foundry_acl.xml',
    host         => '*.api.azureml.ms',
    lower_port   => 443,
    upper_port   => 443
  );
  
  COMMIT;
END;
/
```

### 5.3. Create Database Credential

```sql
-- Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Create credential for Azure AI Foundry
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'AZURE_AI_FOUNDRY_CRED',
    username        => 'AZURE_OPENAI',  -- Placeholder (required by DBMS_CLOUD)
    password        => '<your-azure-ai-foundry-api-key>'
  );
END;
/

-- Verify credential
SELECT credential_name, username, enabled
FROM user_credentials
WHERE credential_name = 'AZURE_AI_FOUNDRY_CRED';

-- Expected: AZURE_AI_FOUNDRY_CRED | AZURE_OPENAI | ENABLED

COMMIT;
```

### 5.4. Test Connectivity and API Call

```sql
-- Test Azure AI Foundry connectivity
SET SERVEROUTPUT ON SIZE UNLIMITED;

DECLARE
    l_api_endpoint VARCHAR2(500) := 'https://<workspace-name>.<region>.api.azureml.ms/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-15-preview';
    l_request_body CLOB;
    l_response CLOB;
BEGIN
    -- Build JSON request
    l_request_body := '{
        "input": "Test connectivity to Azure AI Foundry",
        "model": "text-embedding-3-small"
    }';
    
    -- Make API call
    l_response := DBMS_CLOUD.SEND_REQUEST(
        credential_name => 'AZURE_AI_FOUNDRY_CRED',
        uri             => l_api_endpoint,
        method          => DBMS_CLOUD.METHOD_POST,
        body            => UTL_RAW.CAST_TO_RAW(l_request_body)
    );
    
    -- Check response
    DBMS_OUTPUT.PUT_LINE('Response length: ' || DBMS_LOB.GETLENGTH(l_response) || ' bytes');
    DBMS_OUTPUT.PUT_LINE('First 200 chars: ' || SUBSTR(l_response, 1, 200));
    
    -- Expected: JSON with "data" array containing 1536-dimensional vector
    IF l_response LIKE '%"data"%' AND l_response LIKE '%"embedding"%' THEN
        DBMS_OUTPUT.PUT_LINE('✓ Azure AI Foundry connectivity test PASSED');
    ELSE
        DBMS_OUTPUT.PUT_LINE('✗ Azure AI Foundry connectivity test FAILED');
        DBMS_OUTPUT.PUT_LINE('Response: ' || l_response);
    END IF;
    
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        RAISE;
END;
/

EXIT;
```

---

## 6. PL/SQL Implementation

### 6.1. Embedding Generation Function

Create a function that calls Azure AI Foundry to generate embeddings:

```sql
-- Connect as SEMANTIC_SEARCH user
sqlplus SEMANTIC_SEARCH/<password>@localhost:1521/VECSRCH

-- Create embedding generation function
CREATE OR REPLACE FUNCTION GENERATE_EMBEDDING_AI_FOUNDRY(
    p_text IN VARCHAR2,
    p_workspace_name IN VARCHAR2 DEFAULT '<your-workspace-name>',
    p_region IN VARCHAR2 DEFAULT 'eastus',
    p_deployment_name IN VARCHAR2 DEFAULT 'text-embedding-3-small'
) RETURN VECTOR
IS
    l_api_endpoint VARCHAR2(500);
    l_request_body CLOB;
    l_response CLOB;
    l_embedding_json CLOB;
    l_embedding_vector VECTOR(1536);
    l_vector_array DBMS_VECTOR.VECTOR_ARRAY_T;
    l_start_pos NUMBER;
    l_end_pos NUMBER;
BEGIN
    -- Construct API endpoint
    l_api_endpoint := 'https://' || p_workspace_name || '.' || p_region || 
                      '.api.azureml.ms/openai/deployments/' || p_deployment_name || 
                      '/embeddings?api-version=2024-02-15-preview';
    
    -- Build JSON request body
    l_request_body := JSON_OBJECT(
        'input' VALUE p_text,
        'model' VALUE 'text-embedding-3-small'
    );
    
    -- Call Azure AI Foundry API
    l_response := DBMS_CLOUD.SEND_REQUEST(
        credential_name => 'AZURE_AI_FOUNDRY_CRED',
        uri             => l_api_endpoint,
        method          => DBMS_CLOUD.METHOD_POST,
        body            => UTL_RAW.CAST_TO_RAW(l_request_body),
        headers         => JSON_OBJECT('Content-Type' VALUE 'application/json')
    );
    
    -- Parse JSON response to extract embedding array
    -- Response format: {"data":[{"embedding":[0.123, -0.456, ...]}]}
    SELECT JSON_VALUE(l_response, '$.data[0].embedding') 
    INTO l_embedding_json
    FROM DUAL;
    
    -- Convert JSON array to VECTOR type
    l_embedding_vector := TO_VECTOR(l_embedding_json, 1536, FLOAT32);
    
    RETURN l_embedding_vector;
    
EXCEPTION
    WHEN OTHERS THEN
        -- Log error for debugging
        DBMS_OUTPUT.PUT_LINE('Error in GENERATE_EMBEDDING_AI_FOUNDRY: ' || SQLERRM);
        DBMS_OUTPUT.PUT_LINE('API Endpoint: ' || l_api_endpoint);
        DBMS_OUTPUT.PUT_LINE('Input Text Length: ' || LENGTH(p_text));
        RAISE;
END GENERATE_EMBEDDING_AI_FOUNDRY;
/

-- Grant execute to necessary users
GRANT EXECUTE ON GENERATE_EMBEDDING_AI_FOUNDRY TO PUBLIC;
```

### 6.2. Batch Embedding Procedure

Process multiple narratives efficiently:

```sql
CREATE OR REPLACE PROCEDURE GENERATE_EMBEDDINGS_BATCH_AI_FOUNDRY(
    p_batch_size IN NUMBER DEFAULT 100,
    p_workspace_name IN VARCHAR2 DEFAULT '<your-workspace-name>',
    p_region IN VARCHAR2 DEFAULT 'eastus'
)
IS
    l_processed_count NUMBER := 0;
    l_error_count NUMBER := 0;
    l_start_time TIMESTAMP := SYSTIMESTAMP;
    l_embedding VECTOR(1536);
    
    CURSOR c_pending_narratives IS
        SELECT narrative_id, narrative_text
        FROM SIEBEL_NARRATIVES_STAGING
        WHERE processed_flag = 'N'
          AND ROWNUM <= p_batch_size
        FOR UPDATE SKIP LOCKED;
        
BEGIN
    DBMS_OUTPUT.PUT_LINE('Starting batch embedding generation via Azure AI Foundry');
    DBMS_OUTPUT.PUT_LINE('Batch size: ' || p_batch_size);
    DBMS_OUTPUT.PUT_LINE('Start time: ' || TO_CHAR(l_start_time, 'YYYY-MM-DD HH24:MI:SS'));
    
    FOR rec IN c_pending_narratives LOOP
        BEGIN
            -- Generate embedding via Azure AI Foundry
            l_embedding := GENERATE_EMBEDDING_AI_FOUNDRY(
                p_text => rec.narrative_text,
                p_workspace_name => p_workspace_name,
                p_region => p_region
            );
            
            -- Insert into vector table
            INSERT INTO SIEBEL_KNOWLEDGE_VECTORS (
                narrative_id,
                narrative_text,
                narrative_vector,
                created_date,
                ai_model,
                vector_dimensions
            ) VALUES (
                rec.narrative_id,
                rec.narrative_text,
                l_embedding,
                SYSDATE,
                'text-embedding-3-small (Azure AI Foundry)',
                1536
            );
            
            -- Mark as processed
            UPDATE SIEBEL_NARRATIVES_STAGING
            SET processed_flag = 'Y',
                processed_date = SYSDATE
            WHERE narrative_id = rec.narrative_id;
            
            l_processed_count := l_processed_count + 1;
            
            -- Commit every 10 records
            IF MOD(l_processed_count, 10) = 0 THEN
                COMMIT;
                DBMS_OUTPUT.PUT_LINE('Processed: ' || l_processed_count || ' narratives');
            END IF;
            
        EXCEPTION
            WHEN OTHERS THEN
                l_error_count := l_error_count + 1;
                DBMS_OUTPUT.PUT_LINE('Error processing narrative_id ' || rec.narrative_id || ': ' || SQLERRM);
                
                -- Mark as error
                UPDATE SIEBEL_NARRATIVES_STAGING
                SET processed_flag = 'E',
                    error_message = SUBSTR(SQLERRM, 1, 4000),
                    processed_date = SYSDATE
                WHERE narrative_id = rec.narrative_id;
                
                COMMIT;
        END;
    END LOOP;
    
    -- Final commit
    COMMIT;
    
    -- Summary
    DBMS_OUTPUT.PUT_LINE('========================================');
    DBMS_OUTPUT.PUT_LINE('Batch Embedding Generation Complete');
    DBMS_OUTPUT.PUT_LINE('Total Processed: ' || l_processed_count);
    DBMS_OUTPUT.PUT_LINE('Total Errors: ' || l_error_count);
    DBMS_OUTPUT.PUT_LINE('Duration: ' || EXTRACT(SECOND FROM (SYSTIMESTAMP - l_start_time)) || ' seconds');
    DBMS_OUTPUT.PUT_LINE('========================================');
    
END GENERATE_EMBEDDINGS_BATCH_AI_FOUNDRY;
/
```

### 6.3. Test Embedding Generation

```sql
-- Test single embedding generation
SET SERVEROUTPUT ON SIZE UNLIMITED;

DECLARE
    l_test_vector VECTOR(1536);
BEGIN
    l_test_vector := GENERATE_EMBEDDING_AI_FOUNDRY(
        p_text => 'Customer reports laptop overheating and shutting down randomly',
        p_workspace_name => '<your-workspace-name>',
        p_region => 'eastus'
    );
    
    DBMS_OUTPUT.PUT_LINE('✓ Embedding generated successfully');
    DBMS_OUTPUT.PUT_LINE('Vector dimensions: ' || l_test_vector.COUNT);
    DBMS_OUTPUT.PUT_LINE('First 3 values: ' || l_test_vector(1) || ', ' || l_test_vector(2) || ', ' || l_test_vector(3));
END;
/

-- Test batch processing (small batch)
BEGIN
    GENERATE_EMBEDDINGS_BATCH_AI_FOUNDRY(
        p_batch_size => 5,
        p_workspace_name => '<your-workspace-name>',
        p_region => 'eastus'
    );
END;
/

-- Verify results
SELECT COUNT(*) as total_vectors,
       AVG(vector_dimensions) as avg_dimensions,
       MIN(created_date) as first_created,
       MAX(created_date) as last_created
FROM SIEBEL_KNOWLEDGE_VECTORS;
```

---

## 7. Prompt Flow Integration

Azure AI Foundry's **Prompt Flow** provides a visual designer for orchestrating LLM workflows. While not required for basic embedding generation, it offers powerful capabilities for advanced scenarios.

### 7.1. Use Cases for Prompt Flow

1. **Query Preprocessing**
   - Clean and normalize user queries
   - Extract key entities (product names, error codes)
   - Expand abbreviations

2. **Multi-Step RAG**
   - Generate embedding → Vector search → Re-rank results → Generate summary

3. **Hybrid Search**
   - Combine semantic search with keyword filtering
   - Apply business rules (e.g., only show open tickets)

4. **A/B Testing**
   - Test different embedding models
   - Compare search quality metrics

### 7.2. Create Prompt Flow (Example)

**Scenario:** Preprocess user query before generating embedding

1. Open Azure AI Foundry → **Prompt Flow**
2. Click **+ Create**
3. Select template: **Standard Flow**
4. Add nodes:

```
┌────────────────┐
│  User Query    │ (Input)
└────────┬───────┘
         │
         ▼
┌────────────────┐
│  Clean Text    │ (Python)
│  • Remove HTML │
│  • Fix typos   │
│  • Normalize   │
└────────┬───────┘
         │
         ▼
┌────────────────┐
│  Generate      │ (LLM - text-embedding-3-small)
│  Embedding     │
└────────┬───────┘
         │
         ▼
┌────────────────┐
│  Vector        │ (Output)
└────────────────┘
```

**Python Node Example:**

```python
from promptflow import tool
import re

@tool
def clean_query(query: str) -> str:
    """Clean and normalize user query"""
    
    # Remove HTML tags
    query = re.sub(r'<[^>]+>', '', query)
    
    # Fix common typos
    replacements = {
        'cant': "can't",
        'wont': "won't",
        'doesnt': "doesn't",
        'isnt': "isn't"
    }
    for old, new in replacements.items():
        query = query.replace(old, new)
    
    # Normalize whitespace
    query = ' '.join(query.split())
    
    # Convert to lowercase
    query = query.lower()
    
    return query
```

### 7.3. Deploy Prompt Flow as Endpoint

Once tested, deploy as REST endpoint:

```bash
# Deploy prompt flow
az ml online-endpoint create \
  --name semantic-search-preprocessor \
  --workspace-name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP

az ml online-deployment create \
  --name blue \
  --endpoint semantic-search-preprocessor \
  --workspace-name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --file prompt-flow-deployment.yaml \
  --all-traffic
```

Then call from PL/SQL:

```sql
CREATE OR REPLACE FUNCTION PREPROCESS_QUERY(
    p_query IN VARCHAR2
) RETURN VARCHAR2
IS
    l_endpoint VARCHAR2(500) := 'https://<workspace-name>.<region>.api.azureml.ms/endpoints/semantic-search-preprocessor/score';
    l_request CLOB;
    l_response CLOB;
    l_cleaned_query VARCHAR2(4000);
BEGIN
    l_request := JSON_OBJECT('query' VALUE p_query);
    
    l_response := DBMS_CLOUD.SEND_REQUEST(
        credential_name => 'AZURE_AI_FOUNDRY_CRED',
        uri             => l_endpoint,
        method          => DBMS_CLOUD.METHOD_POST,
        body            => UTL_RAW.CAST_TO_RAW(l_request)
    );
    
    SELECT JSON_VALUE(l_response, '$.cleaned_query')
    INTO l_cleaned_query
    FROM DUAL;
    
    RETURN l_cleaned_query;
END;
/
```

---

## 8. Performance Optimization

### 8.1. Rate Limiting and Throttling

Azure AI Foundry enforces token-per-minute (TPM) limits:

```
text-embedding-3-small:
├── Default TPM: 120,000
├── Max batch size: ~600 texts per request (with avg 200 tokens each)
└── Recommended batch size: 100-200 for balance
```

**Optimization Strategy:**

```sql
-- Add delay between batches to avoid rate limiting
CREATE OR REPLACE PROCEDURE GENERATE_ALL_EMBEDDINGS
IS
    l_total_pending NUMBER;
    l_batch_count NUMBER := 0;
BEGIN
    -- Get total pending
    SELECT COUNT(*) INTO l_total_pending
    FROM SIEBEL_NARRATIVES_STAGING
    WHERE processed_flag = 'N';
    
    DBMS_OUTPUT.PUT_LINE('Total pending: ' || l_total_pending);
    
    -- Process in batches with delay
    WHILE l_total_pending > 0 LOOP
        l_batch_count := l_batch_count + 1;
        
        DBMS_OUTPUT.PUT_LINE('Processing batch ' || l_batch_count);
        
        -- Process batch of 100
        GENERATE_EMBEDDINGS_BATCH_AI_FOUNDRY(p_batch_size => 100);
        
        -- Check remaining
        SELECT COUNT(*) INTO l_total_pending
        FROM SIEBEL_NARRATIVES_STAGING
        WHERE processed_flag = 'N';
        
        -- If more batches needed, wait 30 seconds
        IF l_total_pending > 0 THEN
            DBMS_OUTPUT.PUT_LINE('Waiting 30 seconds before next batch...');
            DBMS_LOCK.SLEEP(30);
        END IF;
    END LOOP;
    
    DBMS_OUTPUT.PUT_LINE('All embeddings generated!');
END;
/
```

### 8.2. Caching Strategy

Cache embeddings to avoid regenerating for duplicate queries:

```sql
-- Create cache table
CREATE TABLE EMBEDDING_CACHE (
    query_hash RAW(32) PRIMARY KEY,
    query_text VARCHAR2(4000),
    embedding_vector VECTOR(1536),
    created_date TIMESTAMP,
    hit_count NUMBER DEFAULT 0,
    last_accessed TIMESTAMP
);

-- Create index
CREATE INDEX idx_cache_accessed ON EMBEDDING_CACHE(last_accessed);

-- Modified embedding function with cache
CREATE OR REPLACE FUNCTION GET_EMBEDDING_CACHED(
    p_text IN VARCHAR2
) RETURN VECTOR
IS
    l_hash RAW(32);
    l_vector VECTOR(1536);
BEGIN
    -- Generate hash of input text
    l_hash := DBMS_CRYPTO.HASH(UTL_RAW.CAST_TO_RAW(UPPER(TRIM(p_text))), DBMS_CRYPTO.HASH_SH256);
    
    -- Check cache
    BEGIN
        SELECT embedding_vector INTO l_vector
        FROM EMBEDDING_CACHE
        WHERE query_hash = l_hash;
        
        -- Update hit count
        UPDATE EMBEDDING_CACHE
        SET hit_count = hit_count + 1,
            last_accessed = SYSTIMESTAMP
        WHERE query_hash = l_hash;
        
        RETURN l_vector;
        
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            -- Generate new embedding
            l_vector := GENERATE_EMBEDDING_AI_FOUNDRY(p_text);
            
            -- Store in cache
            INSERT INTO EMBEDDING_CACHE (
                query_hash, query_text, embedding_vector,
                created_date, last_accessed
            ) VALUES (
                l_hash, SUBSTR(p_text, 1, 4000), l_vector,
                SYSTIMESTAMP, SYSTIMESTAMP
            );
            
            COMMIT;
            RETURN l_vector;
    END;
END;
/
```

### 8.3. Monitoring and Logging

Track API usage and performance:

```sql
-- Create monitoring table
CREATE TABLE AI_FOUNDRY_API_LOG (
    log_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    request_timestamp TIMESTAMP,
    operation VARCHAR2(100),
    input_text_length NUMBER,
    response_time_ms NUMBER,
    token_count NUMBER,
    status VARCHAR2(20),
    error_message VARCHAR2(4000),
    workspace_name VARCHAR2(100),
    deployment_name VARCHAR2(100)
);

-- Modified function with logging
CREATE OR REPLACE FUNCTION GENERATE_EMBEDDING_WITH_LOG(
    p_text IN VARCHAR2
) RETURN VECTOR
IS
    l_start_time TIMESTAMP := SYSTIMESTAMP;
    l_vector VECTOR(1536);
    l_response_time NUMBER;
    l_status VARCHAR2(20) := 'SUCCESS';
    l_error VARCHAR2(4000);
BEGIN
    -- Generate embedding
    l_vector := GENERATE_EMBEDDING_AI_FOUNDRY(p_text);
    
    -- Calculate response time
    l_response_time := EXTRACT(SECOND FROM (SYSTIMESTAMP - l_start_time)) * 1000;
    
    -- Log successful call
    INSERT INTO AI_FOUNDRY_API_LOG (
        request_timestamp, operation, input_text_length,
        response_time_ms, status, workspace_name, deployment_name
    ) VALUES (
        l_start_time, 'GENERATE_EMBEDDING', LENGTH(p_text),
        l_response_time, l_status, '<workspace-name>', 'text-embedding-3-small'
    );
    
    COMMIT;
    RETURN l_vector;
    
EXCEPTION
    WHEN OTHERS THEN
        l_status := 'ERROR';
        l_error := SQLERRM;
        l_response_time := EXTRACT(SECOND FROM (SYSTIMESTAMP - l_start_time)) * 1000;
        
        -- Log error
        INSERT INTO AI_FOUNDRY_API_LOG (
            request_timestamp, operation, input_text_length,
            response_time_ms, status, error_message,
            workspace_name, deployment_name
        ) VALUES (
            l_start_time, 'GENERATE_EMBEDDING', LENGTH(p_text),
            l_response_time, l_status, l_error,
            '<workspace-name>', 'text-embedding-3-small'
        );
        
        COMMIT;
        RAISE;
END;
/

-- Query performance metrics
SELECT 
    TRUNC(request_timestamp, 'HH') as hour,
    COUNT(*) as total_requests,
    ROUND(AVG(response_time_ms), 2) as avg_response_ms,
    ROUND(MAX(response_time_ms), 2) as max_response_ms,
    SUM(CASE WHEN status = 'ERROR' THEN 1 ELSE 0 END) as error_count
FROM AI_FOUNDRY_API_LOG
WHERE request_timestamp >= SYSDATE - 1
GROUP BY TRUNC(request_timestamp, 'HH')
ORDER BY hour DESC;
```

---

## 9. Cost Analysis

### 9.1. Pricing Breakdown

**Azure AI Foundry Workspace:** Free (no charge for workspace itself)

**Azure OpenAI Service (text-embedding-3-small):**
- **Cost per 1M tokens:** $0.02
- **Average tokens per service request:** 200 tokens (title + description + activities)

**Example Cost Calculation:**

```
Initial Indexing:
├── 1,000,000 service requests
├── 200 tokens per request average
├── Total: 200,000,000 tokens
└── Cost: 200M × $0.02 / 1M = $4.00

Ongoing Usage (Monthly):
├── New service requests: 10,000/month
├── User searches: 10,000/month
├── Total tokens: (10K + 10K) × 200 = 4M tokens/month
└── Cost: 4M × $0.02 / 1M = $0.08/month = $0.96/year

Total First Year Cost: $4.96
Subsequent Years: ~$1-2 (delta indexing only)
```

### 9.2. Cost Optimization Tips

1. **Batch Processing**
   - Process multiple texts in single API call
   - Reduce HTTP overhead

2. **Incremental Indexing**
   - Only index new/modified service requests
   - Use delta detection

3. **Query Caching**
   - Cache common queries (implement above)
   - Reduce redundant API calls

4. **Text Preprocessing**
   - Remove unnecessary content before embedding
   - Truncate very long texts (max 8192 tokens)

5. **Monitor Usage**
   ```sql
   -- Daily cost estimate
   SELECT 
       TRUNC(request_timestamp) as day,
       SUM(token_count) as total_tokens,
       ROUND(SUM(token_count) * 0.02 / 1000000, 4) as estimated_cost_usd
   FROM AI_FOUNDRY_API_LOG
   WHERE request_timestamp >= SYSDATE - 30
   GROUP BY TRUNC(request_timestamp)
   ORDER BY day DESC;
   ```

### 9.3. Cost Comparison with Alternatives

| Solution | Annual Cost | Pros | Cons |
|----------|-------------|------|------|
| **Azure AI Foundry (OpenAI)** | **$5-10** | Lowest cost, best quality, Azure-native | Requires setup |
| Azure Cognitive Search | $200-500 | Managed service | Higher cost, less flexible |
| Self-hosted (Sentence Transformers) | $500-1000 | Full control | VM costs, maintenance |
| OCI Generative AI | $100-150 | Oracle-native | Cross-cloud latency, higher cost |

---

## 10. Troubleshooting

### 10.1. Common Issues

#### Issue 1: Connection Refused

**Symptoms:**
```
ORA-29273: HTTP request failed
ORA-12545: Connect failed
```

**Solutions:**
1. Verify private endpoint is created and linked to VNet
2. Check NSG rules allow Oracle 23ai VM → AI Foundry subnet
3. Test DNS resolution: `nslookup <workspace-name>.<region>.api.azureml.ms`
4. Verify network ACL in Oracle database

#### Issue 2: 401 Unauthorized

**Symptoms:**
```
{"error":{"code":"Unauthorized","message":"Access denied"}}
```

**Solutions:**
1. Verify API key is correct in credential
2. Check credential name matches in PL/SQL code
3. Regenerate API key from Azure Portal
4. Verify workspace/deployment names are correct

#### Issue 3: 429 Too Many Requests

**Symptoms:**
```
{"error":{"code":"429","message":"Rate limit exceeded"}}
```

**Solutions:**
1. Implement delays between batch processing (see 8.1)
2. Request TPM limit increase from Azure support
3. Reduce batch size
4. Distribute load across multiple deployments

#### Issue 4: Timeout Errors

**Symptoms:**
```
ORA-29270: Connection timeout
```

**Solutions:**
1. Increase UTL_HTTP timeout:
   ```sql
   UTL_HTTP.SET_TRANSFER_TIMEOUT(300); -- 5 minutes
   ```
2. Check network latency to Azure AI Foundry
3. Reduce text length before embedding
4. Check AI Foundry service health in Azure Portal

#### Issue 5: Invalid Vector Dimensions

**Symptoms:**
```
ORA-51805: Vector dimension count must match column definition
```

**Solutions:**
1. Verify table has VECTOR(1536) definition
2. Check API response format
3. Verify model deployment is text-embedding-3-small (not 3-large which is 3072)

### 10.2. Diagnostic Queries

```sql
-- Check recent errors
SELECT log_id, request_timestamp, operation, error_message
FROM AI_FOUNDRY_API_LOG
WHERE status = 'ERROR'
  AND request_timestamp >= SYSDATE - 1
ORDER BY request_timestamp DESC;

-- Check credential status
SELECT credential_name, username, enabled, created
FROM user_credentials
WHERE credential_name = 'AZURE_AI_FOUNDRY_CRED';

-- Check network ACL
SELECT host, lower_port, upper_port, acl
FROM dba_network_acls
WHERE host LIKE '%azureml%';

-- Check pending narratives
SELECT COUNT(*) as pending,
       COUNT(CASE WHEN processed_flag = 'E' THEN 1 END) as errors,
       COUNT(CASE WHEN processed_flag = 'Y' THEN 1 END) as completed
FROM SIEBEL_NARRATIVES_STAGING;
```

### 10.3. Azure AI Foundry Health Check

```bash
# Check workspace status
az ml workspace show \
  --name $WORKSPACE_NAME \
  --resource-group $RESOURCE_GROUP \
  --query provisioningState

# Check deployment status
az cognitiveservices account deployment show \
  --name siebel-openai-service \
  --resource-group $RESOURCE_GROUP \
  --deployment-name text-embedding-3-small \
  --query properties.provisioningState

# Test API directly
curl -X POST \
  "https://<workspace-name>.<region>.api.azureml.ms/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-02-15-preview" \
  -H "api-key: <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{"input":"test","model":"text-embedding-3-small"}'
```

### 10.4. Support Resources

- **Azure AI Foundry Documentation**: https://learn.microsoft.com/azure/ai-studio/
- **Azure OpenAI Service**: https://learn.microsoft.com/azure/ai-services/openai/
- **Oracle 23ai AI Vector Search**: https://docs.oracle.com/en/database/oracle/oracle-database/23/vecse/
- **Azure Support**: Create ticket in Azure Portal
- **Oracle Support**: My Oracle Support (MOS)

---

## Summary

This guide covered:

✅ **Azure AI Foundry workspace setup** - Complete configuration with private endpoints  
✅ **OpenAI deployment** - text-embedding-3-small with 1536 dimensions  
✅ **Oracle 23ai integration** - PL/SQL functions with DBMS_CLOUD  
✅ **Performance optimization** - Batching, caching, monitoring  
✅ **Cost analysis** - $5-10/year for typical workload  
✅ **Troubleshooting** - Common issues and solutions  

**Next Steps:**
1. Complete Oracle 23ai VM deployment (Deployment Guide Phase 1-3)
2. Create Azure AI Foundry workspace and deploy OpenAI model
3. Configure private endpoint connectivity
4. Implement PL/SQL embedding functions
5. Test with sample service requests
6. Monitor performance and costs
7. Optimize based on usage patterns

For detailed deployment procedures, refer to:
- **Deployment Guide** - Complete step-by-step instructions
- **TDD 2** - Vector database and indexing pipeline
- **TDD 3** - Semantic search API implementation
