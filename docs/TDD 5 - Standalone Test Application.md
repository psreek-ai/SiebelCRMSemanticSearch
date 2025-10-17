# Technical Design Document 5: Standalone Test Application

**Version:** 2.2
**Date:** 2025-10-17
**Owner:** Development Team / QA Team

## 1. Objective
This document provides the technical specification for a standalone Python-based test application that demonstrates and validates the semantic search functionality independently of Siebel CRM. The application provides a user-friendly interface to test search queries, visualize results, and analyze the recommended catalog paths.

## 2. Purpose and Benefits

### 2.1. Why Build a Test Application?

The standalone test application serves multiple critical purposes:

1. **Independent Testing**: Test the ORDS API without requiring full Siebel CRM setup
2. **Development Tool**: Enables rapid development and debugging of search algorithms
3. **Demo Platform**: Showcase semantic search capabilities to stakeholders
4. **Quality Assurance**: Validate search relevance before Siebel integration
5. **Performance Analysis**: Measure response times and analyze search patterns
6. **Training Tool**: Help business users understand semantic search behavior

### 2.2. Key Features

- 🔍 **Natural Language Search**: Enter queries in plain English
- 📊 **Side-by-Side Display**: View matching SRs and recommended catalog paths simultaneously
- 📈 **Similarity Scores**: See relevance scores for each result
- 🎯 **Interactive Filtering**: Filter by catalog category, date range, or similarity threshold
- 📉 **Analytics Dashboard**: Visualize search patterns and performance metrics
- 🔄 **Search History**: Track and replay previous searches
- 💾 **Export Capabilities**: Download results as CSV/JSON for analysis

## 3. Architecture Overview

### 3.1. Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **UI Framework** | Streamlit 1.28+ | Rapid web application development with Python |
| **API Client** | requests 2.31+ | HTTP client for ORDS REST API calls |
| **Data Processing** | pandas 2.1+ | Data manipulation and analysis |
| **Visualization** | plotly 5.18+ | Interactive charts and graphs |
| **Database (Optional)** | oracledb 1.4+ | Direct database connectivity for advanced queries |
| **Configuration** | python-dotenv 1.0+ | Environment variable management |

### 3.2. Application Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Streamlit Web UI                          │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │Search Input│  │Results Panel │  │Analytics Dashboard │  │
│  └────────────┘  └──────────────┘  └────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Python Application Logic                        │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │API Client    │  │Data Processor│  │Cache Manager    │  │
│  │Module        │  │              │  │                 │  │
│  └──────────────┘  └──────────────┘  └─────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          ▼                               ▼
┌─────────────────────┐         ┌──────────────────────┐
│  ORDS REST API      │         │  Oracle Autonomous   │
│  (Pre-configured)   │         │  Database on Azure   │
│  (Primary Mode)     │         │  (Optional Direct)   │
└─────────────────────┘         └──────────────────────┘
```

## 4. Application Components

### 4.1. Module Structure

```
test-app/
├── app.py                      # Main Streamlit application
├── config.py                   # Configuration management
├── requirements.txt            # Python dependencies
├── .env.example               # Environment variable template
├── README.md                  # Application documentation
│
├── api/
│   ├── __init__.py
│   ├── ords_client.py        # ORDS API client
│   └── db_client.py          # Direct DB client (optional)
│
├── components/
│   ├── __init__.py
│   ├── search_panel.py       # Search input component
│   ├── results_panel.py      # Results display component
│   ├── analytics_panel.py    # Analytics dashboard
│   └── sidebar.py            # Application sidebar
│
├── utils/
│   ├── __init__.py
│   ├── data_processor.py     # Data transformation utilities
│   ├── cache_manager.py      # Caching logic
│   └── logger.py             # Logging utilities
│
└── tests/
    ├── __init__.py
    ├── test_api.py           # API client tests
    └── test_utils.py         # Utility function tests
```

## 5. Detailed Component Specifications

### 5.1. Main Application (app.py)

**Purpose**: Entry point for Streamlit application

**Key Functions**:
```python
def main():
    """Main application entry point"""
    # Initialize session state
    # Render page layout
    # Handle user interactions
    pass

def initialize_session_state():
    """Initialize Streamlit session state variables"""
    # search_history: List of previous searches
    # current_results: Current search results
    # selected_sr: Currently selected SR for detail view
    pass

def render_header():
    """Render application header and title"""
    pass

def render_main_content():
    """Render main application content area"""
    # Left panel: Search results
    # Right panel: Catalog recommendations
    pass
