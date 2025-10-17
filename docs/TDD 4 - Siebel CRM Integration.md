# Technical Design Document 4: Siebel CRM Integration

**Version:** 2.2
**Date:** 2025-10-17
**Owner:** Siebel Development Team

## 1. Objective
This document provides the technical specification for integrating the new Semantic Search API (hosted via ORDS) into the Siebel Open UI application. This involves creating business services, configuring presentation models, and modifying the UI to provide an intelligent search experience.

## 2. Prerequisites

Before implementing this TDD, ensure:
- ✅ TDD 3 is complete (ORDS API is deployed and tested)
- ✅ Siebel Tools is installed and configured
- ✅ Access to Siebel repository with check-out privileges
- ✅ Siebel Web Server and Application Server are accessible
- ✅ API endpoint URL and authentication credentials are available

## 3. Architecture Overview

### 3.1. Integration Components
```
Siebel Open UI (Browser)
      ↓
Custom Presentation Model (JavaScript)
      ↓
Business Service (eScript)
      ↓
EAI HTTP Transport Service
      ↓
ORDS API Endpoint (Oracle 23ai on Azure)
```

### 3.2. Siebel Objects to Create/Modify

| Object Type | Object Name | Action | Purpose |
|-------------|-------------|--------|---------|
| Business Service | SemanticSearchAPIService | Create | Encapsulates API calls |
| Business Service Method | InvokeSearch | Create | Makes HTTP POST request to ORDS |
| Applet | Service Request Catalog Search Applet | Modify | Add search functionality |
| Applet | Service Request Catalog Results Applet | Modify | Display ranked results |
| Named Subsystem | SemanticSearchORDS | Create | Store API configuration |
| Web Template | manifest_search.js | Create | Register custom PM |
| Presentation Model | CatalogSearchPM.js | Create | Handle async API calls |
| Physical Renderer | CatalogSearchPR.js | Create (Optional) | Custom UI rendering |

---

## 4. Implementation Steps

### Step 1: Configure Named Subsystem

**Executor:** Siebel Administrator  
**Duration:** 10 minutes

#### 1.1. Create Named Subsystem via Siebel Client

1. Log in to Siebel Client with administrative privileges
2. Navigate to **Administration - Server Configuration > Profile Configuration**
3. Click **New Record** in the Named Subsystems list
4. Set the following values:

| Field | Value |
|-------|-------|
| Name | `SemanticSearchORDS` |
| Description | `Configuration for Semantic Search ORDS API` |
| Status | `Active` |

5. In the **Parameters** list applet, add the following parameters:

**Parameter 1:**
- Parameter Name: `URL`
- Parameter Value: `http://your-ords-server:8080/ords/semantic_search/siebel/search`
- Comments: `ORDS API endpoint URL`

**Parameter 2:**
- Parameter Name: `APIKey`
- Parameter Value: `YOUR_SECURE_API_KEY_HERE`
- Comments: `API authentication key (if using API key auth)`

**Parameter 3:**
- Parameter Name: `Timeout`
- Parameter Value: `5000`
- Comments: `Request timeout in milliseconds`

**Parameter 4:**
- Parameter Name: `MaxRetries`
- Parameter Value: `2`
- Comments: `Number of retry attempts on failure`

6. Save the record

#### 1.2. Verify Named Subsystem via SQL

```sql
-- Connect to Siebel database
SELECT 
    NAME,
    DESC_TEXT,
    STATUS_FLG
FROM S_SYS_PROF
WHERE NAME = 'SemanticSearchORDS';

-- View parameters
SELECT 
    sp.NAME AS SUBSYSTEM_NAME,
    spp.NAME AS PARAM_NAME,
    spp.VAL AS PARAM_VALUE
FROM S_SYS_PROF sp
JOIN S_SYS_PROF_PROP spp ON sp.ROW_ID = spp.PAR_ROW_ID
WHERE sp.NAME = 'SemanticSearchORDS'
ORDER BY spp.NAME;
```

---

### Step 2: Create Business Service

