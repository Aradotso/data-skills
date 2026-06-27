---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I fetch data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create a Streamlit analytics dashboard with SQL queries
  - process and store Harvard museum collection data
  - visualize artifact metadata with plotly charts
  - set up a data engineering project with museum API
  - paginate through Harvard Art Museums API results
  - transform nested JSON artifact data to relational tables
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection application:
- Fetches artifact data from the Harvard Art Museums API with pagination
- Performs ETL operations to transform nested JSON into relational tables
- Stores data in MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards
- Handles rate limiting and batch operations for performance

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Key Setup

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os

# Set API key as environment variable
os.environ['HARVARD_API_KEY'] = 'your_api_key_here'

# Or load from .env file
from dotenv import load_dotenv
load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

### Database Configuration

```python
import mysql.connector

# Database connection configuration
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Create connection
connection = mysql.connector.connect(**db_config)
cursor = connection.cursor()
```

## Core API Usage

### Fetching Artifacts with Pagination

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key
        page: Page number (default: 1)
        size: Results per page (max: 100)
    
    Returns:
        dict: API response with records and info
    """
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
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages with rate limiting
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts."""
    all_records = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_records.extend(data['records'])
        
        # Rate limiting - respect API limits
        time.sleep(0.5)
        
        # Check if more pages available
        if page >= data['info']['pages']:
            break
    
    return all_records
```

### Extracting Specific Fields

```python
def extract_artifact_fields(record):
    """Extract relevant fields from artifact record."""
    return {
        'objectid': record.get('objectid'),
        'title': record.get('title'),
        'culture': record.get('culture'),
        'period': record.get('period'),
        'century': record.get('century'),
        'classification': record.get('classification'),
        'medium': record.get('medium'),
        'department': record.get('department'),
        'division': record.get('division'),
        'dated': record.get('dated'),
        'url': record.get('url'),
        'creditline': record.get('creditline')
    }
```

## ETL Pipeline Implementation

### Transform: Nested JSON to Relational Tables

```python
import pandas as pd

def transform_artifacts_to_metadata(records):
    """Transform API records to metadata table format."""
    metadata = []
    
    for record in records:
        metadata.append({
            'objectid': record.get('objectid'),
            'title': record.get('title', 'Unknown'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'medium': record.get('medium'),
            'department': record.get('department'),
            'division': record.get('division'),
            'dated': record.get('dated'),
            'url': record.get('url'),
            'creditline': record.get('creditline')
        })
    
    return pd.DataFrame(metadata)

def transform_artifacts_to_media(records):
    """Extract media/image data from artifacts."""
    media_data = []
    
    for record in records:
        objectid = record.get('objectid')
        images = record.get('images', [])
        
        for img in images:
            media_data.append({
                'objectid': objectid,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            })
    
    return pd.DataFrame(media_data)

def transform_artifacts_to_colors(records):
    """Extract color data from artifacts."""
    color_data = []
    
    for record in records:
        objectid = record.get('objectid')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent'),
                'hue': color.get('hue')
            })
    
    return pd.DataFrame(color_data)
```

### Load: Batch Insert to SQL

```python
def create_tables(cursor):
    """Create database schema."""
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            medium TEXT,
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            url VARCHAR(500),
            creditline TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl VARCHAR(500),
            format VARCHAR(50),
            height INT,
            width INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            percent FLOAT,
            hue VARCHAR(100),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)

def batch_insert_metadata(cursor, df):
    """Batch insert artifact metadata."""
    query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, 
         medium, department, division, dated, url, creditline)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(query, data)