```

**Layout Structure**:
```
┌─────────────────────────────────────────────────────────────┐
│  🔍 Siebel CRM Semantic Search - Test Application           │
├─────────────────────────────────────────────────────────────┤
│  Search Query: [_________________________________] [Search]  │
│  Top-K: [5▼]  Response Time: 1.23s  Results: 25            │
├──────────────────────────┬──────────────────────────────────┤
│  Matching Service        │  Recommended Catalog Paths       │
│  Requests (Left Panel)   │  (Right Panel)                   │
│  ┌────────────────────┐  │  ┌────────────────────────────┐ │
│  │ SR #12345          │  │  │ 📁 IT Support > Hardware   │ │
│  │ Score: 0.92        │  │  │    Count: 15 (35%)         │ │
│  │ "Laptop broken..." │  │  │                            │ │
│  │ ━━━━━━━━━━━━━━━━━ │  │  │ 📁 Hardware > Laptop       │ │
│  │                    │  │  │    Count: 12 (28%)         │ │
│  │ SR #12346          │  │  │                            │ │
│  │ Score: 0.89        │  │  │ 📁 IT Support > Repairs    │ │
│  │ "Screen not..."    │  │  │    Count: 8 (19%)          │ │
│  └────────────────────┘  │  └────────────────────────────┘ │
└──────────────────────────┴──────────────────────────────────┘
```

### 5.2. Configuration Module (config.py)

**Purpose**: Centralized configuration management

**Configuration Parameters**:

```python
class Config:
    """Application configuration"""
    
    # ORDS API Configuration
    ORDS_BASE_URL: str          # Base URL for ORDS endpoint
    ORDS_SCHEMA: str            # Database schema name
    ORDS_MODULE: str            # ORDS module name
    API_KEY: str                # API authentication key
    
    # Oracle Database Configuration (Optional)
    DB_USER: str                # Database username
    DB_PASSWORD: str            # Database password
    DB_HOST: str                # Database hostname
    DB_PORT: int                # Database port (default: 1521)
    DB_SERVICE: str             # Database service name
    
    # Application Settings
    DEFAULT_TOP_K: int          # Default number of results (default: 5)
    MAX_TOP_K: int              # Maximum allowed Top-K (default: 20)
    CACHE_ENABLED: bool         # Enable result caching
    CACHE_TTL: int              # Cache time-to-live in seconds
    ENABLE_DATABASE_MODE: bool  # Allow direct DB queries
    LOG_LEVEL: str              # Logging level (DEBUG, INFO, WARN, ERROR)
    
    # UI Settings
    PAGE_TITLE: str             # Browser page title
    PAGE_ICON: str              # Browser favicon
    LAYOUT: str                 # Streamlit layout (wide/centered)
    THEME: dict                 # Custom theme colors
```

**Environment Variables** (.env file):
```ini
# API Configuration
ORDS_BASE_URL=http://localhost:8080/ords
ORDS_SCHEMA=semantic_search
ORDS_MODULE=siebel
API_KEY=your_secure_api_key_here

# Database Configuration (Optional)
DB_USER=SEMANTIC_SEARCH
DB_PASSWORD=your_password_here
DB_HOST=localhost
DB_PORT=1521
DB_SERVICE=vector_db_service

# Application Settings
DEFAULT_TOP_K=5
MAX_TOP_K=20
CACHE_ENABLED=true
CACHE_TTL=300
ENABLE_DATABASE_MODE=false
LOG_LEVEL=INFO

# UI Settings
PAGE_TITLE=Siebel Semantic Search Test
PAGE_ICON=🔍
LAYOUT=wide
```

### 5.3. ORDS API Client (api/ords_client.py)

**Purpose**: Handle all communication with ORDS REST API

**Class Definition**:

```python
class ORDSClient:
    """Client for ORDS REST API communication"""
    
    def __init__(self, base_url: str, api_key: str):
        """Initialize ORDS client
        
        Args:
            base_url: Base URL for ORDS endpoint
            api_key: API authentication key
        """
        self.base_url = base_url
        self.api_key = api_key
        self.session = requests.Session()
        self.session.headers.update({
            'X-API-Key': api_key,
            'Content-Type': 'text/plain'
        })
    
    def search(self, query: str, top_k: int = 5) -> dict:
        """Execute semantic search
        
        Args:
            query: Natural language search query
            top_k: Number of recommendations to return
            
        Returns:
            dict: Search results with structure:
                {
                    'recommendations': [
                        {
                            'catalog_path': str,
                            'count': int,
                            'percentage': float,
                            'sample_sr_ids': [str]
                        }
                    ],
                    'matching_srs': [
                        {
                            'sr_id': str,
                            'narrative': str,
                            'similarity_score': float,
                            'catalog_path': str,
                            'created_date': str
                        }
                    ],
                    'query_text': str,
                    'response_time_ms': int,
                    'total_matches': int
                }
        
        Raises:
            APIError: If API call fails
        """
        pass
    
    def health_check(self) -> bool:
        """Check API health and connectivity
        
        Returns:
            bool: True if API is accessible
        """
        pass
    
    def get_statistics(self) -> dict:
        """Get API usage statistics
        
        Returns:
            dict: Statistics including total vectors, avg response time
        """
        pass
