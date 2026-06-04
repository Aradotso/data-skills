---
name: harvard-artifacts-collection-data-engineering-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with museum artifacts
  - set up analytics dashboard for Harvard API
  - implement artifact collection data pipeline
  - build Streamlit app for museum data visualization
  - create SQL analytics for art collection data
  - extract and transform Harvard museum API data
  - develop data warehouse for artifacts collection
---

# Harvard Artifacts Collection Data Engineering Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering and analytics solution for the Harvard Art Museums API. It demonstrates production-grade ETL pipelines that extract artifact metadata, transform nested JSON into relational schemas, load into SQL databases (MySQL/TiDB Cloud), and visualize insights through an interactive Streamlit dashboard.

The application handles:
- API pagination and rate limiting
- Nested JSON transformation into normalized tables
- Batch SQL operations for performance
- 20+ analytical queries for artifact insights
- Real-time interactive visualizations with Plotly

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
```

## Configuration

### API Key Setup

Obtain a Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Configure the API key in your environment:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or store in Streamlit secrets (`.streamlit/secrets.toml`):

```toml
HARVARD_API_KEY = "your_api_key_here"
```

### Database Setup

Configure MySQL/TiDB Cloud connection:

```python
import os

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Project Structure

```
├── app.py                 # Main Streamlit application
├── etl_pipeline.py        # ETL logic for data extraction/transformation
├── db_utils.py            # Database connection and operations
├── sql_queries.py         # Analytical SQL queries
├── requirements.txt       # Python dependencies
└── .streamlit/
    └── secrets.toml       # API keys and DB credentials
```

## Database Schema

### Table: artifactmetadata

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    provenance TEXT,
    copyright TEXT,
    url VARCHAR(500)
);
```

### Table: artifactmedia

```sql
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    thumbnail_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### Table: artifactcolors

```sql
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_records.extend(records)
            
            if not records or len(all_records) >= num_records:
                break
                
            page += 1
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_records]
```

### Transform: Normalize JSON to Tables

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Metadata table
        metadata_list.append({
            'id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'provenance': artifact.get('provenance'),
            'copyright': artifact.get('copyright'),
            'url': artifact.get('url')
        })
        
        # Media table
        images = artifact.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'thumbnail_url': img.get('thumbnailurl'),
                'media_type': 'image'
            })
        
        # Colors table
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert to SQL

```python
import mysql.connector
from mysql.connector import Error

def create_connection(config):
    """Create database connection"""
    try:
        connection = mysql.connector.connect(**config)
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def batch_insert_metadata(connection, df):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, dated, classification, 
     department, division, technique, medium, dimensions, 
     creditline, provenance, copyright, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    values = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()

def batch_insert_media(connection, df):
    """Batch insert artifact media"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, image_url, thumbnail_url, media_type)
    VALUES (%s, %s, %s, %s)
    """
    
    values = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()

def batch_insert_colors(connection, df):
    """Batch insert artifact colors"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    values = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()
```

## Analytical SQL Queries

### Top Cultures by Artifact Count

```python
QUERY_TOP_CULTURES = """
SELECT 
    culture, 
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""
```

### Artifacts by Century

```python
QUERY_BY_CENTURY = """
SELECT 
    century, 
    COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""
```

### Media Availability Analysis

```python
QUERY_MEDIA_STATS = """
SELECT 
    COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
    COUNT(*) as total_media_items,
    AVG(media_per_artifact) as avg_media_per_artifact
FROM artifactmedia m
JOIN (
    SELECT artifact_id, COUNT(*) as media_per_artifact
    FROM artifactmedia
    GROUP BY artifact_id
) sub ON m.artifact_id = sub.artifact_id;
"""
```

### Color Distribution

```python
QUERY_COLOR_DISTRIBUTION = """
SELECT 
    color,
    COUNT(*) as usage_count,
    AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15;
"""
```

### Department-wise Classification

```python
QUERY_DEPT_CLASSIFICATION = """
SELECT 
    department,
    classification,
    COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY count DESC
LIMIT 20;
"""
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
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_viz_page()

def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    api_key = st.text_input("Enter Harvard API Key", type="password")
    num_records = st.number_input("Number of records", 10, 1000, 100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            raw_data = fetch_artifacts(api_key, num_records)
            st.success(f"Fetched {len(raw_data)} records")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed")
            
            col1, col2, col3 = st.columns(3)
            col1.metric("Metadata Records", len(metadata_df))
            col2.metric("Media Records", len(media_df))
            col3.metric("Color Records", len(colors_df))
        
        with st.spinner("Loading to database..."):
            connection = create_connection(DB_CONFIG)
            batch_insert_metadata(connection, metadata_df)
            batch_insert_media(connection, media_df)
            batch_insert_colors(connection, colors_df)
            connection.close()
            st.success("Data loaded successfully!")

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures": QUERY_TOP_CULTURES,
        "Artifacts by Century": QUERY_BY_CENTURY,
        "Color Distribution": QUERY_COLOR_DISTRIBUTION
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = create_connection(DB_CONFIG)
        cursor = connection.cursor()
        cursor.execute(queries[selected_query])
        
        results = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        
        df = pd.DataFrame(results, columns=columns)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig, use_container_width=True)
        
        cursor.close()
        connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Executing Custom Analytics

```python
def run_custom_query(connection, query):
    """Execute custom SQL query and return DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)

# Usage
connection = create_connection(DB_CONFIG)
query = """
    SELECT culture, COUNT(*) as count 
    FROM artifactmetadata 
    GROUP BY culture
"""
df = run_custom_query(connection, query)
connection.close()
```

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    return result or 0

def fetch_new_artifacts(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    url = f"https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'after': last_id,
        'size': 100
    }
    response = requests.get(url, params=params)
    return response.json().get('records', [])
```

## Troubleshooting

### API Rate Limiting

If you encounter 429 errors:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session

# Use in API calls
session = create_session_with_retries()
response = session.get(url, params=params)
```

### Database Connection Issues

```python
def test_db_connection(config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**config)
        if connection.is_connected():
            print("✓ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE()")
            db_name = cursor.fetchone()[0]
            print(f"✓ Connected to database: {db_name}")
            cursor.close()
            connection.close()
            return True
    except Error as e:
        print(f"✗ Connection failed: {e}")
        return False
```

### Missing Data Handling

```python
def safe_transform(artifact):
    """Handle missing fields gracefully"""
    return {
        'id': artifact.get('id', 0),
        'title': artifact.get('title', 'Unknown'),
        'culture': artifact.get('culture') or 'Not Specified',
        'century': artifact.get('century') or 'Unknown Period',
        # Use .get() with defaults for all fields
    }
```

### Streamlit Performance Optimization

```python
@st.cache_data(ttl=3600)
def fetch_cached_artifacts(api_key, num_records):
    """Cache API results for 1 hour"""
    return fetch_artifacts(api_key, num_records)

@st.cache_resource
def get_db_connection():
    """Reuse database connection"""
    return create_connection(DB_CONFIG)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Execute specific SQL query
python -c "from sql_queries import *; print(execute_query(QUERY_TOP_CULTURES))"
```

## Environment Variables Reference

```bash
# Required
HARVARD_API_KEY=your_api_key

# Database
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306

# Optional
ETL_BATCH_SIZE=100
API_RATE_LIMIT_DELAY=0.5
```