**Executor:** Siebel Developer (Siebel Tools)  
**Duration:** 45 minutes

#### 2.1. Create Business Service Object

1. Open **Siebel Tools**
2. Navigate to **Business Service** object type
3. Right-click and select **New Record**
4. Set the following properties:

| Property | Value |
|----------|-------|
| Name | `SemanticSearchAPIService` |
| Display Name | `Semantic Search API Service` |
| Project | `<Your Project Name>` |
| Class | `CSSService` |
| Cache | `No` |
| Server Enabled | `Yes` |
| Comments | `Service to invoke semantic search ORDS API` |

5. Save the record (Ctrl+S)

#### 2.2. Create Business Service Method

1. In the **Business Service Methods** list, click **New Record**
2. Set properties for the **InvokeSearch** method:

| Property | Value |
|----------|-------|
| Name | `InvokeSearch` |
| Display Name | `Invoke Search` |
| Method Name | `InvokeSearch` |
| Comments | `Calls ORDS API with user query and returns recommendations` |

3. Save the record

#### 2.3. Create Business Service Method Arguments

Create the following **Input Arguments**:

**Argument 1:**
- Name: `QueryText`
- Type: `String`
- Required: `Yes`
- Comments: `User's search query text`

**Argument 2:**
- Name: `TopK`
- Type: `String`
- Required: `No`
- Default Value: `5`
- Comments: `Number of results to return`

Create the following **Output Arguments**:

**Argument 1:**
- Name: `ResultSet`
- Type: `Hierarchy`
- Required: `Yes`
- Comments: `Property set containing search results`

**Argument 2:**
- Name: `Error`
- Type: `String`
- Required: `No`
- Comments: `Error message if request fails`

4. Save all changes (Ctrl+S)

#### 2.4. Implement eScript Logic

1. Select the **SemanticSearchAPIService** business service
2. Go to **Business Service > Business Service User Props**
3. In the **eScript** field, add the following code:

