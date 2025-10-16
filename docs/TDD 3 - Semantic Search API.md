# Technical Design Document 3: Semantic Search API 

**Version:** 2.1
**Date:** 2025-10-16
**Owner:** Database Development Team

## 1. Objective
This document specifies the design of the RESTful API that will provide the semantic search functionality. This API will be built and hosted directly within the Oracle 23ai database using **Oracle REST Data Services (ORDS)**. It serves as a high-performance, secure endpoint that encapsulates all AI and vector database logic, abstracting it from the client applications (Siebel and the test app).

## 2. Technology Stack
* **Hosting Platform:** Oracle Database 23ai (Enterprise Edition)
* **API Engine:** Oracle REST Data Services (ORDS)
* **Language:** PL/SQL, SQL with JSON extensions

## 3. API Endpoint Definition

The public contract of the API remains the same to ensure client decoupling.

### 3.1. Search Endpoint
* **URL:** `https://<db_host>/ords/semantic_search/siebel/search`
* **Method:** `POST`

#### 3.1.1. Request Payload
The request body will be plain text containing the user's query. A header will specify the number of results desired.
* **Header:** `Top-K: 5`
* **Body (text/plain):** `The screen on my work laptop is flickering and showing strange colors.`

#### 3.1.2. Response Payload (Success - HTTP 200 OK)
```json
{
  "search_id": "a7b2c9f4-e8d1-4f6a-9b3c-5d1e2f7a8b9c",
  "recommendations": [
    {
      "rank": 1,
      "catalog_item_id": "1-ABCDE",
      "catalog_path": " > Hardware > Laptop > Screen Repair",
      "relevance_score": 0.8954
    },
    {
      "rank": 2,
      "catalog_item_id": "1-FGHIJ",
      "catalog_path": " > Hardware > Laptop > Full Replacement",
      "relevance_score": 0.8512
    }
  ]
}
````

## 4\. Core Logic: PL/SQL Implementation

All API logic will be encapsulated in a single PL/SQL stored procedure within the `SEMANTIC_SEARCH` schema.

### 4.1. Prerequisites for OCI Callouts

1.  **Network ACL:** The database administrator must configure a Network Access Control List (ACL) to allow the `SEMANTIC_SEARCH` schema to make outbound HTTPS requests to the OCI Generative AI endpoint.
2.  **OCI Credentials:** An OCI credential object must be created in the database to securely handle authentication with the OCI APIs.
    ```sql
    BEGIN
      DBMS_CLOUD.CREATE_CREDENTIAL(
        credential_name => 'OCI_GENAI_CREDENTIAL',
        user_ocid       => 'ocid1.user.oc1..aaaa...',
        tenancy_ocid    => 'ocid1.tenancy.oc1..aaaa...',
        private_key     => '-----BEGIN PRIVATE KEY-----...',
        fingerprint     => 'f1:2e:...'
      );
    END;
    /
    ```

### 4.2. Stored Procedure: `GET_SEMANTIC_RECOMMENDATIONS`

This procedure will contain the end-to-end search logic.

```sql
CREATE OR REPLACE PROCEDURE SEMANTIC_SEARCH.GET_SEMANTIC_RECOMMENDATIONS (
    p_query IN CLOB,
    p_top_k IN NUMBER DEFAULT 5
) AS
    l_query_vector      VECTOR;
    l_json_response     CLOB;
    l_oci_response      CLOB;
    l_oci_req_body      CLOB;
    l_embedding_url     VARCHAR2(512) := '[https://inference.generativeai](https://inference.generativeai).<region>[.oci.oraclecloud.com/20231130/actions/embedText](https://.oci.oraclecloud.com/20231130/actions/embedText)';