```

**API Request Format**:
```http
POST /ords/semantic_search/siebel/search
Content-Type: text/plain
X-API-Key: your_api_key
Top-K: 5

My laptop screen is broken and flickering
```

**API Response Format**:
```json
{
    "recommendations": [
        {
            "catalog_path": "IT Support > Hardware > Laptop",
            "count": 15,
            "percentage": 35.5,
            "sample_sr_ids": ["SR-001", "SR-002", "SR-003"]
        },
        {
            "catalog_path": "Hardware > Display Issues",
            "count": 12,
            "percentage": 28.3,
            "sample_sr_ids": ["SR-004", "SR-005"]
        }
    ],
    "matching_srs": [
        {
            "sr_id": "SR-12345",
            "narrative": "Laptop screen broken, flickering display...",
            "similarity_score": 0.923,
            "catalog_path": "IT Support > Hardware > Laptop",
            "created_date": "2024-10-15"
        }
    ],
    "query_text": "My laptop screen is broken and flickering",
    "response_time_ms": 1234,
    "total_matches": 42
}
```

### 5.4. Direct Database Client (api/db_client.py)

**Purpose**: Optional direct database connectivity for advanced queries

**Class Definition**:

```python
class DatabaseClient:
    """Direct Oracle database client for advanced queries"""
    
    def __init__(self, user: str, password: str, dsn: str):
        """Initialize database connection
        
        Args:
            user: Database username
            password: Database password
            dsn: Database DSN (host:port/service)
        """
        self.connection = oracledb.connect(
            user=user,
            password=password,
            dsn=dsn
        )
    
    def get_vector_count(self) -> int:
        """Get total count of vectors in database"""
        pass
    
    def get_sr_details(self, sr_id: str) -> dict:
        """Get full details for a specific SR
        
        Args:
            sr_id: Service Request ID
            
        Returns:
            dict: Full SR details including all activities
        """
        pass
    
    def get_catalog_tree(self) -> dict:
        """Get complete catalog hierarchy
        
        Returns:
            dict: Nested catalog structure
        """
        pass
    
    def search_direct(self, query: str, top_k: int) -> list:
        """Execute search directly against database
        
        This bypasses the ORDS API and calls the PL/SQL
        procedure directly for debugging purposes.
        """
        pass
```

### 5.5. Search Panel Component (components/search_panel.py)

**Purpose**: Render search input and controls

**Key Functions**:

```python
def render_search_panel() -> tuple[str, int]:
    """Render search input panel
    
    Returns:
        tuple: (query_text, top_k)
    """
    st.subheader("🔍 Search Query")
    
    # Text input for search query
    query = st.text_area(
        "Enter your search query:",
        placeholder="e.g., My laptop screen is broken",
        height=100,
        help="Enter a natural language description of the issue"
    )
    
    # Search controls
    col1, col2, col3 = st.columns([2, 1, 1])
    
    with col1:
        top_k = st.slider(
            "Number of recommendations:",
            min_value=1,
            max_value=20,
            value=5,
            help="How many catalog recommendations to return"
        )
    
    with col2:
        search_button = st.button(
            "🔍 Search",
            type="primary",
            use_container_width=True
        )
    
    with col3:
        clear_button = st.button(
            "🗑️ Clear",
            use_container_width=True
        )
    
    return query, top_k, search_button, clear_button
```

### 5.6. Results Panel Component (components/results_panel.py)

**Purpose**: Display search results in two-column layout

**Key Functions**:

```python
def render_results_panel(results: dict):
    """Render search results in two-column layout
    
    Args:
        results: Search results from API
    """
    # Display summary metrics
    render_metrics(results)
    
    # Create two columns
    col_left, col_right = st.columns([1, 1])
    
    with col_left:
        render_matching_srs(results['matching_srs'])
    
    with col_right:
        render_catalog_recommendations(results['recommendations'])

