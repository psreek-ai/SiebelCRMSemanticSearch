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
```

## 4. Core Logic: PL/SQL Implementation
All API logic will be encapsulated in a single PL/SQL package within the `SEMANTIC_SEARCH` schema for better organization and maintenance.

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
    l_embedding_url     VARCHAR2(512) := 'https://inference.generativeai.<region>.oci.oraclecloud.com/20231130/actions/embedText';
    l_search_id         VARCHAR2(64) := SYS_GUID();
BEGIN
    -- Step 1: Generate vector for the user query by calling OCI GenAI service
    l_oci_req_body := JSON_OBJECT(
        'servingMode'   KEY 'servingType' VALUE 'ON_DEMAND',
        'compartmentId' KEY 'value' VALUE '<your_compartment_ocid>',
        'modelId'       KEY 'value' VALUE 'cohere.embed-english-v3.0',
        'inputs'        KEY 'value' VALUE JSON_ARRAY(
            JSON_OBJECT('text' KEY 'value' VALUE p_query)
        )
    );
    
    l_oci_response := DBMS_CLOUD.SEND_REQUEST(
        credential_name => 'OCI_GENAI_CREDENTIAL',
        uri             => l_embedding_url,
        method          => 'POST',
        body            => UTL_RAW.CAST_TO_RAW(l_oci_req_body)
    );
    
    -- Extract the vector from the JSON response
    SELECT jt.embedding INTO l_query_vector
    FROM JSON_TABLE(l_oci_response, '$.embeddings[*]' COLUMNS (embedding VECTOR PATH '$[*]'));

    -- Step 2: Perform vector search, rank results, and construct the final JSON response
    SELECT JSON_OBJECT (
        'search_id' KEY l_search_id,
        'recommendations' KEY (
            SELECT JSON_ARRAYAGG(
                JSON_OBJECT(
                    'rank' KEY r.rnk,
                    'catalog_item_id' KEY r.CATALOG_ITEM_ID,
                    'catalog_path' KEY r.CATALOG_PATH,
                    'relevance_score' KEY r.relevance_score
                )
                ORDER BY r.rnk
            )
            FROM (
                SELECT
                    v.CATALOG_ITEM_ID,
                    v.CATALOG_PATH,
                    VECTOR_DISTANCE(v.NARRATIVE_VECTOR, l_query_vector, COSINE) as relevance_score,
                    DENSE_RANK() OVER (ORDER BY VECTOR_DISTANCE(v.NARRATIVE_VECTOR, l_query_vector, COSINE) DESC) as rnk
                FROM
                    SIEBEL_KNOWLEDGE_VECTORS v
                ORDER BY
                    relevance_score DESC
                FETCH FIRST p_top_k ROWS ONLY
            ) r
        )
    ) INTO l_json_response
    FROM DUAL;

    -- Step 3: Set HTTP headers and print the response
    ORDS.SET_HEADER('Content-Type', 'application/json');
    HTP.P(l_json_response);

EXCEPTION
    WHEN OTHERS THEN
        ORDS.SET_STATUS(500); -- Internal Server Error
        HTP.P('{"error": "' || SQLERRM || '"}');
END GET_SEMANTIC_RECOMMENDATIONS;
/
```

### 4.3. ORDS Configuration
The following block registers the PL/SQL procedure as a REST endpoint.

```sql
BEGIN
    ORDS.ENABLE_SCHEMA(p_enabled => TRUE, p_schema => 'SEMANTIC_SEARCH');

    ORDS.DEFINE_MODULE(
        p_module_name    => 'siebel',
        p_base_path      => '/siebel/',
        p_items_per_page => 0
    );

    ORDS.DEFINE_TEMPLATE(
        p_module_name => 'siebel',
        p_pattern     => 'search'
    );

    ORDS.DEFINE_HANDLER(
        p_module_name    => 'siebel',
        p_pattern        => 'search',
        p_method         => 'POST',
        p_source_type    => ORDS.source_type_plsql,
        p_source         => 'BEGIN SEMANTIC_SEARCH.GET_SEMANTIC_RECOMMENDATIONS(p_query => :body_text, p_top_k => :top_k); END;',
        p_items_per_page => 0
    );
    
    -- Define the parameter for the Top-K header
    ORDS.DEFINE_PARAMETER(
        p_module_name        => 'siebel',
        p_pattern            => 'search',
        p_method             => 'POST',
        p_name               => 'top_k',
        p_bind_variable_name => 'top_k',
        p_source_type        => 'HEADER',
        p_param_type         => 'INT',
        p_access_method      => 'IN'
    );

    COMMIT;
END;
/
```
