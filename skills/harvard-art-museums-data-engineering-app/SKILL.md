---
name: harvard-art-museums-data-engineering-app
description: End-to-end data engineering pipeline with Harvard Art Museums API, ETL processing, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create ETL pipeline for museum artifact data
  - set up analytics dashboard with Streamlit and SQL
  - extract and analyze Harvard Art Museums collection data
  - implement data engineering workflow with API and database
  - visualize museum artifacts data with interactive dashboards
  - design relational database for art collection metadata
  - query and analyze museum artifact statistics
---

# Harvard Art Museums Data Engineering App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with end-to-end data engineering applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytics queries, and interactive Streamlit visualizations for museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection Data Engineering App is a complete data pipeline that:

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud databases
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

## Architecture

```
Harvard API → ETL Process → SQL Database → Analytics Engine → Streamlit Dashboard
```

**Data Flow:**
1. API calls with authentication and pagination
2. JSON parsing and data normalization
3. Batch inserts to relational tables
4. SQL query execution
5. Interactive visualization rendering

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

**Requirements.txt typically includes:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Database Schema

The application uses three main tables with relational integrity:

```sql
-- Artifact Metadata (main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    objectid INT UNIQUE,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(200),
    provenance TEXT,
    description TEXT,
    url VARCHAR(500)
);

-- Artifact Media (images, videos)
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

def collect_all_artifacts(max_pages=10):
    """
    Collect multiple pages of artifacts with rate limiting
    """
    import time
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        
        # Rate limiting: 1 request per second
        time.sleep(1)
    
    return all_artifacts
```

### Transform: Data Normalization

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata
    """
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'provenance': artifact.get('provenance'),
            'description': artifact.get('description'),
            'url': artifact.get('url')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """
    Extract media information (images, IIIF URIs)
    """
    media_records = []
    
    for artifact in artifacts:
        if 'images' in artifact and artifact['images']:
            record = {
                'objectid': artifact.get('objectid'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """
    Extract color palette data
    """
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load: Database Insertion

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata_to_db(df_metadata):
    """
    Batch insert artifact metadata
    """
    connection = get_db_connection()
    if not connection:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, objectid, title, culture, century, dated, department, 
         division, classification, medium, technique, period, 
         provenance, description, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    records = df_metadata.to_records(index=False)
    data_tuples = [tuple(record) for record in records]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()
        connection.close()

def load_media_to_db(df_media):
    """
    Batch insert artifact media
    """
    connection = get_db_connection()
    if not connection:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (objectid, baseimageurl, primaryimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
    """
    
    records = df_media.to_records(index=False)
    data_tuples = [tuple(record) for record in records]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()
        connection.close()