def render_metrics(results: dict):
    """Render search metrics in columns"""
    col1, col2, col3, col4 = st.columns(4)
    
    with col1:
        st.metric(
            "Response Time",
            f"{results['response_time_ms']}ms"
        )
    
    with col2:
        st.metric(
            "Total Matches",
            results['total_matches']
        )
    
    with col3:
        st.metric(
            "Recommendations",
            len(results['recommendations'])
        )
    
    with col4:
        avg_score = calculate_avg_similarity(results['matching_srs'])
        st.metric(
            "Avg Similarity",
            f"{avg_score:.2%}"
        )

def render_matching_srs(matching_srs: list):
    """Render list of matching service requests
    
    Args:
        matching_srs: List of matching SR dictionaries
    """
    st.subheader("📋 Matching Service Requests")
    
    for sr in matching_srs:
        with st.expander(
            f"**{sr['sr_id']}** - Similarity: {sr['similarity_score']:.2%}",
            expanded=False
        ):
            st.write(f"**Catalog Path:** {sr['catalog_path']}")
            st.write(f"**Created:** {sr['created_date']}")
            st.write(f"**Narrative:**")
            st.text(sr['narrative'][:500] + "..." if len(sr['narrative']) > 500 else sr['narrative'])
            
            # Similarity score bar
            st.progress(sr['similarity_score'])

def render_catalog_recommendations(recommendations: list):
    """Render catalog path recommendations
    
    Args:
        recommendations: List of recommendation dictionaries
    """
    st.subheader("🎯 Recommended Catalog Paths")
    
    for idx, rec in enumerate(recommendations, 1):
        # Create container for each recommendation
        with st.container():
            col1, col2 = st.columns([3, 1])
            
            with col1:
                st.write(f"**{idx}. {rec['catalog_path']}**")
            
            with col2:
                st.write(f"**{rec['count']}** ({rec['percentage']:.1f}%)")
            
            # Progress bar showing percentage
            st.progress(rec['percentage'] / 100)
            
            # Sample SR IDs
            with st.expander("View sample SRs"):
                st.write(", ".join(rec['sample_sr_ids']))
            
            st.divider()
```

### 5.7. Analytics Panel Component (components/analytics_panel.py)

**Purpose**: Visualize search analytics and patterns

**Key Visualizations**:

1. **Similarity Score Distribution**
```python
def render_similarity_distribution(matching_srs: list):
    """Plot distribution of similarity scores"""
    import plotly.express as px
    
    scores = [sr['similarity_score'] for sr in matching_srs]
    
    fig = px.histogram(
        x=scores,
        nbins=20,
        title="Similarity Score Distribution",
        labels={'x': 'Similarity Score', 'y': 'Count'}
    )
    
    st.plotly_chart(fig, use_container_width=True)
```

2. **Catalog Path Frequency**
```python
def render_catalog_frequency(recommendations: list):
    """Plot catalog path frequencies as bar chart"""
    import plotly.express as px
    
    df = pd.DataFrame(recommendations)
    
    fig = px.bar(
        df,
        x='count',
        y='catalog_path',
        orientation='h',
        title="Recommended Catalog Paths",
        labels={'count': 'Frequency', 'catalog_path': 'Catalog Path'}
    )
    
    st.plotly_chart(fig, use_container_width=True)
```

3. **Response Time Trend**
```python
def render_response_time_trend(search_history: list):
    """Plot response time trend over search history"""
    import plotly.graph_objects as go
    
    timestamps = [s['timestamp'] for s in search_history]
    response_times = [s['response_time_ms'] for s in search_history]
    
    fig = go.Figure()
    fig.add_trace(go.Scatter(
        x=timestamps,
        y=response_times,
        mode='lines+markers',
        name='Response Time'
    ))
    
    fig.add_hline(y=3000, line_dash="dash", line_color="red",
                  annotation_text="3s Target")
    
    fig.update_layout(
        title="Response Time Trend",
        xaxis_title="Timestamp",
        yaxis_title="Response Time (ms)"
    )
    
    st.plotly_chart(fig, use_container_width=True)
```

### 5.8. Data Processor Utility (utils/data_processor.py)

**Purpose**: Transform and process API responses

**Key Functions**:

```python
def process_api_response(raw_response: dict) -> dict:
    """Process and enrich API response
    
    Args:
        raw_response: Raw response from API
        
    Returns:
        dict: Processed and enriched response
    """
    # Add calculated fields
    # Normalize data structures
    # Handle missing values
    pass

def calculate_statistics(results: dict) -> dict:
    """Calculate aggregate statistics from results
    
    Returns:
        dict: {
            'avg_similarity': float,
            'median_similarity': float,
            'std_similarity': float,
            'total_unique_catalogs': int,
            'coverage_percentage': float
        }
    """
    pass

def export_to_csv(results: dict, filename: str):
    """Export results to CSV file"""
    pass

def export_to_json(results: dict, filename: str):
    """Export results to JSON file"""
    pass