```javascript
/*****************************************************************************
 * Business Service: SemanticSearchAPIService
 * Method: InvokeSearch
 * Purpose: Calls ORDS semantic search API with input sanitization and retry logic
 * 
 * IMPROVEMENTS:
 * - Input sanitization to prevent injection attacks
 * - Configurable retry logic for transient network errors (503, 504, 408)
 * - Exponential backoff between retries
 * - MaxRetries parameter in Named Subsystem
 *****************************************************************************/

function Service_PreInvokeMethod(MethodName, Inputs, Outputs)
{
    if (MethodName == "InvokeSearch")
    {
        return (CancelOperation);
    }
    return (ContinueOperation);
}

// NEW: Input sanitization function
function SanitizeQueryText(queryText)
{
    if (!queryText || queryText === "")
    {
        return "";
    }
    
    var sanitized = queryText;
    
    // Remove control characters
    sanitized = sanitized.replace(/[\x00-\x1F\x7F]/g, '');
    
    // Trim excessive whitespace
    sanitized = sanitized.replace(/\s+/g, ' ').trim();
    
    // Limit length to prevent payload attacks
    if (sanitized.length > 5000)
    {
        sanitized = sanitized.substring(0, 5000);
    }
    
    // Remove potential script injection
    sanitized = sanitized.replace(/<script[^>]*>.*?<\/script>/gi, '');
    sanitized = sanitized.replace(/<[^>]+>/g, '');
    
    return sanitized;
}

function InvokeSearch(Inputs, Outputs)
{
    try
    {
        var queryText = Inputs.GetProperty("QueryText");
        var topK = Inputs.GetProperty("TopK");
        
        // Validate input
        if (!queryText || queryText.trim() === "")
        {
            Outputs.SetProperty("Error", "Query text cannot be empty");
            return (CancelOperation);
        }
        
        // NEW: Sanitize input
        queryText = SanitizeQueryText(queryText);
        
        if (!queryText || queryText === "")
        {
            Outputs.SetProperty("Error", "Invalid query text after sanitization");
            return (CancelOperation);
        }
        
        // Default TopK if not provided
        if (!topK || topK === "")
        {
            topK = "5";
        }
        
        // Validate TopK is numeric
        var topKNum = parseInt(topK);
        if (isNaN(topKNum) || topKNum < 1 || topKNum > 20)
        {
            topK = "5";
        }
        
        // Get API configuration from Named Subsystem
        var apiUrl = TheApplication().GetProfileAttr("SemanticSearchORDS.URL");
        var apiKey = TheApplication().GetProfileAttr("SemanticSearchORDS.APIKey");
        var timeout = TheApplication().GetProfileAttr("SemanticSearchORDS.Timeout");
        var maxRetries = TheApplication().GetProfileAttr("SemanticSearchORDS.MaxRetries");  // NEW
        
        if (!apiUrl)
        {
            Outputs.SetProperty("Error", "API URL not configured in Named Subsystem");
            TheApplication().Trace("ERROR: SemanticSearchORDS.URL parameter not found");
            return (CancelOperation);
        }
        
        // Default timeout if not configured
        if (!timeout || timeout === "")
        {
            timeout = "5000";
        }
        
        // NEW: Default max retries
        if (!maxRetries || maxRetries === "")
        {
            maxRetries = "2";
        }
        var maxRetriesNum = parseInt(maxRetries);
        
        TheApplication().Trace("Calling Semantic Search API: " + apiUrl);
        TheApplication().Trace("Query (sanitized): " + queryText);
        TheApplication().Trace("TopK: " + topK);
        TheApplication().Trace("Max Retries: " + maxRetriesNum);
        
        // NEW: Retry loop for transient errors
        var retryCount = 0;
        var lastError = "";
        var httpStatus = "";
        var responseBody = "";
        
        while (retryCount <= maxRetriesNum)
        {
            try
            {
        }
        
        TheApplication().Trace("Calling Semantic Search API: " + apiUrl);
        TheApplication().Trace("Query (sanitized): " + queryText);
        TheApplication().Trace("TopK: " + topK);
        TheApplication().Trace("Max Retries: " + maxRetriesNum);
        
        // NEW: Retry loop for transient errors
        var retryCount = 0;
        var lastError = "";
        var httpStatus = "";
        var responseBody = "";
        
        while (retryCount <= maxRetriesNum)
        {
            try
            {
                // Prepare HTTP request
                var httpSvc = TheApplication().GetService("EAI HTTP Transport");
                var httpInputs = TheApplication().NewPropertySet();
                var httpOutputs = TheApplication().NewPropertySet();
                
                // Set HTTP request parameters
                httpInputs.SetProperty("HTTPRequestMethod", "POST");
                httpInputs.SetProperty("HTTPRequestURLEncoding", "UTF-8");
                httpInputs.SetProperty("HTTPRequestURL", apiUrl);
                httpInputs.SetProperty("HTTPContentType", "text/plain; charset=UTF-8");
                httpInputs.SetProperty("HTTPRequestBody", queryText);
                httpInputs.SetProperty("HTTPHeader.Top-K", topK);
                httpInputs.SetProperty("HTTPTimeout", timeout);
                
                // Add API key header if configured
                if (apiKey && apiKey !== "")
                {
                    httpInputs.SetProperty("HTTPHeader.X-API-Key", apiKey);
                }
                
                // Add additional headers
                httpInputs.SetProperty("HTTPHeader.Accept", "application/json");
                httpInputs.SetProperty("HTTPHeader.User-Agent", "SiebelCRM/23.0");
                
                // Invoke HTTP transport service
                httpSvc.InvokeMethod("SendReceive", httpInputs, httpOutputs);
                
                // Get response
                httpStatus = httpOutputs.GetProperty("HTTPResponseStatus");
                responseBody = httpOutputs.GetProperty("HTTPResponseBody");
                
                TheApplication().Trace("HTTP Status: " + httpStatus);
                
                // Check for HTTP errors
                if (!httpStatus || httpStatus === "")
                {
                    throw new Error("No response from API server");
                }
                
                // Parse status code
                var statusCode = parseInt(httpStatus.split(" ")[0]);
                
                // NEW: Check if error is retryable
                if (statusCode === 503 || statusCode === 504 || statusCode === 408)
                {
                    lastError = "Transient error: " + httpStatus;
                    TheApplication().Trace("Transient error, retry " + (retryCount + 1) + "/" + maxRetriesNum);
                    
                    if (retryCount < maxRetriesNum)
                    {
                        retryCount++;
                        // Exponential backoff
                        var waitTime = Math.pow(2, retryCount - 1);
                        TheApplication().Trace("Waiting " + waitTime + " second(s) before retry");
                        
                        // Simple wait using loop
                        var start = new Date().getTime();
                        while (new Date().getTime() < start + (waitTime * 1000)) { /* wait */ }
                        
                        continue; // Retry
                    }
                    else
                    {
                        throw new Error("Service unavailable after " + maxRetriesNum + " retries");
                    }
                }
                else if (statusCode !== 200)
                {
                    throw new Error("API returned error status: " + httpStatus);
                }
                
                // Success - break out of retry loop
                break;
            }
            catch (e)
            {
                lastError = e.toString();
                TheApplication().Trace("Error in retry attempt " + (retryCount + 1) + ": " + lastError);
                
                // If this was the last retry, throw the error
                if (retryCount >= maxRetriesNum)
                {
                    throw e;
                }
                
                retryCount++;
            }
        }
        
        // Parse JSON response (after successful retry loop)
        TheApplication().Trace("Response Body: " + responseBody);
        if (!responseBody || responseBody === "")
        {
            Outputs.SetProperty("Error", "Empty response from API");
            TheApplication().Trace("ERROR: Empty response body");
            return;
        }
        
        var responseObj;
        try
        {
            responseObj = JSON.parse(responseBody);
        }
        catch (e)
        {
            Outputs.SetProperty("Error", "Invalid JSON response: " + e.toString());
            TheApplication().Trace("ERROR: JSON parse failed - " + e.toString());
            return;
        }
        
        // Check for API-level errors
        if (responseObj.error)
        {
            Outputs.SetProperty("Error", "API Error: " + responseObj.error);
            TheApplication().Trace("ERROR: API returned error - " + responseObj.error);
            return;
        }
        
        // Process recommendations
        var recommendations = responseObj.recommendations;
        if (!recommendations || recommendations.length === 0)
        {
            Outputs.SetProperty("Error", "No recommendations found");
            TheApplication().Trace("INFO: No recommendations returned");
            return;
        }
        
        // Build result property set
        var psResults = TheApplication().NewPropertySet();
        psResults.SetType("ResultSet");
        psResults.SetProperty("SearchId", responseObj.search_id || "");
        psResults.SetProperty("Query", responseObj.query || queryText);
        psResults.SetProperty("Timestamp", responseObj.timestamp || "");
        psResults.SetProperty("Count", recommendations.length.toString());
        
        // Add each recommendation as a child property set
        for (var i = 0; i < recommendations.length; i++)
        {
            var rec = recommendations[i];
            var psRec = TheApplication().NewPropertySet();
            psRec.SetType("Recommendation");
            psRec.SetProperty("Rank", rec.rank ? rec.rank.toString() : (i + 1).toString());
            psRec.SetProperty("CatalogItemId", rec.catalog_item_id || "");
            psRec.SetProperty("CatalogPath", rec.catalog_path || "");
            psRec.SetProperty("RelevanceScore", rec.relevance_score ? rec.relevance_score.toString() : "0");
            psRec.SetProperty("Frequency", rec.frequency ? rec.frequency.toString() : "0");
            psRec.SetProperty("MaxScore", rec.max_score ? rec.max_score.toString() : "0");
            psResults.AddChild(psRec);
        }
        
        // Return results
        Outputs.AddChild(psResults);
        
        TheApplication().Trace("SUCCESS: Returned " + recommendations.length + " recommendations");
    }
    catch (e)
    {
        var errorMsg = "Unexpected error in InvokeSearch: " + e.toString();
        Outputs.SetProperty("Error", errorMsg);
        TheApplication().Trace("ERROR: " + errorMsg);
        TheApplication().Trace("Stack trace: " + e.stack);
    }
    
    return (CancelOperation);
}
```

