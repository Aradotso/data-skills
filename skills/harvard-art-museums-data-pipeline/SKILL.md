---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL for Harvard artifacts collection
  - create analytics dashboard for museum data
  - query Harvard Art Museums API and store in database
  - build Streamlit app for artifact data visualization
  - implement SQL analytics for museum collections
  - extract and transform Harvard museum artifact data
  - create end-to-end data engineering pipeline for art collections
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics application that:
- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Implements ETL pipelines to transform nested JSON into relational data
- Stores data in SQL databases (MySQL/TiDB Cloud) with proper schema design
- Executes analytical SQL queries on artifact metadata, media, and color data
- Visualizes insights through interactive Streamlit dashboards with Plotly charts

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
# Store in .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=your_database_name
```

### Database Connection

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

def get_db_connection():
    """Create MySQL database connection"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return connection
```

## Database Schema

### Create Tables

```python
def create_tables(connection):
    """Create database schema for artifacts"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            division VARCHAR(200),
            url VARCHAR(500),
            lastupdate DATETIME
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            format VARCHAR(50),
            caption TEXT,
            technique VARCHAR(200),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline

### Extract: Fetch from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
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
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
            # Rate limiting: 2500 requests/day, ~100/hour safe
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Parse Nested JSON

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform raw API data into structured dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        
        # Extract metadata
        metadata = {
            'objectid': object_id,
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'division': artifact.get('division', '')[:200],
            'url': artifact.get('url', '')[:500],
            'lastupdate': artifact.get('lastupdate')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media = {
                'objectid': object_id,
                'imageurl': img.get('baseimageurl', '')[:500],
                'iiifbaseuri': img.get('iiifbaseuri', '')[:500],
                'format': img.get('format', '')[:50],
                'caption': img.get('caption', ''),
                'technique': img.get('technique', '')[:200]
            }
            media_records.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'objectid': object_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_record)
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load: Batch Insert to SQL

```python
def load_to_database(metadata_df, media_df, colors_df, connection):
    """Load transformed data into database with batch inserts"""
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, 
         department, dated, medium, dimensions, creditline, division, url, lastupdate)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (objectid, imageurl, iiifbaseuri, format, caption, technique)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(metadata_df)} artifacts, {len(media_df)} media, {len(colors_df)} colors")
```

## Analytics Queries

### Example SQL Queries

```python
# Query 1: Artifacts by culture
query_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 15
"""

# Query 2: Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Top colors across artifacts
query_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query 4: Artifacts with most images
query_most_images = """
    SELECT m.objectid, m.title, COUNT(a.id) as image_count
    FROM artifactmetadata m
    JOIN artifactmedia a ON m.objectid = a.objectid
    GROUP BY m.objectid, m.title
    ORDER BY image_count DESC
    LIMIT 10
"""

# Query 5: Department distribution
query_departments = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    df = pd.read_sql(query, connection)
    return df
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for API key input
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    try:
        conn = get_db_connection()
        st.sidebar.success("✅ Database Connected")
    except Exception as e:
        st.sidebar.error(f"❌ Database Error: {e}")
        return
    
    # ETL Section
    st.header("📥 Data Collection")
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Pages to fetch", min_value=1, max_value=50, value=5)
    with col2:
        page_size = st.number_input("Items per page", min_value=10, max_value=100, value=100)
    
    if st.button("🚀 Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(api_key, num_pages, page_size)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Transformation complete")
        
        with st.spinner("Loading to database..."):
            create_tables(conn)
            load_to_database(metadata_df, media_df, colors_df, conn)
            st.success("✅ ETL Pipeline Complete!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_options = {
        "Artifacts by Culture": query_culture,
        "Artifacts by Century": query_century,
        "Top Colors": query_colors,
        "Most Imaged Artifacts": query_most_images,
        "Department Distribution": query_departments
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("🔍 Run Query"):
        query = query_options[selected_query]
        df = execute_query(conn, query)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

### Run the Dashboard

```bash
streamlit run app.py
```

## Common Patterns

### Rate Limiting Strategy

```python
from functools import wraps
import time

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    
    def decorator(func):
        last_called = [0.0]
        
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
def fetch_single_artifact(api_key, object_id):
    """Fetch single artifact with rate limiting"""
    url = f"https://api.harvardartmuseums.org/object/{object_id}"
    response = requests.get(url, params={'apikey': api_key})
    return response.json()
```

### Error Handling and Retry Logic

```python
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retries():
    """Create requests session with retry logic"""
    session = requests.Session()
    retry = Retry(
        total=3,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session

def safe_api_call(url, params):
    """API call with error handling"""
    session = get_session_with_retries()
    try:
        response = session.get(url, params=params, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API call failed: {e}")
        return None
```

## Troubleshooting

### API Key Issues

```python
# Test API key validity
def test_api_key(api_key):
    """Verify API key works"""
    url = "https://api.harvardartmuseums.org/object"
    response = requests.get(url, params={'apikey': api_key, 'size': 1})
    
    if response.status_code == 401:
        return False, "Invalid API key"
    elif response.status_code == 200:
        return True, "API key valid"
    else:
        return False, f"Error: {response.status_code}"
```

### Database Connection Issues

```python
def test_database_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        return True, "Database connection successful"
    except mysql.connector.Error as e:
        return False, f"Database error: {e}"
```

### Handle Missing Data

```python
def safe_get(data, key, default=''):
    """Safely extract nested data"""
    try:
        value = data.get(key, default)
        return value if value is not None else default
    except (AttributeError, TypeError):
        return default

# Use in transformation
metadata = {
    'title': safe_get(artifact, 'title', 'Untitled')[:500],
    'culture': safe_get(artifact, 'culture', 'Unknown')[:200]
}
```

### Memory Management for Large Datasets

```python
def fetch_artifacts_chunked(api_key, total_pages, chunk_size=5):
    """Fetch and process data in chunks to manage memory"""
    for chunk_start in range(1, total_pages + 1, chunk_size):
        chunk_end = min(chunk_start + chunk_size, total_pages + 1)
        
        artifacts = fetch_artifacts(api_key, chunk_end - chunk_start, 100)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        conn = get_db_connection()
        load_to_database(metadata_df, media_df, colors_df, conn)
        conn.close()
        
        print(f"Processed pages {chunk_start} to {chunk_end - 1}")
        
        # Clear memory
        del artifacts, metadata_df, media_df, colors_df
```