```

### 5.9. Cache Manager Utility (utils/cache_manager.py)

**Purpose**: Implement result caching to improve performance

**Implementation**:

```python
class CacheManager:
    """Manage search result caching"""
    
    def __init__(self, ttl: int = 300):
        """Initialize cache with TTL
        
        Args:
            ttl: Time-to-live in seconds
        """
        self.cache = {}
        self.ttl = ttl
    
    def get(self, query: str, top_k: int) -> Optional[dict]:
        """Retrieve cached result
        
        Args:
            query: Search query
            top_k: Number of results
            
        Returns:
            Cached result or None if not found/expired
        """
        cache_key = self._generate_key(query, top_k)
        
        if cache_key in self.cache:
            entry = self.cache[cache_key]
            
            # Check if expired
            if time.time() - entry['timestamp'] < self.ttl:
                return entry['data']
            else:
                # Remove expired entry
                del self.cache[cache_key]
        
        return None
    
    def set(self, query: str, top_k: int, data: dict):
        """Store result in cache"""
        cache_key = self._generate_key(query, top_k)
        
        self.cache[cache_key] = {
            'data': data,
            'timestamp': time.time()
        }
    
    def clear(self):
        """Clear all cached entries"""
        self.cache = {}
    
    def _generate_key(self, query: str, top_k: int) -> str:
        """Generate cache key from query and parameters"""
        import hashlib
        key_string = f"{query.lower().strip()}|{top_k}"
        return hashlib.md5(key_string.encode()).hexdigest()
```

## 6. Application Workflows

### 6.1. Standard Search Workflow

```
User enters query → Click Search button
        ↓
Validate input (non-empty, reasonable length)
        ↓
Check cache for existing results
        ↓
If cached: Return cached results
        ↓
If not cached:
    → Call ORDS API with query + top_k
    → Wait for response (show spinner)
    → Process response
    → Store in cache
        ↓
Update session state with results
        ↓
Render results in two-column layout
        ↓
Update analytics panel
        ↓
Add to search history
```

### 6.2. Search History Replay Workflow

```
User clicks search in history list
        ↓
Load query and parameters from history
        ↓
Populate search input fields
        ↓
Execute standard search workflow
        ↓
Highlight that this is a replayed search
```

### 6.3. Export Results Workflow

```
User clicks "Export Results" button
        ↓
Show export format dialog (CSV/JSON)
        ↓
User selects format and confirms
        ↓
Process current results
        ↓
Generate file in selected format
        ↓
Trigger browser download
```

## 7. Error Handling

### 7.1. Error Scenarios

| Error Type | Handling Strategy | User Message |
|------------|------------------|--------------|
| **API Unreachable** | Retry with exponential backoff (3 attempts) | "Unable to connect to search service. Please check your network connection." |
| **Invalid API Key** | Stop and prompt for configuration | "Authentication failed. Please check your API key in .env file." |
| **Empty Query** | Prevent search, show validation message | "Please enter a search query." |
| **Timeout** | Cancel request after 30s | "Search request timed out. Please try again with a simpler query." |
| **Invalid Response** | Log error, show generic message | "Received invalid response from server. Please contact support." |
| **Rate Limit** | Show cooldown timer | "Rate limit exceeded. Please wait {X} seconds before searching again." |

### 7.2. Error Display Component

```python
def display_error(error_type: str, message: str, details: Optional[str] = None):
    """Display error message to user
    
    Args:
        error_type: Type of error (error, warning, info)
        message: User-friendly error message
        details: Technical details for debugging
    """
    if error_type == "error":
        st.error(message)
    elif error_type == "warning":
        st.warning(message)
    else:
        st.info(message)
    
    if details and st.session_state.get('debug_mode', False):
        with st.expander("Technical Details"):
            st.code(details)
```

## 8. Configuration and Setup

### 8.1. Installation Steps

**Step 1: Clone Repository**
```bash
cd test-app
```

**Step 2: Create Virtual Environment**
```bash
# Windows
python -m venv venv
venv\Scripts\activate

# Linux/Mac
python3 -m venv venv
source venv/bin/activate
```

**Step 3: Install Dependencies**
```bash
pip install -r requirements.txt
```

**Step 4: Configure Environment**
```bash
# Copy example configuration
copy .env.example .env

