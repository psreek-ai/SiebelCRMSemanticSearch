# Deployment Guide: AI-Powered Semantic Search Solution

**Version:** 2.1 (ORDS Edition)
**Date:** 2025-10-16
**Audience:** Deployment Team, Database Administrators, System Administrators

## 1. Introduction
This guide provides a step-by-step process for deploying the AI-Powered Semantic Search solution into a target environment (e.g., DEV, UAT, Production). This version specifically details the deployment using the **Oracle REST Data Services (ORDS)** architecture. It is critical that these steps are followed in the specified sequence.

## 2. Pre-Deployment Checklist
- [ ] **Database:** Oracle Database 23ai is provisioned. ORDS is installed and configured to be accessible over the network.
- [ ] **Network:** Network connectivity is established from the Oracle 23ai database host to the OCI Generative AI service endpoint. A Network ACL has been configured.
- [ ] **Credentials:** OCI API credentials (user OCID, tenancy OCID, private key, fingerprint) for the GenAI service are available and securely stored.
- [ ] **Siebel:** The final Siebel SRF file and custom JS files have been compiled and are ready for deployment.
- [ ] **Scripts:** All SQL and PL/SQL scripts (for schema creation, API logic, and ORDS configuration) are version-controlled and available.
- [ ] **Configuration:** All environment-specific values (e.g., OCI compartment OCID, region) are documented and ready.

## 3. Deployment Sequence

### Phase 1: Backend Database Setup (Oracle 23ai)
* **Objective:** Prepare the database schema and tables.
* **Executor:** Database Administrator

1.  **Create Schema:** Connect to the 23ai database as a privileged user (e.g., `SYS`) and run the script to create the `SEMANTIC_SEARCH` user/schema with the necessary privileges (`CREATE SESSION`, `CREATE TABLE`, `UNLIMITED TABLESPACE`).
2.  **Create Table:** Connect as the `SEMANTIC_SEARCH` user and run the DDL script to create the `SIEBEL_KNOWLEDGE_VECTORS` table.

### Phase 2: API Layer Deployment (ORDS Configuration)
* **Objective:** Deploy the PL/SQL logic and expose it as a secure REST endpoint.
* **Executor:** Database Administrator / PL/SQL Developer

1.  **Create OCI Credential:** Connect as the `SEMANTIC_SEARCH` user and run the `DBMS_CLOUD.CREATE_CREDENTIAL` PL/SQL block to store the OCI API keys securely within the database.
2.  **Deploy PL/SQL Procedure:** Compile the `GET_SEMANTIC_RECOMMENDATIONS` stored procedure in the `SEMANTIC_SEARCH` schema. This procedure contains all the core AI search logic.
3.  **Enable ORDS Endpoint:** Run the ORDS PL/SQL configuration block. This will:
    * Enable the `SEMANTIC_SEARCH` schema for REST access.
    * Define the module (`/siebel/`).
    * Define the template and handler for the `/search` endpoint, linking it to the stored procedure.
4.  **Commit Changes:** Commit the transaction to finalize the ORDS configuration.
5.  **Verify Endpoint:** Access the ORDS endpoint URL in a browser or with a tool like Postman to ensure it is active (it may return an error for a GET request, which is expected, but should not return a 404).

### Phase 3: Frontend Integration Deployment (Siebel)
* **Objective:** Deploy the Siebel repository changes and configure the connection to the new ORDS API.
* **Executor:** Siebel Administrator

1.  **Deploy Siebel Repository & Web Files:**
    * Stop the target Siebel Application Object Manager(s).
    * Copy the new SRF file to the `objects` directory.
    * Copy the custom JavaScript files (e.g., `CatalogSearchPM.js`) to the appropriate custom web scripts directory.
    * Clear the browser cache and any server-side web cache.
    * Start the Siebel Application Object Manager(s).
2.  **Configure Named Subsystem:**
    * Log in to the Siebel client with administrative privileges.
    * Navigate to `Administration - Server Configuration > Profile Configuration`.
    * Create a new Named Subsystem profile named `SemanticSearchORDS`.
    * Add the following parameters to the profile:
        * `URL`: Set this to the full ORDS invocation URL (e.g., `https://<db_host>/ords/semantic_search/siebel/search`).
        * `APIKey` (Optional): If an API Gateway is placed in front of ORDS, set the required API Key here.
3.  **Restart Siebel Servers:**
    * Perform a rolling restart of the Siebel server components to ensure the new Named Subsystem parameters are loaded into the cache.

### Phase 4: Initial Data Load and Indexing
* **Objective:** Populate the vector database with historical data.
* **Executor:** Data Engineering / DBA Team

1.  **Run Data Extraction:** Execute the batched data extraction SQL scripts on the **Oracle 12c** database. Export the results to CSV files and transfer them to a location accessible by the indexing script.
2.  **Run Indexing Pipeline:** Trigger the Python indexing pipeline script. The script will read the CSV files, call the OCI GenAI service for embeddings, and populate the `SIEBEL_KNOWLEDGE_VECTORS` table in the **Oracle 23ai** database.
3.  **Create Vector Index:** After the data load is complete, connect to the Oracle 23ai database as the `SEMANTIC_SEARCH` user and run the `CREATE VECTOR INDEX` DDL statement.

## 4. Post-Deployment Verification
1.  Use a REST client (Postman, curl) to make a `POST` request directly to the ORDS endpoint. Verify it returns a valid JSON response.
2.  Log in to the Siebel application. Navigate to the "Raise a Request" view and perform a search.
3.  Confirm that the loading indicator appears and that ranked results are returned from the API, displayed correctly in the results applet.
4.  Check the `CREATED_DATE` on the `SIEBEL_KNOWLEDGE_VECTORS` table to confirm the initial data load was successful.

## 5. Rollback Plan
* **Siebel:** Deploy the previous version of the SRF and custom JS files. Revert the changes to the search applet if necessary. Restart the Siebel servers.
* **ORDS API:**
    * Run the `ORDS.DELETE_MODULE` command to remove the API definition.
    * Run `ORDS.ENABLE_SCHEMA(p_enabled => FALSE, p_schema => 'SEMANTIC_SEARCH');` to disable REST access.
* **Database Objects:** The PL/SQL procedure and vector table can be dropped if a full rollback is required.
