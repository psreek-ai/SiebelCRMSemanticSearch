# Technical Design Document 4: Siebel CRM Integration

**Version:** 2.1 (ORDS Edition)
**Date:** 2025-10-16
**Owner:** Siebel Development Team

## 1. Objective
This document provides the technical specification for integrating the new Semantic Search API (hosted via ORDS) into the Siebel Open UI application. This involves modifying the existing catalog search functionality to call the API and display its intelligent recommendations.

## 2. Siebel Objects to be Modified/Created

| Object Type | Object Name | Action | Description |
| :--- | :--- | :--- | :--- |
| **Business Service** | `SemanticSearchAPIService`| Create | A new BS to encapsulate the logic for calling the ORDS API. |
| **Applet** | `Service Request Catalog Search Applet` (Example Name) | Modify | The existing search applet's "Search" button will be reconfigured to invoke the new logic. |
| **Applet** | `Service Request Catalog Results Applet` (Example Name) | Modify | The results applet to handle the new display order and UI elements. |
| **Named Subsystem** | `SemanticSearchORDS` | Create | To securely store the ORDS API URL and any necessary authentication tokens/keys. |
| **Presentation Model**| `CatalogSearchPM.js` (Example Name) | Modify/Create | A custom PM to handle the asynchronous API call and dynamic refresh of the results. |

## 3. User Interface and Experience Flow
1.  The user navigates to the "Raise a Request" view.
2.  The user types their query (e.g., "my computer is slow") into the search input field of the search applet.
3.  The user clicks the "Search" button.
4.  **New:** The existing list of results in the `Service Request Catalog Results Applet` is cleared. A loading indicator (e.g., a spinner icon) is displayed over the applet to provide immediate feedback.
5.  The custom Presentation Model (PM) calls the `SemanticSearchAPIService`.
6.  The business service makes an HTTPS `POST` request to the ORDS endpoint.
7.  Upon receiving the API response, the PM hides the loading indicator.
8.  The PM parses the JSON response and constructs a dynamic `SearchSpec` and `SortSpec`.
9.  The PM applies the new search and sort specifications to the results applet's business component, which refreshes to display the recommended items in the exact order provided by the API.



## 4. Business Service: `SemanticSearchAPIService`

### 4.1. Method: `InvokeSearch`
* **Inputs:**
    * `QueryText` (String): The user's search query.
    * `TopK` (String): The number of results to fetch, e.g., "5".
* **Outputs:**
    * `ResultSet` (PropertySet): A property set containing the ordered list of catalog item IDs, paths, and ranks.
    * `Error` (String): An error message if the API call fails.

### 4.2. eScript Implementation Detail (`InvokeSearch` method)
This script will use the standard `EAI HTTP Transport` Business Service to make the outbound REST call.

```javascript
// eScript for SemanticSearchAPIService.InvokeSearch method
function InvokeSearch(inputs, outputs) {
    var queryText = inputs.GetProperty("QueryText");
    var topK = inputs.GetProperty("TopK") || "5"; // Default to 5
    var psResults = TheApplication().NewPropertySet();
    psResults.SetType("ResultSet");

    try {
        // 1. Retrieve API details securely from Named Subsystem
        var url = TheApplication().GetProfileAttr("SemanticSearchORDS_URL");
        var apiKey = TheApplication().GetProfileAttr("SemanticSearchORDS_APIKey"); // If using an API Gateway in front of ORDS

        // 2. Prepare HTTP request for ORDS
        var httpSvc = TheApplication().GetService("EAI HTTP Transport");
        var httpInputs = TheApplication().NewPropertySet();
        var httpOutputs = TheApplication().NewPropertySet();

        httpInputs.SetProperty("HTTPRequestMethod", "POST");
        httpInputs.SetProperty("HTTPRequestBody", queryText); // The body is plain text for this API
        httpInputs.SetProperty("HTTPRequestURL", url);
        httpInputs.SetProperty("HTTPContentType", "text/plain"); // Set content type to plain text
        httpInputs.SetProperty("HTTPHeader.Top-K", topK); // Pass Top-K as a header
        if (apiKey) {
            httpInputs.SetProperty("HTTPHeader.X-API-Key", apiKey); // Example security header for API Gateway
        }

        // 3. Invoke the API
        httpSvc.InvokeMethod("SendReceive", httpInputs, httpOutputs);

        // 4. Parse the JSON response
        var responseStr = httpOutputs.GetValue();
        if (responseStr) {
            var responseObj = JSON.parse(responseStr);
            var recommendations = responseObj.recommendations;

            if (recommendations && recommendations.length > 0) {
                for (var i = 0; i < recommendations.length; i++) {
                    var rec = recommendations[i];
                    var psRec = TheApplication().NewPropertySet();
                    psRec.SetProperty("Id", rec.catalog_item_id);
                    psRec.SetProperty("Path", rec.catalog_path);
                    psRec.SetProperty("Rank", rec.rank);
                    psResults.AddChild(psRec);
                }
            }
        }
    } catch (e) {
        outputs.SetProperty("Error", e.toString());
        // The PM will handle displaying a user-friendly error message.
    } finally {
        outputs.AddChild(psResults);
    }
    return (CancelOperation);
}
````

## 5\. Applet and View Integration

### 5.1. Presentation Model (PM) Logic (`CatalogSearchPM.js`)

The PM will be the central controller for the user interaction.

```javascript
// Conceptual logic for the custom Presentation Model
// This code would be attached to the "Search" button's click event handler.

