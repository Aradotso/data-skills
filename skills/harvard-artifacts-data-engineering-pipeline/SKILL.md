---
name: harvard-artifacts-data-engineering-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline from Harvard Art Museums API
  - create ETL workflow for museum artifact data
  - set up Harvard artifacts analytics dashboard
  - extract and analyze Harvard Art Museums collection data
  - build Streamlit app for art museum data visualization
  - implement SQL analytics for Harvard artifacts
  - create museum data engineering pipeline
  - analyze Harvard Art Museums API data
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with the Harvard-Artifacts-Collection-Data-Engineering-Analytics-App, an end-to-end data engineering solution that extracts artifact data from the Harvard Art Museums API, performs ETL operations, stores data in SQL databases, and provides interactive analytics dashboards using Streamlit.

## What This Project Does

The Harvard Artifacts Data Engineering Pipeline demonstrates a complete data engineering workflow:

1. **API Integration**: Connects to Harvard Art Museums API with proper authentication and pagination
2. **ETL Pipeline**: Extracts artifact metadata, transforms nested JSON into relational format, loads into SQL
3. **SQL Analytics**: Runs 20+ predefined analytical queries on artifact collections
4. **Interactive Visualization**: Displays results through Streamlit dashboards with Plotly charts

The pipeline handles artifact metadata, media/image information, and color data across three normalized database tables.

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud database instance
- Harvard Art Museums API key (get from: https://www.harvardartmuseums.org/collections/api)

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Configuration

Create a `.env` file or use Streamlit secrets for configuration:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Or configure via Streamlit secrets (`.streamlit/secrets.toml`):

```toml
[database]
host = "your_database_host"
port = 3306
user = "your_db_user"
password = "your_db_password"
database = "harvard_artifacts"

[api]
harvard_api_key = "your_api_key_here"
```

## Database Schema

The project uses three main tables with proper relational structure:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    accession_number VARCHAR(100),
    credit_line TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    primary_image_url TEXT,
    image_count INT DEFAULT 0,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Integration Patterns

### Connecting to Harvard Art Museums API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Total pages: {data['info']['pages']}")
print(f"Records fetched: {len(data['records'])}")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data['records'])
            
            print(f"Fetched page {page}/{max_pages}")
            
            # Respect rate limits
            import time
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract Phase

```python
import pandas as pd

def extract_artifact_data(artifact):
    """Extract relevant fields from artifact JSON"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title'),
        'culture': artifact.get('culture'),
        'century': artifact.get('century'),
        'classification': artifact.get('classification'),
        'department': artifact.get('department'),
        'division': artifact.get('division'),
        'dated': artifact.get('dated'),
        'technique': artifact.get('technique'),
        'medium': artifact.get('medium'),
        'dimensions': artifact.get('dimensions'),
        'accession_number': artifact.get('accessionyear'),
        'credit_line': artifact.get('creditline')
    }

def extract_media_data(artifact):
    """Extract media/image information"""
    return {
        'artifact_id': artifact.get('id'),
        'primary_image_url': artifact.get('primaryimageurl'),
        'image_count': artifact.get('totalpageviews', 0)
    }

def extract_color_data(artifact):
    """Extract color information"""
    colors = []
    if 'colors' in artifact and artifact['colors']:
        for color in artifact['colors']:
            colors.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    return colors
```

### Transform Phase

```python
def transform_artifacts_to_dataframes(artifacts):
    """Transform list of artifacts into structured DataFrames"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_list.append(extract_artifact_data(artifact))
        
        # Extract media
        media_list.append(extract_media_data(artifact))
        
        # Extract colors
        colors_list.extend(extract_color_data(artifact))
    
    # Create DataFrames
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    # Clean and validate
    df_metadata['title'] = df_metadata['title'].fillna('Unknown')
    df_metadata = df_metadata.drop_duplicates(subset=['id'])
    
    return df_metadata, df_media, df_colors
```

