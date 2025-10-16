# Deployment Guide: AI-Powered Semantic Search Solution

**Version:** 2.0
**Date:** 2025-10-16
**Audience:** Deployment Team, System Administrators

## 1. Introduction
This guide provides a step-by-step process for deploying the AI-Powered Semantic Search solution into a target environment (e.g., DEV, UAT, Production). It is critical that these steps are followed in the specified sequence to ensure a successful deployment.

## 2. Pre-Deployment Checklist
- [ ] OCI user has necessary permissions to manage Functions, API Gateway, and Vault.
- [ ] Network connectivity is established between the OCI VCN and the Siebel environment (if on-premise).
- [ ] Terraform CLI is installed and configured on the deployment machine.
- [ ] All environment-specific configuration values (API keys, DB credentials, URLs) are populated in a `.tfvars` file.
- [ ] The final Siebel SRF file has been compiled and is ready for deployment.

## 3. Deployment Sequence (Infrastructure as Code)
Deployment will be automated using Terraform where possible.

### Phase 1: Backend & API Layer Deployment (Terraform)
1.  **Initialize Terraform:** `terraform init`
2.  **Plan Deployment:** `terraform plan -var-file=<env>.tfvars`
3.  **Apply Deployment:** `terraform apply -var-file=<env>.tfvars`
    * This single command will provision:
        * The `SEMANTIC_SEARCH` schema in the Oracle 23ai database.
        * The `SIEBEL_KNOWLEDGE_VECTORS` table.
        * The OCI Vault secrets for database credentials.
        * The OCI Function application and the API function itself.
        * The API Gateway and its endpoint configuration, including the API key.
        * All necessary IAM policies.

### Phase 2: Frontend Integration Deployment (Siebel)
1.  **Deploy Siebel Repository:**
    * Stop the target Siebel Application Object Manager.
    * Copy the new SRF file to the `objects` directory.
    * Start the Siebel Application Object Manager.
2.  **Configure Named Subsystem:**
    * Log in to the Siebel client with administrative privileges.
    * Navigate to `Administration - Server Configuration > Profile Configuration`.
    * Create a new Named Subsystem profile named `SemanticSearchAPI`.
    * Add two parameters to the profile:
        * `URL`: Set this to the API Gateway invocation URL output from the Terraform script.
        * `APIKey`: Set this to the public API Key value output from the Terraform script.
3.  **Restart Siebel Servers:**
    * Perform a rolling restart of the Siebel server components to ensure the changes are loaded.

### Phase 3: Initial Data Load
1.  **Run Data Extraction:**
    * Execute the batched data extraction SQL scripts on the Oracle 12c database.
    * Export the results to CSV files and upload them to an OCI Object Storage bucket.
2.  **Run Indexing Pipeline:**
    * Trigger the Python indexing pipeline script (this can be a manually run script or a CI/CD job).
    * The script will read the CSV files from Object Storage, generate embeddings, and populate the vector table.
3.  **Create Vector Index:**
    * After the data load is complete, connect to the Oracle 23ai database.
    * Run the DDL script to create the `siebel_vector_idx` vector index.

## 4. Post-Deployment Verification
1.  Check the OCI console to ensure all resources (Function, Gateway) are active.
2.  Use the Standalone Test App or Postman to make a test call to the API endpoint.
3.  Log in to the Siebel application and perform a search to verify end-to-end functionality.

## 5. Rollback Plan
* **Terraform:** Run `terraform destroy` to tear down all OCI resources.
* **Siebel:** Deploy the previous version of the SRF file and restart the Siebel servers.
* **Database:** The vector table can be truncated (`TRUNCATE TABLE SEMANTIC_SEARCH.SIEBEL_KNOWLEDGE_VECTORS;`) if a re-load is needed.