// 1. Get the search query from the input control
var query = this.Get("GetSearchQueryControlValue");
if (!query || query.trim() === "") {
    SiebelApp.S_App.ui.ShowError("Please enter a search query.");
    return;
}

// 2. Show a loading indicator and clear previous results
var resultsApplet = this.Get("GetResultsApplet");
var bc = resultsApplet.GetBusComp();
bc.NewRecord(false); // Clear the applet
this.Get("ShowLoadingIndicator")(true); // This would be a custom function to show a spinner

// 3. Prepare inputs for the Business Service
var bsInputs = SiebelApp.S_App.NewPropertySet();
bsInputs.SetProperty("QueryText", query);
bsInputs.SetProperty("TopK", "5");

// 4. Call the Business Service asynchronously
SiebelApp.S_App.GetService("SemanticSearchAPIService").InvokeMethod("InvokeSearch", bsInputs, {
    async: true,
    scope: this,
    callback: function(methodName, inputs, outputs) {
        // 5. Hide loading indicator on callback
        this.Get("ShowLoadingIndicator")(false);

        // 6. Process the response
        var error = outputs.GetProperty("Error");
        if (error) {
            SiebelApp.S_App.ui.ShowError("Search is temporarily unavailable. Please try again later.");
            return;
        }

        var resultSet = outputs.GetChildByType("ResultSet");
        if (resultSet) {
            var recommendations = resultSet.GetChildren();
            if (recommendations.length > 0) {
                var idList = [];
                var sortClauses = [];
                for (var i = 0; i < recommendations.length; i++) {
                    var rec = recommendations[i];
                    var id = rec.GetProperty("Id");
                    var rank = rec.GetProperty("Rank");
                    idList.push("'" + id + "'");
                    sortClauses.push("WHEN '" + id + "' THEN " + rank);
                }
                
                // 7. Construct the SearchSpec and SortSpec to enforce the API's ranking
                var searchSpec = "[Id] IN (" + idList.join(",") + ")";
                var sortSpec = "CASE [Id] " + sortClauses.join(" ") + " ELSE 99 END";

                // 8. Apply the query to the results applet's business component
                bc.ClearToQuery();
                bc.SetSortSpec(sortSpec); // CRITICAL: Set SortSpec before SearchSpec
                bc.SetSearchSpecStr(searchSpec);
                bc.ExecuteQuery(SiebelApp.S_App.GetSWEConsts().get("SWE_QUERY_FWD_ONLY"));
            } else {
                // Handle case with no results by displaying a message in the UI
                SiebelApp.S_App.ui.ShowError("No relevant results found for your query.");
            }
        }
    }
});
```

## 6\. Deployment Objects

  * **Business Service:** `SemanticSearchAPIService`
  * **Named Subsystem:** `SemanticSearchORDS` (with `URL` and optional `APIKey` parameters)
  * **Custom JS Files:** `CatalogSearchPM.js` (and associated PR if needed for loading indicator)
  * **Siebel Web Templates:** Manifest registration for the new custom JS files.
  * **SRF:** All compiled object definitions.