def batch_insert_media(cursor, df):
    """Batch insert media data."""
    query = """
        INSERT INTO artifactmedia 
        (objectid, imageid, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(query, data)

def batch_insert_colors(cursor, df):
    """Batch insert color data."""
    query = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, percent, hue)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(query, data)
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, db_config, max_pages=5):
    """
    Complete ETL pipeline execution.
    
    Args:
        api_key: Harvard API key
        db_config: Database configuration dict
        max_pages: Number of API pages to fetch
    """
    # Extract
    print("Extracting data from API...")
    records = fetch_all_artifacts(api_key, max_pages=max_pages)
    print(f"Extracted {len(records)} artifacts")
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifacts_to_metadata(records)
    media_df = transform_artifacts_to_media(records)
    colors_df = transform_artifacts_to_colors(records)
    
    # Load
    print("Loading data to database...")
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    create_tables(cursor)
    
    batch_insert_metadata(cursor, metadata_df)
    print(f"Loaded {len(metadata_df)} metadata records")
    
    batch_insert_media(cursor, media_df)
    print(f"Loaded {len(media_df)} media records")
    
    batch_insert_colors(cursor, colors_df)
    print(f"Loaded {len(colors_df)} color records")
    
    connection.commit()
    cursor.close()
    connection.close()
    
    print("ETL pipeline completed successfully!")
```

## Analytical SQL Queries

```python
ANALYTICAL_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Artifacts with Multiple Images": """
        SELECT a.objectid, a.title, COUNT(m.imageid) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY a.objectid, a.title
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Average Color Percentage by Spectrum": """
        SELECT spectrum, AVG(percent) as avg_percent, COUNT(*) as count
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY avg_percent DESC
    """
}

def execute_query(cursor, query):
    """Execute SQL query and return results as DataFrame."""
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    db_config = {
        'host': st.sidebar.text_input("DB Host", value="localhost"),
        'user': st.sidebar.text_input("DB User", value="root"),
        'password': st.sidebar.text_input("DB Password", type="password"),
        'database': st.sidebar.text_input("Database", value="harvard_artifacts")
    }
    
    # ETL Section
    st.header("📥 ETL Pipeline")
    max_pages = st.number_input("Pages to Fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline(api_key, db_config, max_pages=max_pages)
                st.success("ETL pipeline completed successfully!")
            except Exception as e:
                st.error(f"Error: {str(e)}")
    
    # Analytics Section
    st.header("📊 Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        try:
            connection = mysql.connector.connect(**db_config)
            cursor = connection.cursor()
            
            df = execute_query(cursor, ANALYTICAL_QUERIES[query_name])
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)
            
            cursor.close()
            connection.close()
            
        except Exception as e:
            st.error(f"Query Error: {str(e)}")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Error Handling for API Requests

```python
import requests
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    """Create requests session with retry logic."""
    session = requests.Session()
    
    retries = Retry(
        total=3,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    
    adapter = HTTPAdapter(max_retries=retries)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    
    return session

# Usage
session = create_session_with_retries()
response = session.get(url, params=params, timeout=10)
```

### Incremental ETL Updates

```python
def get_last_updated_timestamp(cursor):
    """Get timestamp of last ETL run."""
    cursor.execute("""
        SELECT MAX(created_at) FROM etl_metadata
    """)
    result = cursor.fetchone()
    return result[0] if result[0] else None

def incremental_etl(api_key, db_config):
    """Only fetch new/updated artifacts."""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    last_update = get_last_updated_timestamp(cursor)
    
    params = {'apikey': api_key}
    if last_update:
        params['lastupdate'] = last_update.strftime('%Y-%m-%d')
    
    # Fetch and process only new data
    # ... ETL logic
```

## Troubleshooting

**API Rate Limiting:**
```python
# Add exponential backoff
import time

def fetch_with_backoff(url, params, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 429:
            wait_time = 2 ** attempt
            time.sleep(wait_time)
            continue
        return response
    raise Exception("Max retries exceeded")
```

**Database Connection Issues:**
```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

connection = db_pool.get_connection()
```

**Memory Issues with Large Datasets:**
```python
# Process in chunks
def process_in_chunks(records, chunk_size=1000):
    for i in range(0, len(records), chunk_size):
        chunk = records[i:i + chunk_size]
        # Process chunk
        yield chunk
```