5. Save the eScript (Ctrl+S)
6. Compile the business service

---

### Step 3: Test Business Service

**Executor:** Siebel Developer  
**Duration:** 15 minutes

#### 3.1. Test via Siebel Client

1. Navigate to **Administration - Business Service > Business Service Simulator**
2. In the **Business Service** field, select `SemanticSearchAPIService`
3. In the **Method** field, select `InvokeSearch`
4. Click **Simulate**
5. In the input hierarchy, add:
   - `QueryText` = "My laptop screen is broken"
   - `TopK` = "5"
6. Click **Execute**
7. Verify the output contains:
   - `ResultSet` hierarchy with recommendations
   - No `Error` property

#### 3.2. Test via eScript in Browser Console

Enable Siebel tracing to see detailed logs:
1. Append `?SWECmd=SetTraceLevel&SWELevel=4` to your Siebel URL
2. Open browser console and check for trace messages
3. Look for "Calling Semantic Search API" and "SUCCESS" messages

---

### Step 4: Create Custom Presentation Model

**Executor:** Siebel Web Developer  
**Duration:** 60 minutes

#### 4.1. Create Custom PM File

Create file: `CatalogSearchPM.js` in your custom files directory:
`<SIEBEL_ROOT>/public/<LANGUAGE>/files/custom/`