# Edit .env with your values
notepad .env
```

**Step 5: Run Application**
```bash
streamlit run app.py
```

### 8.2. Configuration Checklist

Before running the application:

- [ ] ORDS_BASE_URL configured (e.g., `http://localhost:8080/ords`)
- [ ] ORDS_SCHEMA set to `semantic_search`
- [ ] ORDS_MODULE set to `siebel`
- [ ] API_KEY obtained and configured
- [ ] (Optional) Database credentials configured if using direct mode
- [ ] DEFAULT_TOP_K set to appropriate value (recommended: 5)
- [ ] CACHE_ENABLED set based on testing needs
- [ ] LOG_LEVEL set appropriately (INFO for normal use, DEBUG for troubleshooting)

### 8.3. Verification Steps

After starting the application:

1. **Verify API Connectivity**
   - Check sidebar for connection status indicator
   - Should show green "✅ Connected" if API is reachable

2. **Test Basic Search**
   - Enter query: "laptop screen broken"
   - Click Search
   - Verify results appear in < 3 seconds

3. **Verify Result Display**
   - Check left panel shows matching SRs with similarity scores
   - Check right panel shows catalog recommendations with counts
   - Verify percentages add up correctly

4. **Test Error Handling**
   - Try empty query → Should show validation message
   - Try disconnecting network → Should show connection error

## 9. Testing the Test Application

### 9.1. Unit Tests

Create tests in `tests/` directory:

```python
# tests/test_api.py
import pytest
from api.ords_client import ORDSClient

def test_search_valid_query():
    """Test API client with valid query"""
    client = ORDSClient(base_url="http://localhost:8080/ords", api_key="test_key")
    result = client.search("laptop broken", top_k=5)
    
    assert 'recommendations' in result
    assert 'matching_srs' in result
    assert len(result['recommendations']) <= 5

def test_search_empty_query():
    """Test API client with empty query"""
    client = ORDSClient(base_url="http://localhost:8080/ords", api_key="test_key")
    
    with pytest.raises(ValueError):
        client.search("", top_k=5)

def test_health_check():
    """Test API health check"""
    client = ORDSClient(base_url="http://localhost:8080/ords", api_key="test_key")
    
    assert client.health_check() == True
```

Run tests:
```bash
pytest tests/ -v
```

### 9.2. Integration Tests

```python
# tests/test_integration.py
def test_end_to_end_search():
    """Test complete search workflow"""
    # Initialize app components
    # Execute search
    # Verify results displayed correctly
    # Verify analytics updated
    # Verify search history updated
    pass
```

## 10. Usage Scenarios

### 10.1. Developer Testing

**Scenario**: Developer wants to test new embedding model

1. Start application with API pointing to test environment
2. Execute standard test queries
3. Compare results with baseline
4. Export results to CSV for analysis
5. Use analytics panel to compare similarity score distributions

### 10.2. QA Validation

**Scenario**: QA needs to validate search relevance

1. Load golden set test queries from CSV
2. Execute each query systematically
3. Record top 3 recommendations for each query
4. Compare against expected results
5. Calculate Precision@3 metric
6. Export results for reporting

### 10.3. Business Demo

**Scenario**: Demonstrate semantic search to stakeholders

1. Start application with production-like data
2. Use realistic business queries
3. Show side-by-side comparison of queries
4. Highlight similarity scores and relevance
5. Show analytics dashboard with trends
6. Demonstrate search history replay

### 10.4. Performance Testing

**Scenario**: Validate response time under load

1. Enable database mode for direct queries
2. Execute series of searches with timestamps
3. Use analytics panel to view response time trend
4. Identify slow queries
5. Export timing data for analysis

## 11. Advanced Features

### 11.1. Batch Query Testing

**Feature**: Test multiple queries from file

```python
def batch_test_from_file(filepath: str) -> pd.DataFrame:
    """Execute batch queries from CSV file
    
    Args:
        filepath: Path to CSV with columns: query, expected_category
        
    Returns:
        DataFrame with results and success/fail indicators
    """
    # Read queries from CSV
    # Execute each query
    # Compare results with expected
    # Generate summary report
    pass
```

**UI Implementation**:
- Add "Batch Testing" tab in sidebar
- File uploader for CSV input
- Progress bar for batch execution
- Results table with pass/fail indicators
- Download button for results

### 11.2. A/B Comparison Mode

**Feature**: Compare results from two different configurations

```python
def compare_configurations(query: str, config_a: dict, config_b: dict) -> dict:
    """Compare search results from two configurations
    
    Returns:
        dict: Side-by-side comparison metrics
    """
    # Execute search with config A
    # Execute search with config B
    # Calculate comparison metrics
    # Identify differences
    pass
```