BEGIN
    -- Step 1: Generate vector for the user query by calling OCI GenAI service
    l_oci_req_body := '{"servingMode": {"servingType": "ON_DEMAND"}, "compartmentId": "<your_compartment_ocid>", "inputs": [{"text": "' || p_query || '"}], "modelId": "cohere.embed-english-v3.0"}';
    
    l_oci_response := DBMS_CLOUD.SEND_REQUEST(
        credential_name => 'OCI_GENAI_CREDENTIAL',
        uri             => l_embedding_url,
        method          => 'POST',
        body            => l_oci_req_body
    );
    
    -- Extract the vector from the JSON response
    SELECT jt.embedding INTO l_query_vector
    FROM JSON_TABLE(l_oci_response, '$.embeddings[*]' COLUMNS (embedding VECTOR PATH '$[*]'));

    -- Step 2: Perform vector search, rank results, and construct the final JSON response
    SELECT JSON_OBJECT (
        'search_id' KEY SYS_GUID(),
        'recommendations' KEY (
            SELECT JSON_ARRAYAGG (
                JSON_OBJECT (
                    'rank'            KEY rnk,
                    'catalog_item_id' KEY catalog_item_id,
                    'catalog_path'    KEY catalog_path,
                    'relevance_score' KEY relevance_score
                ) ORDER BY rnk
            )
            FROM (
                -- Subquery to perform ranking (frequency count in this example)
                SELECT
                    catalog_item_id,
                    catalog_path,
                    -- Simple relevance score based on frequency in top results
                    COUNT(*) / 50 AS relevance_score, 
                    ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) as rnk
                FROM (
                    -- Core vector search query
                    SELECT
                        v.CATALOG_ITEM_ID,
                        v.CATALOG_PATH
                    FROM SEMANTIC_SEARCH.SIEBEL_KNOWLEDGE_VECTORS v
                    ORDER BY VECTOR_DISTANCE(v.NARRATIVE_VECTOR, l_query_vector, COSINE)
                    FETCH FIRST 50 ROWS ONLY
                )
                GROUP BY catalog_item_id, catalog_path
                ORDER BY COUNT(*) DESC
                FETCH FIRST p_top_k ROWS ONLY
            )
        )
    ) INTO l_json_response
    FROM DUAL;

    -- Step 3: Set HTTP headers and print the JSON response
    ORDS.SET_HEADER('Content-Type', 'application/json');
    HTP.P(l_json_response);

EXCEPTION
    WHEN OTHERS THEN
        ORDS.SET_HTTP_STATUS_CODE(500);
        HTP.P('{"error": "An internal error occurred during search.", "details": "' || SQLERRM || '"}');
END;
/
```

## 5\. ORDS Endpoint Configuration

The following PL/SQL block will be executed once to publish the stored procedure as a REST endpoint.

```sql
BEGIN
    -- Enable ORDS for the SEMANTIC_SEARCH schema
    ORDS.ENABLE_SCHEMA(
        p_enabled => TRUE,
        p_schema  => 'SEMANTIC_SEARCH'
    );

    -- Define a module as a container for our APIs
    ORDS.DEFINE_MODULE(
        p_module_name => 'siebel.api',
        p_base_path   => '/siebel/'
    );

    -- Define the specific endpoint template
    ORDS.DEFINE_TEMPLATE(
        p_module_name => 'siebel.api',
        p_pattern     => 'search'
    );

    -- Define the handler for the POST method
    ORDS.DEFINE_HANDLER(
        p_module_name => 'siebel.api',
        p_pattern     => 'search',
        p_method      => 'POST',
        p_source_type => ORDS.source_type_plsql,
        -- This block maps the HTTP request to the PL/SQL procedure
        p_source      => 'BEGIN SEMANTIC_SEARCH.GET_SEMANTIC_RECOMMENDATIONS(p_query => :body_text, p_top_k => :top_k); END;',
        p_items_per_page => 0
    );
    
    -- Define the :top_k header parameter
    ORDS.DEFINE_PARAMETER(
        p_handler_name  => 'siebel.api',
        p_pattern       => 'search',
        p_method        => 'POST',
        p_name          => 'top_k',
        p_bind_variable_name => 'top_k',
        p_source_type   => 'HEADER',
        p_param_type    => 'INT',
        p_access_method => 'IN'
    );

    COMMIT;
END;
/
```

## 6\. Security

  * **Authentication:** The ORDS endpoint can be protected using ORDS roles and privileges, or more robustly with an OAuth2 client credentials flow. For initial integration, it can be protected via an API Gateway which requires an API Key.
  * **Network Security:** The ORDS listener should be placed in a private subnet, with access controlled by an OCI Load Balancer or API Gateway.
  * **Credential Management:** Database credentials are no longer needed externally. The OCI credential for the GenAI service is stored securely within the database itself. This significantly improves the security posture.