```javascript
/**
 * Presentation Model: CatalogSearchPM
 * Purpose: Handle semantic search for service catalog items
 * Extends: Standard Applet PM
 */

if (typeof(SiebelAppFacade.CatalogSearchPM) === "undefined") {

    SiebelJS.Namespace("SiebelAppFacade.CatalogSearchPM");

    define("siebel/custom/CatalogSearchPM", ["siebel/jqgridrenderer"],
        function () {
            
            SiebelAppFacade.CatalogSearchPM = (function () {

                function CatalogSearchPM(pm) {
                    SiebelAppFacade.CatalogSearchPM.superclass.constructor.apply(this, arguments);
                }

                SiebelJS.Extend(CatalogSearchPM, SiebelAppFacade.PresentationModel);

                /**
                 * Initialize the PM
                 */
                CatalogSearchPM.prototype.Init = function () {
                    SiebelAppFacade.CatalogSearchPM.superclass.Init.apply(this, arguments);
                    
                    // Get references to applets
                    this.searchAppletName = this.Get("GetName");
                    this.resultsAppletName = "Service Request Catalog Results Applet"; // Update with actual name
                    
                    console.log("CatalogSearchPM initialized for: " + this.searchAppletName);
                };

                /**
                 * Setup event handlers
                 */
                CatalogSearchPM.prototype.Setup = function (propSet) {
                    SiebelAppFacade.CatalogSearchPM.superclass.Setup.apply(this, arguments);
                    
                    // Attach custom search button handler
                    var self = this;
                    var searchBtn = $("#s_" + this.Get("GetFullId") + "_search_button");
                    
                    if (searchBtn.length > 0) {
                        searchBtn.off("click").on("click", function (e) {
                            e.preventDefault();
                            self.ExecuteSemanticSearch();
                        });
                    }
                };

                /**
                 * Execute semantic search
                 */
                CatalogSearchPM.prototype.ExecuteSemanticSearch = function () {
                    var self = this;
                    
                    // Get search query from input control
                    var controlName = "SearchQuery"; // Update with actual control name
                    var query = this.ExecuteMethod("GetControlValue", controlName);
                    
                    if (!query || query.trim() === "") {
                        this.ShowErrorMessage("Please enter a search query");
                        return;
                    }
                    
                    console.log("Executing semantic search for: " + query);
                    
                    // Show loading indicator
                    this.ShowLoadingIndicator(true);
                    
                    // Clear previous results
                    this.ClearResultsApplet();
                    
                    // Prepare business service inputs
                    var psInputs = SiebelApp.S_App.NewPropertySet();
                    psInputs.SetProperty("QueryText", query);
                    psInputs.SetProperty("TopK", "5");
                    
                    // Call business service asynchronously
                    SiebelApp.S_App.GetService("SemanticSearchAPIService").InvokeMethod(
                        "InvokeSearch",
                        psInputs,
                        {
                            async: true,
                            scope: self,
                            cb: function (methodName, inputPS, outputPS) {
                                self.HandleSearchResponse(outputPS);
                            }
                        }
                    );
                };

                /**
                 * Handle API response
                 */
                CatalogSearchPM.prototype.HandleSearchResponse = function (outputPS) {
                    // Hide loading indicator
                    this.ShowLoadingIndicator(false);
                    
                    // Check for errors
                    var error = outputPS.GetProperty("Error");
                    if (error) {
                        console.error("Search error: " + error);
                        this.ShowErrorMessage("Search is temporarily unavailable. Please try again later.");
                        return;
                    }
                    
                    // Get result set
                    var resultSet = outputPS.GetChildByType("ResultSet");
                    if (!resultSet) {
                        console.warn("No result set returned");
                        this.ShowErrorMessage("No results found");
                        return;
                    }
                    
                    // Get recommendations
                    var recommendations = resultSet.GetChildren();
                    if (!recommendations || recommendations.length === 0) {
                        console.log("No recommendations found");
                        this.ShowErrorMessage("No relevant items found for your query");
                        return;
                    }
                    
                    console.log("Received " + recommendations.length + " recommendations");
                    
                    // Process recommendations
                    this.ApplySearchResults(recommendations);
                };

                /**
                 * Apply search results to results applet
                 */
                CatalogSearchPM.prototype.ApplySearchResults = function (recommendations) {
                    // Build search spec and sort spec
                    var idList = [];
                    var sortClauses = [];
                    
                    for (var i = 0; i < recommendations.length; i++) {
                        var rec = recommendations[i];
                        var id = rec.GetProperty("CatalogItemId");
                        var rank = rec.GetProperty("Rank");
                        
                        idList.push("'" + id + "'");
                        sortClauses.push("WHEN '" + id + "' THEN " + rank);
                    }
                    
                    // Build search specification
                    var searchSpec = "[Id] IN (" + idList.join(",") + ")";
                    
                    // Build sort specification (CRITICAL: apply ranking)
                    var sortSpec = "CASE [Id] " + sortClauses.join(" ") + " ELSE 99 END (ASCENDING)";
                    
                    console.log("SearchSpec: " + searchSpec);
                    console.log("SortSpec: " + sortSpec);
                    
                    // Get results applet PM
                    var resultsPM = SiebelApp.S_App.GetActivePM().Get(this.resultsAppletName);
                    if (!resultsPM) {
                        console.error("Results applet PM not found: " + this.resultsAppletName);
                        return;
                    }
                    
                    // Get business component
                    var bc = resultsPM.Get("GetBusComp");
                    if (!bc) {
                        console.error("Business component not found");
                        return;
                    }
                    
                    // Apply query
                    try {
                        bc.ClearToQuery();
                        bc.SetSortSpec(sortSpec); // IMPORTANT: Set sort spec BEFORE search spec
                        bc.SetSearchSpec(searchSpec);
                        bc.ExecuteQuery(SiebelApp.Constants.get("SWE_QUERY_FWD_ONLY"));
                        
                        console.log("Search results applied successfully");
                    } catch (e) {
                        console.error("Error applying search results: " + e.toString());
                        this.ShowErrorMessage("Error displaying results");
                    }
                };

                /**
                 * Clear results applet
                 */
                CatalogSearchPM.prototype.ClearResultsApplet = function () {
                    try {
                        var resultsPM = SiebelApp.S_App.GetActivePM().Get(this.resultsAppletName);
                        if (resultsPM) {
                            var bc = resultsPM.Get("GetBusComp");
                            if (bc) {
                                bc.ClearToQuery();
                                bc.ExecuteQuery(SiebelApp.Constants.get("SWE_QUERY_FWD_ONLY"));
                            }
                        }
                    } catch (e) {
                        console.warn("Could not clear results applet: " + e.toString());
                    }
                };

                /**
                 * Show/hide loading indicator
                 */
                CatalogSearchPM.prototype.ShowLoadingIndicator = function (show) {
                    var loadingDiv = $("#semantic-search-loading");
                    
                    if (show) {
                        if (loadingDiv.length === 0) {
                            // Create loading indicator
                            loadingDiv = $("<div>")
                                .attr("id", "semantic-search-loading")
                                .css({
                                    "position": "fixed",
                                    "top": "50%",
                                    "left": "50%",
                                    "transform": "translate(-50%, -50%)",
                                    "background": "rgba(0,0,0,0.7)",
                                    "color": "white",
                                    "padding": "20px",
                                    "border-radius": "5px",
                                    "z-index": "9999"
                                })
                                .html('<div class="siebui-icon-spinner"></div> Searching...');
                            $("body").append(loadingDiv);
                        }
                        loadingDiv.show();
                    } else {
                        loadingDiv.hide();
                    }
                };

                /**
                 * Show error message
                 */
                CatalogSearchPM.prototype.ShowErrorMessage = function (message) {
                    SiebelApp.S_App.NotifyUser(message);
                    console.error(message);
                };

                return CatalogSearchPM;
            }());

            return "SiebelAppFacade.CatalogSearchPM";
        }
    );
}
```