### Load Phase

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(config):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=config['host'],
            port=config['port'],
            user=config['user'],
            password=config['password'],
            database=config['database']
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_dataframe_to_sql(df, table_name, connection, if_exists='append'):
    """Load DataFrame into SQL table with batch inserts"""
    cursor = connection.cursor()
    
    # Prepare insert statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"""
        INSERT INTO {table_name} ({columns})
        VALUES ({placeholders})
        ON DUPLICATE KEY UPDATE id=id
    """
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Loaded {len(data_tuples)} rows into {table_name}")
    except Error as e:
        connection.rollback()
        print(f"Error loading data: {e}")
    finally:
        cursor.close()

# Complete ETL pipeline
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """Execute full ETL pipeline"""
    
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts_to_dataframes(artifacts)
    
    # Load
    print("Loading data to database...")
    connection = create_database_connection(db_config)
    
    if connection:
        load_dataframe_to_sql(df_metadata, 'artifactmetadata', connection)
        load_dataframe_to_sql(df_media, 'artifactmedia', connection)
        load_dataframe_to_sql(df_colors, 'artifactcolors', connection)
        
        connection.close()
        print("ETL pipeline completed successfully!")
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN image_count > 0 THEN 'With Images' ELSE 'Without Images' END as status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY status
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "classification_breakdown": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_analytical_query(connection, query_name):
    """Execute a predefined analytical query"""
    cursor = connection.cursor(dictionary=True)
    
    try:
        cursor.execute(ANALYTICAL_QUERIES[query_name])
        results = cursor.fetchall()
        return pd.DataFrame(results)
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        cursor.close()
```

## Streamlit Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        
        # API Key input
        api_key = st.text_input("Harvard API Key", type="password",
                               value=os.getenv('HARVARD_API_KEY', ''))
        
        # Database config
        db_config = {
            'host': st.text_input("DB Host", value=os.getenv('DB_HOST', 'localhost')),
            'port': st.number_input("DB Port", value=3306),
            'user': st.text_input("DB User", value=os.getenv('DB_USER', '')),
            'password': st.text_input("DB Password", type="password",
                                     value=os.getenv('DB_PASSWORD', '')),
            'database': st.text_input("DB Name", value=os.getenv('DB_NAME', 'harvard_artifacts'))
        }
        
        # ETL Controls
        st.header("ETL Pipeline")
        num_pages = st.slider("Pages to fetch", 1, 20, 5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                run_etl_pipeline(api_key, db_config, num_pages)
                st.success("ETL completed!")
    
    # Main content tabs
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔍 Query Explorer", "📈 Visualizations"])
    
    with tab1:
        display_analytics_dashboard(db_config)
    
    with tab2:
        display_query_explorer(db_config)
    
    with tab3:
        display_visualizations(db_config)

def display_analytics_dashboard(db_config):
    """Display predefined analytics"""
    st.header("Quick Analytics")
    
    connection = create_database_connection(db_config)
    if not connection:
        st.error("Database connection failed")
        return
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Artifacts by Culture")
        df = execute_analytical_query(connection, "artifacts_by_culture")
        if df is not None:
            fig = px.bar(df, x='culture', y='count', title="Top Cultures")
            st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        st.subheader("Artifacts by Century")
        df = execute_analytical_query(connection, "artifacts_by_century")
        if df is not None:
            fig = px.bar(df, x='century', y='count', title="Distribution by Century")
            st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

def display_query_explorer(db_config):
    """Custom SQL query interface"""
    st.header("Custom Query Explorer")
    
    query = st.text_area("Enter SQL Query", height=150,
                         value="SELECT * FROM artifactmetadata LIMIT 10")
    
    if st.button("Execute Query"):
        connection = create_database_connection(db_config)
        if connection:
            try:
                df = pd.read_sql(query, connection)
                st.dataframe(df, use_container_width=True)
                st.download_button("Download CSV", 
                                  df.to_csv(index=False),
                                  "query_results.csv")
            except Exception as e:
                st.error(f"Query error: {e}")
            finally:
                connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="your_host"
export DB_USER="your_user"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_rate_limited(api_key, page):
    return fetch_artifacts(api_key, page)
```

### Error Handling and Retry Logic

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=10))
def fetch_with_retry(api_key, page):
    """Fetch artifacts with automatic retry on failure"""
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params={'apikey': api_key, 'page': page, 'size': 100}
    )
    response.raise_for_status()
    return response.json()
```

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl(api_key, db_config):
    """Load only new artifacts since last ETL run"""
    connection = create_database_connection(db_config)
    last_id = get_last_artifact_id(connection)
    
    # Fetch only artifacts with id > last_id
    new_artifacts = fetch_artifacts_after_id(api_key, last_id)
    
    # Transform and load
    df_metadata, df_media, df_colors = transform_artifacts_to_dataframes(new_artifacts)
    load_dataframe_to_sql(df_metadata, 'artifactmetadata', connection)
    load_dataframe_to_sql(df_media, 'artifactmedia', connection)
    load_dataframe_to_sql(df_colors, 'artifactcolors', connection)
    
    connection.close()
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is working
def test_api_connection(api_key):
    """Test Harvard API connection"""
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1}
        )
        response.raise_for_status()
        print("✓ API connection successful")
        return True
    except requests.exceptions.HTTPError as e:
        if e.response.status_code == 401:
            print("✗ Invalid API key")
        else:
            print(f"✗ API error: {e}")
        return False
```

### Database Connection Issues

```python
def diagnose_database_connection(config):
    """Diagnose database connection problems"""
    try:
        connection = mysql.connector.connect(
            host=config['host'],
            port=config['port'],
            user=config['user'],
            password=config['password'],
            database=config['database'],
            connect_timeout=10
        )
        print("✓ Database connection successful")
        connection.close()
        return True
    except Error as e:
        print(f"✗ Database error: {e}")
        if "Access denied" in str(e):
            print("  Check username and password")
        elif "Unknown database" in str(e):
            print("  Database does not exist - create it first")
        elif "Can't connect" in str(e):
            print("  Check host and port")
        return False
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(api_key, db_config, batch_size=100):
    """Process artifacts in batches to manage memory"""
    connection = create_database_connection(db_config)
    page = 1
    
    while True:
        # Fetch batch
        data = fetch_artifacts(api_key, page=page, size=batch_size)
        
        if not data['records']:
            break
        
        # Process and load immediately
        df_metadata, df_media, df_colors = transform_artifacts_to_dataframes(data['records'])
        load_dataframe_to_sql(df_metadata, 'artifactmetadata', connection)
        load_dataframe_to_sql(df_media, 'artifactmedia', connection)
        load_dataframe_to_sql(df_colors, 'artifactcolors', connection)
        
        print(f"Processed batch {page}")
        page += 1
    
    connection.close()
```

This skill provides comprehensive guidance for building and working with the Harvard Artifacts data engineering pipeline, from API integration through ETL to analytics and visualization.
