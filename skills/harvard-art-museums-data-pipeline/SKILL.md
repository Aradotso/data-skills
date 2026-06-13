---
name: harvard-art-museums-data-pipeline
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create an ETL pipeline with Harvard Art Museums API
  - set up analytics dashboard for art collection data
  - implement SQL-based artifact analytics
  - build Streamlit app for museum data visualization
  - work with Harvard Art Museums API data engineering
  - create end-to-end data pipeline with API and SQL
  - visualize museum collection data with Plotly
---

# Harvard Art Museums Data Pipeline & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums collection. It demonstrates:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational SQL tables
- **SQL Analytics**: Stores data in MySQL/TiDB Cloud with normalized schema
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for data exploration

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Key Dependencies**:
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

Create a `.env` file or configure in your app:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
import os
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_art_db'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

### 3. Database Schema Setup

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY,
    artifactid INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    description TEXT,
    technique VARCHAR(200),
    FOREIGN KEY (artifactid) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    artifactid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifactid) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit app will open at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_pages=5):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,  # Max per page
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### 2. ETL Transform Logic

```python
def transform_artifacts(raw_data):
    """Transform nested JSON into normalized dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Metadata extraction
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Media extraction
        for media in artifact.get('images', []):
            media_list.append({
                'mediaid': media.get('imageid'),
                'artifactid': artifact.get('id'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format'),
                'description': media.get('description'),
                'technique': media.get('technique')
            })
        
        # Color extraction
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifactid': artifact.get('id'),
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

### 3. SQL Load Operations

```python
def batch_insert_data(conn, df_metadata, df_media, df_colors):
    """Batch insert transformed data into SQL database"""
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         dated, url, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE 
        title=VALUES(title), totalpageviews=VALUES(totalpageviews)
    """
    cursor.executemany(metadata_query, df_metadata.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (mediaid, artifactid, baseimageurl, format, description, technique)
        VALUES (%s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE baseimageurl=VALUES(baseimageurl)
    """
    cursor.executemany(media_query, df_media.values.tolist())
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors 
        (artifactid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    cursor.executemany(colors_query, df_colors.values.tolist())
    
    conn.commit()
    cursor.close()
```

### 4. Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT m.id, m.title, COUNT(media.mediaid) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifactid
        GROUP BY m.id, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Most Viewed Artifacts": """
        SELECT title, culture, century, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 10
    """
}

def run_analytics_query(conn, query_name):
    """Execute analytics query and return results as DataFrame"""
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        return None
    
    return pd.read_sql(query, conn)
```

### 5. Streamlit Visualization

```python
import streamlit as st
import plotly.express as px

def create_dashboard(conn):
    """Create interactive analytics dashboard"""
    st.title("🎨 Harvard Art Museums Analytics")
    
    # Query selector
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = run_analytics_query(conn, query_name)
            
            if df is not None and not df.empty:
                st.subheader("Results")
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(
                        df, 
                        x=df.columns[0], 
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Full ETL Pipeline Execution

```python
import os
from dotenv import load_dotenv
import mysql.connector

def run_etl_pipeline():
    """Execute complete ETL pipeline"""
    load_dotenv()
    
    # 1. Extract
    api_key = os.getenv('HARVARD_API_KEY')
    raw_artifacts = fetch_artifacts(api_key, num_pages=10)
    
    # 2. Transform
    df_meta, df_media, df_colors = transform_artifacts(raw_artifacts)
    
    # 3. Load
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    batch_insert_data(conn, df_meta, df_media, df_colors)
    conn.close()
    
    print(f"ETL Complete: {len(df_meta)} artifacts loaded")

if __name__ == "__main__":
    run_etl_pipeline()
```

### Incremental Data Updates

```python
def get_latest_artifact_id(conn):
    """Get the most recent artifact ID in database"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, conn):
    """Load only new artifacts since last update"""
    latest_id = get_latest_artifact_id(conn)
    
    # Fetch artifacts with ID greater than latest
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'desc',
        'hasimage': 1
    }
    
    response = requests.get(url, params=params)
    new_artifacts = [a for a in response.json()['records'] if a['id'] > latest_id]
    
    if new_artifacts:
        df_meta, df_media, df_colors = transform_artifacts(new_artifacts)
        batch_insert_data(conn, df_meta, df_media, df_colors)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting

The Harvard API has rate limits. Add delays between requests:

```python
import time

def fetch_artifacts_with_rate_limit(api_key, num_pages=5, delay=1):
    """Fetch with rate limiting"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        response = requests.get(base_url, params=params)
        if response.status_code == 200:
            all_artifacts.extend(response.json()['records'])
            time.sleep(delay)  # Delay between requests
        elif response.status_code == 429:
            print("Rate limited. Waiting 60 seconds...")
            time.sleep(60)
    
    return all_artifacts
```

### Database Connection Issues

```python
def get_db_connection_with_retry(max_retries=3):
    """Create database connection with retry logic"""
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(**db_config)
            return conn
        except mysql.connector.Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(5)
            else:
                raise
```

### Handling Missing Data

```python
def safe_extract(artifact, key, default=None):
    """Safely extract nested values"""
    return artifact.get(key, default)

def transform_with_validation(raw_data):
    """Transform with null handling"""
    metadata = {
        'id': safe_extract(artifact, 'id', 0),
        'title': safe_extract(artifact, 'title', 'Unknown')[:500],  # Truncate
        'culture': safe_extract(artifact, 'culture'),
        # ... handle all fields safely
    }
```

### Memory Optimization for Large Datasets

```python
def process_in_chunks(api_key, total_pages=100, chunk_size=10):
    """Process data in chunks to avoid memory issues"""
    conn = get_db_connection_with_retry()
    
    for start_page in range(1, total_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size, total_pages + 1)
        
        # Process chunk
        chunk_data = fetch_artifacts(api_key, start_page, end_page)
        df_meta, df_media, df_colors = transform_artifacts(chunk_data)
        batch_insert_data(conn, df_meta, df_media, df_colors)
        
        print(f"Processed pages {start_page}-{end_page}")
    
    conn.close()
```

## Advanced Features

### Custom Analytics Query Builder

```python
def build_custom_query(table, filters=None, group_by=None, order_by=None, limit=10):
    """Build custom SQL queries programmatically"""
    query = f"SELECT * FROM {table}"
    
    if filters:
        conditions = " AND ".join([f"{k} = '{v}'" for k, v in filters.items()])
        query += f" WHERE {conditions}"
    
    if group_by:
        query += f" GROUP BY {group_by}"
    
    if order_by:
        query += f" ORDER BY {order_by} DESC"
    
    query += f" LIMIT {limit}"
    
    return query
```

This skill provides everything needed to build, deploy, and extend the Harvard Art Museums data pipeline for ETL and analytics workflows.