#### 4.2. Register Custom PM

Create manifest file: `manifest_search.js` in the same directory:

```javascript
/**
 * Manifest for Semantic Search customization
 */

if (typeof(SiebelApp.CustomPMMapping) === "undefined") {
    SiebelJS.Namespace("SiebelApp.CustomPMMapping");
}

SiebelApp.CustomPMMapping["Service Request Catalog Search Applet"] = "siebel/custom/CatalogSearchPM";

// Register the file
define("siebel/custom/manifest_search", [
    "siebel/custom/CatalogSearchPM"
], function () {
    return;
});
```

#### 4.3. Configure Web Template

1. Navigate to Siebel Tools
2. Go to **Web Template** object type
3. Find `manifest.js` template
4. Add reference to custom manifest:

```javascript
// Add to existing manifest.js
"siebel/custom/manifest_search"
```

5. Save and compile

---

### Step 5: Deploy to Siebel Web Server

**Executor:** Siebel Administrator  
**Duration:** 30 minutes

#### 5.1. Copy Custom Files

```bash
# Copy JavaScript files to web server
cp CatalogSearchPM.js <SIEBEL_ROOT>/public/enu/files/custom/
cp manifest_search.js <SIEBEL_ROOT>/public/enu/files/custom/

# Set proper permissions
chmod 644 <SIEBEL_ROOT>/public/enu/files/custom/*.js
```

