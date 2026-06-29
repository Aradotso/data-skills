---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - create a data pipeline for Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - set up analytics dashboard with Streamlit and SQL
  - extract and visualize Harvard museum collection data
  - implement artifact metadata ETL process
  - query and analyze museum artifact data with SQL
  - build interactive museum data visualization app
  - create Harvard Art Museums data engineering project
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational SQL tables, and provides interactive analytics through a Streamlit dashboard. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

Key capabilities:
- Paginated API data collection with rate limiting
- ETL pipeline transforming nested JSON to relational tables
- SQL database design with proper foreign key relationships
- 20+ predefined analytical queries
- Interactive Plotly visualizations

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
export DB_USER="your_db_username"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

## Configuration

### Database Setup

The project uses MySQL or TiDB Cloud with three main tables:

```python
# Database connection configuration
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error: {e}")
        return None
```

### Table Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    period VARCHAR(200),
    dated VARCHAR(200),
    url TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline

### Extract: API Data Collection

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
        page_size: Items per page (max 100)
    
    Returns:
        List of artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{num_pages}")
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### Transform: Data Processing

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured DataFrames
    
    Args:
        raw_data: List of artifact dictionaries from API
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'url': artifact.get('url', '')
        })
        
        # Extract media information
        if artifact.get('images'):
            for image in artifact['images']:
                media_records.append({
                    'artifact_id': artifact['id'],
                    'image_url': image.get('baseimageurl'),
                    'media_type': 'image'
                })
        
        # Extract color information
        if artifact.get('colors'):
            for color_data in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact['id'],
                    'color': color_data.get('color'),
                    'spectrum': color_data.get('spectrum'),
                    'percent': color_data.get('percent')
                })
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load: Database Insertion

```python
def load_to_database(connection, metadata_df, media_df, colors_df):
    """
    Load transformed data into SQL database
    
    Args:
        connection: MySQL connection object
        metadata_df: Artifact metadata DataFrame
        media_df: Media information DataFrame
        colors_df: Color information DataFrame
    """
    cursor = connection.cursor()
    
    # Insert metadata (batch insert for performance)
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, technique, period, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    metadata_values = metadata_df.values.tolist()
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
    """
    
    if not media_df.empty:
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
    """
    
    if not colors_df.empty:
        color_values = colors_df.values.tolist()
        cursor.executemany(colors_query, color_values)
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

## Analytics Queries

### Sample SQL Analytics

```python
# Analytics queries dictionary
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE 
                WHEN image_url IS NOT NULL THEN 'With Images'
                ELSE 'Without Images'
            END as image_status,
            COUNT(DISTINCT a.id) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY image_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    # Database connection
    conn = create_connection()
    
    if page == "Data Collection":
        show_data_collection_page(conn)
    elif page == "SQL Analytics":
        show_analytics_page(conn)
    elif page == "Visualizations":
        show_visualization_page(conn)

def show_data_collection_page(conn):
    """Data collection interface"""
    st.header("📥 Collect Artifact Data")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                             value=os.getenv('HARVARD_API_KEY', ''))
    num_pages = st.slider("Number of pages to fetch", 1, 50, 10)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(api_key, num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(conn, metadata_df, media_df, colors_df)
            st.success("Data loaded successfully!")

def show_analytics_page(conn):
    """SQL analytics interface"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.expander("View SQL Query"):
            st.code(query, language='sql')
        
        results_df = execute_query(conn, query)
        st.dataframe(results_df, use_container_width=True)
        
        # Auto-generate visualization
        if len(results_df.columns) >= 2:
            fig = px.bar(results_df, 
                         x=results_df.columns[0], 
                         y=results_df.columns[1],
                         title=query_name)
            st.plotly_chart(fig, use_container_width=True)

def show_visualization_page(conn):
    """Interactive visualizations"""
    st.header("📈 Data Visualizations")
    
    # Example: Color distribution
    color_df = execute_query(conn, ANALYTICS_QUERIES["Top Colors Used"])
    
    fig = px.pie(color_df, values='usage_count', names='color',
                 title='Color Distribution in Artifacts')
    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_pages):
    """ETL pipeline with error handling"""
    try:
        # Extract
        artifacts = fetch_artifacts(api_key, num_pages)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate data
        assert not metadata_df.empty, "Metadata is empty"
        
        # Load
        conn = create_connection()
        load_to_database(conn, metadata_df, media_df, colors_df)
        conn.close()
        
        return True, f"Successfully processed {len(artifacts)} artifacts"
    
    except Exception as e:
        return False, f"ETL failed: {str(e)}"
```

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the most recent artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, connection):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(connection)
    
    # Fetch artifacts with ID filter
    params = {
        'apikey': api_key,
        'q': f'id:>{last_id}',
        'size': 100
    }
    # Continue with ETL...
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session():
    session = requests.Session()
    retry = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues
```python
# Test connection before ETL
def test_db_connection():
    try:
        conn = create_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        conn.close()
        return True
    except:
        return False
```

### Memory Optimization for Large Datasets
```python
# Process in chunks
def chunked_load(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_to_database(conn, metadata_df, media_df, colors_df)
```