**UI Layout**:
```
┌─────────────────────────────────────────────────────┐
│  Query: "laptop broken"                             │
├─────────────────────────┬───────────────────────────┤
│  Configuration A        │  Configuration B          │
│  (Current Production)   │  (Test Environment)       │
├─────────────────────────┼───────────────────────────┤
│  Response: 1.2s         │  Response: 0.8s ⚡       │
│  Top Result:            │  Top Result:              │
│  IT > Hardware (35%)    │  Hardware > Laptop (42%) │
│  ...                    │  ...                      │
└─────────────────────────┴───────────────────────────┘
```

### 11.3. Search History Persistence

**Feature**: Save search history to SQLite database

```python
class SearchHistoryManager:
    """Manage persistent search history"""
    
    def save_search(self, query: str, results: dict, timestamp: datetime):
        """Save search to database"""
        pass
    
    def load_history(self, limit: int = 100) -> list:
        """Load recent searches from database"""
        pass
    
    def export_history(self, filepath: str, format: str):
        """Export search history to file"""
        pass
```

### 11.4. Catalog Path Explorer

**Feature**: Browse full catalog hierarchy

```python
def render_catalog_explorer():
    """Render interactive catalog tree view"""
    # Query database for full catalog structure
    # Display as expandable tree
    # Allow filtering and search
    # Show SR counts at each level
    pass
```

## 12. Performance Optimization

### 12.1. Caching Strategy

- Cache API responses for 5 minutes (configurable)
- Use query + top_k as cache key
- Implement LRU eviction if cache grows too large
- Provide manual cache clear button

### 12.2. Lazy Loading

- Load SR details on-demand when expanded
- Implement pagination for large result sets
- Use Streamlit's `@st.cache_data` decorator for data processing functions

### 12.3. Async API Calls

```python
async def search_async(query: str, top_k: int) -> dict:
    """Asynchronous search request"""
    # Use aiohttp for non-blocking API calls
    # Implement concurrent requests for batch testing
    pass
```

## 13. Security Considerations

### 13.1. API Key Protection

- Store API key in .env file (never commit to git)
- Add .env to .gitignore
- Mask API key in UI (show only last 4 characters)
- Implement key rotation support

### 13.2. Input Validation

```python
def validate_query(query: str) -> tuple[bool, str]:
    """Validate user input
    
    Returns:
        tuple: (is_valid, error_message)
    """
    if not query or query.strip() == "":
        return False, "Query cannot be empty"
    
    if len(query) > 5000:
        return False, "Query is too long (max 5000 characters)"
    
    # Check for SQL injection patterns
    dangerous_patterns = ["DROP", "DELETE", "UPDATE", "INSERT", "--", ";"]
    if any(pattern in query.upper() for pattern in dangerous_patterns):
        return False, "Query contains invalid characters"
    
    return True, ""
```

### 13.3. Rate Limiting

```python
class RateLimiter:
    """Client-side rate limiting"""
    
    def __init__(self, max_requests: int, time_window: int):
        """Initialize rate limiter
        
        Args:
            max_requests: Max requests allowed
            time_window: Time window in seconds
        """
        self.max_requests = max_requests
        self.time_window = time_window
        self.requests = []
    
    def can_proceed(self) -> tuple[bool, int]:
        """Check if request can proceed
        
        Returns:
            tuple: (can_proceed, wait_seconds)
        """
        now = time.time()
        
        # Remove old requests outside time window
        self.requests = [r for r in self.requests if now - r < self.time_window]
        
        if len(self.requests) < self.max_requests:
            self.requests.append(now)
            return True, 0
        else:
            wait_time = self.time_window - (now - self.requests[0])
            return False, int(wait_time) + 1
```

## 14. Troubleshooting Guide

### 14.1. Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **"Connection refused"** | ORDS not running | Start ORDS service: `ords serve` |
| **"Invalid API key"** | Wrong or missing key | Verify API_KEY in .env matches registered key |
| **"Module not found"** | Missing dependencies | Run `pip install -r requirements.txt` |
| **Slow response times** | Network latency | Check network connection, try direct DB mode |
| **Empty results** | No matching vectors | Verify vector index populated, check query |
| **UI not updating** | Session state issue | Refresh browser, clear Streamlit cache |

### 14.2. Debug Mode

Enable debug mode in sidebar:

```python
if st.sidebar.checkbox("🐛 Debug Mode"):
    st.session_state.debug_mode = True
    
    # Show additional debug information
    with st.expander("Debug Info"):
        st.json({
            'session_state': dict(st.session_state),
            'config': config.__dict__,
            'cache_stats': cache_manager.get_stats()
        })
```

### 14.3. Logging

Configure logging in `utils/logger.py`:

```python
import logging

def setup_logger(name: str, level: str = "INFO") -> logging.Logger:
    """Setup application logger"""
    logger = logging.getLogger(name)
    logger.setLevel(level)
    
    # Console handler
    handler = logging.StreamHandler()
    formatter = logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    )
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    
    return logger
```