#### 5.2. Generate and Deploy SRF

1. In Siebel Tools, go to **Tools > Check In**
2. Select all modified objects
3. Check in to repository
4. Generate new SRF file:
   ```bash
   srvrmgr -g gateway -e enterprise -u SADMIN -p password
   compile srf for server <server_name>
   ```

5. Deploy SRF to Siebel Servers:
   ```bash
   # Stop Siebel servers
   stop server <server_name> component SiebelServer
   
   # Copy SRF file
   cp siebel.srf <SIEBEL_ROOT>/siebsrvr/objects/
   
   # Start Siebel servers
   start server <server_name> component SiebelServer
   ```

#### 5.3. Clear Browser Cache

Clear all browser caches and restart application servers to ensure new JavaScript files are loaded.

---

### Step 6: Testing and Validation

**Executor:** QA Team  
**Duration:** 30 minutes

#### 6.1. Functional Testing

1. Log in to Siebel application
2. Navigate to Service Request screen
3. Enter search query: "My laptop is broken"
4. Click Search button
5. Verify:
   - Loading indicator appears
   - Results are displayed in ranked order
   - Clicking on result navigates to catalog item
   - Error handling works (try with empty query)

#### 6.2. Browser Console Testing

1. Open browser developer console (F12)
2. Perform a search
3. Look for console messages:
   - "Executing semantic search for: ..."
   - "Received X recommendations"
   - "Search results applied successfully"