```

## Analytics Queries

### Sample SQL Queries

```python
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "media_availability": """
        SELECT 
            CASE 
                WHEN primaryimageurl IS NOT NULL THEN 'Has Image'
                ELSE 'No Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "top_classifications": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_coverage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "department_breakdown": """
        SELECT department, COUNT(*) as artifacts, 
               COUNT(DISTINCT classification) as unique_classifications
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifacts DESC
    """,
    
    "artifacts_with_provenance": """
        SELECT 
            CASE 
                WHEN provenance IS NOT NULL AND provenance != '' THEN 'Has Provenance'
                ELSE 'No Provenance'
            END as provenance_status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY provenance_status
    """
}

def execute_analytics_query(query_name):
    """
    Execute predefined analytics query and return DataFrame
    """
    connection = get_db_connection()
    if not connection:
        return None
    
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums - Data Engineering & Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics Dashboard", "Database Explorer"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics Dashboard":
        show_analytics_dashboard()
    elif page == "Database Explorer":
        show_database_explorer()

def show_data_collection():
    """
    ETL process interface
    """
    st.header("📥 Data Collection & ETL")
    
    col1, col2 = st.columns(2)
    
    with col1:
        max_pages = st.number_input("Number of Pages to Fetch", 1, 50, 5)
        
    with col2:
        items_per_page = st.number_input("Items Per Page", 10, 100, 100)
    
    if st.button("🚀 Start ETL Process"):
        with st.spinner("Fetching data from API..."):
            artifacts = collect_all_artifacts(max_pages=max_pages)
            st.success(f"✅ Collected {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            st.success("✅ Data transformation complete")
        
        with st.spinner("Loading to database..."):
            load_metadata_to_db(df_metadata)
            load_media_to_db(df_media)
            st.success("✅ Data loaded successfully")

def show_analytics_dashboard():
    """
    Interactive analytics dashboard
    """
    st.header("📊 Analytics Dashboard")
    
    query_options = list(ANALYTICS_QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        df_result = execute_analytics_query(selected_query)
        
        if df_result is not None and not df_result.empty:
            st.subheader("Query Results")
            st.dataframe(df_result, use_container_width=True)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                col_x = df_result.columns[0]
                col_y = df_result.columns[1]
                
                fig = px.bar(
                    df_result,
                    x=col_x,
                    y=col_y,
                    title=f"{selected_query.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No data returned from query")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Updates

```python
def incremental_etl(last_update_timestamp):
    """
    Fetch only new/updated artifacts since last ETL run
    """
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'lastupdate',
        'sortorder': 'desc',
        'after': last_update_timestamp
    }
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    return response.json()
```

### Pattern 2: Error Handling & Retry Logic

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    """
    Create requests session with automatic retry logic
    """
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session

def fetch_with_retry(url, params):
    session = create_session_with_retries()
    response = session.get(url, params=params, timeout=10)
    return response.json()
```

### Pattern 3: Data Quality Checks

```python
def validate_artifact_data(df):
    """
    Perform data quality checks before loading
    """
    issues = []
    
    # Check for required fields
    if df['objectid'].isnull().any():
        issues.append("Missing objectid values")
    
    # Check for duplicates
    if df['objectid'].duplicated().any():
        issues.append(f"Found {df['objectid'].duplicated().sum()} duplicate objectids")
    
    # Check data types
    if not pd.api.types.is_integer_dtype(df['objectid']):
        issues.append("objectid should be integer type")
    
    return issues
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard API
HARVARD_API_KEY=your_api_key_from_harvard

# Database Configuration
DB_HOST=your_mysql_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts

# Optional: TiDB Cloud
TIDB_CLOUD_HOST=gateway01.region.prod.aws.tidbcloud.com
TIDB_CLOUD_PORT=4000
TIDB_CLOUD_USER=your_tidb_user
TIDB_CLOUD_PASSWORD=your_tidb_password
```

### Streamlit Configuration

Create `.streamlit/config.toml`:

```toml
[theme]
primaryColor = "#A51C30"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F0F2F6"
textColor = "#262730"
font = "sans serif"

[server]
headless = true
port = 8501
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Run ETL pipeline only (if separated)
python etl_pipeline.py

# Run specific analytics query
python analytics.py --query artifacts_by_culture
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time

def fetch_with_backoff(url, params, max_retries=5):
    for i in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 429:
            wait_time = 2 ** i
            print(f"Rate limited. Waiting {wait_time}s...")
            time.sleep(wait_time)
        else:
            return response
```

### Database Connection Issues
```python
# Test connection
def test_db_connection():
    try:
        connection = get_db_connection()
        if connection.is_connected():
            print("✅ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE();")
            db_name = cursor.fetchone()
            print(f"Connected to database: {db_name[0]}")
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Memory Issues with Large Datasets
```python
# Process in chunks
def chunked_etl(total_pages, chunk_size=10):
    for chunk_start in range(1, total_pages, chunk_size):
        chunk_end = min(chunk_start + chunk_size, total_pages)
        artifacts = collect_all_artifacts_range(chunk_start, chunk_end)
        
        # Transform and load chunk
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata_to_db(df_metadata)
        
        # Clear memory
        del artifacts, df_metadata
```

This skill provides complete coverage of building data engineering pipelines with the Harvard Art Museums API, from ETL processes to interactive analytics dashboards.