## 15. Deployment

### 15.1. Standalone Deployment

Run locally for testing:
```bash
streamlit run app.py --server.port 8501
```

### 15.2. Docker Deployment

Create `Dockerfile`:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8501

CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

Build and run:
```bash
docker build -t semantic-search-test-app .
docker run -p 8501:8501 --env-file .env semantic-search-test-app
```

### 15.3. Cloud Deployment

Deploy to Streamlit Cloud:
1. Push code to GitHub repository
2. Connect repository to Streamlit Cloud
3. Configure secrets (API_KEY, DB_PASSWORD, etc.)
4. Deploy with one click

## 16. Future Enhancements

### 16.1. Planned Features

- [ ] **Multi-language Support**: Support for non-English queries
- [ ] **Voice Input**: Allow voice-based search queries
- [ ] **Result Explanation**: Show why specific results were returned
- [ ] **Custom Metrics**: Allow users to define custom relevance metrics
- [ ] **Result Feedback**: Thumbs up/down for result relevance
- [ ] **Query Suggestions**: Autocomplete based on common queries
- [ ] **Dashboard**: Real-time monitoring dashboard for production

### 16.2. Integration Possibilities

- Export to PowerBI/Tableau for advanced analytics
- Integrate with Jira for issue tracking
- Connect to Slack for search notifications
- REST API wrapper for automated testing

---

## 17. Appendix

### 17.1. Complete requirements.txt

```txt
# Core Framework
streamlit==1.28.1

# API Client
requests==2.31.0
aiohttp==3.9.0

# Data Processing
pandas==2.1.3
numpy==1.26.2

# Visualization
plotly==5.18.0
matplotlib==3.8.2

# Database (Optional)
oracledb==1.4.2

# Configuration
python-dotenv==1.0.0

# Utilities
python-dateutil==2.8.2

# Testing
pytest==7.4.3
pytest-asyncio==0.21.1
pytest-mock==3.12.0

# Logging
coloredlogs==15.0.1
```

### 17.2. Sample .env File

```ini
# ORDS API Configuration
ORDS_BASE_URL=http://localhost:8080/ords
ORDS_SCHEMA=semantic_search
ORDS_MODULE=siebel
API_KEY=sk_test_1234567890abcdef

# Oracle Database Configuration (Optional - for direct mode)
DB_USER=SEMANTIC_SEARCH
DB_PASSWORD=SecurePassword123!
DB_HOST=localhost
DB_PORT=1521
DB_SERVICE=VECTOR_DB

# Application Settings
DEFAULT_TOP_K=5
MAX_TOP_K=20
CACHE_ENABLED=true
CACHE_TTL=300
ENABLE_DATABASE_MODE=false
LOG_LEVEL=INFO

# UI Settings
PAGE_TITLE=Siebel Semantic Search - Test Application
PAGE_ICON=🔍
LAYOUT=wide

# Rate Limiting
MAX_REQUESTS_PER_MINUTE=60

# Feature Flags
ENABLE_BATCH_TESTING=true
ENABLE_ANALYTICS=true
ENABLE_EXPORT=true
ENABLE_DEBUG_MODE=true
```

### 17.3. Sample Test Queries

```csv
query,expected_category,description
"My laptop screen is broken","IT Support > Hardware > Laptop",Hardware issue
"Need to reset my password","IT Support > Account Management",Account access
"Printer not working","Hardware > Office Equipment > Printer",Office equipment
"Request access to SAP","IT Support > Application Access",Application access
"Conference room projector broken","Facilities > Meeting Rooms",Facilities issue
"Need new desk chair","Facilities > Furniture",Office furniture
"Outlook not syncing emails","IT Support > Email",Email issue
"VPN connection keeps dropping","IT Support > Network",Network connectivity
"Need software license for Adobe","IT Support > Software Licensing",Software request
"Office AC not working","Facilities > HVAC",Environmental issue
```

### 17.4. Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl + Enter` | Execute search |
| `Ctrl + K` | Focus search input |
| `Ctrl + L` | Clear search |
| `Ctrl + H` | Toggle search history |
| `Ctrl + E` | Export results |
| `Ctrl + R` | Refresh data |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-10-17 | Development Team | Initial version |
| 2.0 | 2025-10-17 | Development Team | Added advanced features and deployment |
| 2.2 | 2025-10-17 | Development Team | Comprehensive rewrite with full implementation details |

**Next Review Date:** 2025-11-17
**Document Owner:** Development Team / QA Team
**Approval Required From:** Technical Lead, QA Lead, Business Owner