4. Check for errors in console

#### 6.3. Network Traffic Analysis

1. Open browser Network tab
2. Perform a search
3. Verify API call:
   - URL matches ORDS endpoint
   - Method is POST
   - Headers include API key
   - Response is 200 OK
   - Response contains JSON with recommendations

---

## 5. Troubleshooting Guide

### 5.1. Common Issues

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Business service not found | SRF not deployed | Regenerate and deploy SRF |
| API call returns 401 | Invalid API key | Check Named Subsystem configuration |
| Results not displayed | Wrong applet name in PM | Verify results applet name matches |
| JavaScript errors | PM not registered | Check manifest registration |
| No loading indicator | Custom PM not loaded | Clear browser cache, check file path |

### 5.2. Debugging Commands

**Enable Siebel Tracing:**
```
URL?SWECmd=SetTraceLevel&SWELevel=5
```

**Check Business Service:**
```sql
SELECT NAME, PROJ_ID
FROM SIEBEL.S_SERVICE
WHERE NAME = 'SemanticSearchAPIService';
```

**View Siebel Server Logs:**
```bash
tail -f <SIEBEL_ROOT>/siebsrvr/log/SiebelServer_xxx.log | grep -i "semantic"
```

**Browser Console Commands:**
```javascript
// Check if PM is registered
SiebelApp.S_App.GetPM("Service Request Catalog Search Applet");

// Get Named Subsystem value
TheApplication().GetProfileAttr("SemanticSearchORDS.URL");

// Test business service directly
var psIn = SiebelApp.S_App.NewPropertySet();
psIn.SetProperty("QueryText", "test");
SiebelApp.S_App.GetService("SemanticSearchAPIService").InvokeMethod("InvokeSearch", psIn, {
    async: false
});
```

---

## 6. Performance Optimization

### 6.1. Caching Strategy

Consider caching frequent queries in Siebel:

```javascript
// Add to PM
CatalogSearchPM.prototype.searchCache = {};

CatalogSearchPM.prototype.ExecuteSemanticSearch = function () {
    var query = this.GetQueryText();
    var cacheKey = query.toLowerCase().trim();
    
    // Check cache
    if (this.searchCache[cacheKey]) {
        console.log("Using cached results for: " + query);
        this.ApplySearchResults(this.searchCache[cacheKey]);
        return;
    }
    
    // Continue with API call...
};
```

### 6.2. Connection Pooling

Configure EAI HTTP Transport for optimal performance:
1. Navigate to **Administration - Web Services > HTTP Configuration**
2. Set Max Connections: 20
3. Set Connection Timeout: 5000ms
4. Set Read Timeout: 10000ms

---

## 7. Maintenance Procedures

### 7.1. Update API Endpoint

```sql
-- Update URL if API endpoint changes
UPDATE S_SYS_PROF_PROP
SET VAL = 'http://new-server:8080/ords/semantic_search/siebel/search'
WHERE NAME = 'URL'
  AND PAR_ROW_ID = (SELECT ROW_ID FROM S_SYS_PROF WHERE NAME = 'SemanticSearchORDS');

COMMIT;
```

### 7.2. Monitor API Usage

Create monitoring query:

```sql
-- Check recent search activity from Siebel
SELECT 
    COUNT(*) AS search_count,
    TO_CHAR(request_timestamp, 'YYYY-MM-DD HH24:MI') AS time_bucket
FROM SEMANTIC_SEARCH.API_SEARCH_LOG
WHERE request_timestamp >= SYSDATE - 1
GROUP BY TO_CHAR(request_timestamp, 'YYYY-MM-DD HH24:MI')
ORDER BY time_bucket DESC;
```

---

## 8. Next Steps

Once this TDD is complete:
- Review **Deployment Guide** for complete deployment checklist
- Review **Testing Guide** for comprehensive test scenarios
- Schedule UAT with business users
- Plan go-live and rollback procedures

