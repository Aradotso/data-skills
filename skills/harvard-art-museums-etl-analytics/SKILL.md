---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards for the Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifact data
  - connect to Harvard Art Museums API and store in SQL
  - build data engineering pipeline with Streamlit visualization
  - extract and transform Harvard museum artifacts data
  - set up SQL analytics for art collections API
  - visualize Harvard Art Museums data with Plotly
  - implement batch ETL for museum artifacts
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that demonstrates real-world ETL patterns for museum artifact data. It extracts data from the Harvard Art Museums API, transforms nested JSON into normalized SQL tables, loads data efficiently with batch inserts, and provides interactive analytics dashboards using Streamlit and Plotly.

**Architecture:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit app
streamlit run app.py
```

### Required Dependencies

```python
# requirements.txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

The Harvard Art Museums API requires an API key. Store it as an environment variable:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
MYSQL_HOST=your_database_host
MYSQL_USER=your_database_user
MYSQL_PASSWORD=your_database_password
MYSQL_DATABASE=harvard_artifacts
```

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
DB_CONFIG = {
    'host': os.getenv('MYSQL_HOST'),
    'user': os.getenv('MYSQL_USER'),
    'password': os.getenv('MYSQL_PASSWORD'),
    'database': os.getenv('MYSQL_DATABASE')
}
```

### Database Schema

The application uses three normalized tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    creditline TEXT,
    accessionyear INT
);

CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(100),
    height INT,
    width INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core ETL Components

### 1. Extract: API Data Collection

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Extract artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts.extend(data.get('records', []))
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return artifacts
```

### 2. Transform: Data Normalization

```python
import pandas as pd

def transform_artifacts(artifacts_json):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts_json:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        images = artifact.get('images', [])
        for image in images:
            media = {
                'mediaid': image.get('imageid'),
                'objectid': artifact.get('objectid'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            }
            media_records.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'objectid': artifact.get('objectid'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 3. Load: Batch SQL Inserts

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """
    Load dataframes into MySQL database with batch inserts
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (objectid, title, culture, period, century, classification, 
             medium, dimensions, department, division, dated, creditline, accessionyear)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = [tuple(row) for row in df_metadata.values]
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media (batch)
        media_query = """
            INSERT INTO artifactmedia 
            (mediaid, objectid, baseimageurl, format, height, width)
            VALUES (%s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE baseimageurl=VALUES(baseimageurl)
        """
        media_values = [tuple(row) for row in df_media.values]
        cursor.executemany(media_query, media_values)
        
        # Insert colors (batch)
        colors_query = """
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        colors_values = [tuple(row) for row in df_colors.values]
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_values)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Analytics Queries

### Sample Analytical Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
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
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.objectid IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(a.objectid) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def run_analytics_query(query, db_config):
    """
    Execute analytical query and return results as DataFrame
    """
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        if connection.is_connected():
            connection.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for API configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("ETL Pipeline")
        num_pages = st.slider("Pages to fetch", 1, 50, 10)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Extracting data..."):
                artifacts = fetch_artifacts(api_key, num_pages)
                st.success(f"Extracted {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                df_meta, df_media, df_colors = transform_artifacts(artifacts)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                load_to_database(df_meta, df_media, df_colors, DB_CONFIG)
                st.success("Data loaded to SQL")
    
    # Analytics section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICAL_QUERIES[query_name]
        
        with st.spinner("Running query..."):
            df_result = run_analytics_query(query, DB_CONFIG)
        
        if df_result is not None and not df_result.empty:
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(df_result, 
                            x=df_result.columns[0], 
                            y=df_result.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
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
def fetch_single_artifact(artifact_id, api_key):
    url = f"https://api.harvardartmuseums.org/object/{artifact_id}"
    response = requests.get(url, params={'apikey': api_key})
    return response.json()
```

### Error Handling for ETL

```python
def safe_etl_pipeline(api_key, num_pages, db_config):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        artifacts = fetch_artifacts(api_key, num_pages)
        if not artifacts:
            raise ValueError("No artifacts extracted")
        
        # Transform
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        
        # Validate
        if df_meta.empty:
            raise ValueError("Metadata transformation failed")
        
        # Load
        load_to_database(df_meta, df_media, df_colors, db_config)
        
        return {
            'status': 'success',
            'artifacts': len(artifacts),
            'metadata_rows': len(df_meta),
            'media_rows': len(df_media),
            'color_rows': len(df_colors)
        }
        
    except requests.exceptions.RequestException as e:
        return {'status': 'error', 'stage': 'extract', 'message': str(e)}
    except ValueError as e:
        return {'status': 'error', 'stage': 'transform', 'message': str(e)}
    except Error as e:
        return {'status': 'error', 'stage': 'load', 'message': str(e)}
```

## Troubleshooting

### API Key Issues

```python
def validate_api_key(api_key):
    """Test API key validity"""
    test_url = "https://api.harvardartmuseums.org/object"
    try:
        response = requests.get(test_url, params={'apikey': api_key, 'size': 1})
        if response.status_code == 401:
            return False, "Invalid API key"
        elif response.status_code == 200:
            return True, "API key valid"
        else:
            return False, f"Unexpected status: {response.status_code}"
    except Exception as e:
        return False, str(e)
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            cursor = connection.cursor()
            cursor.execute("SELECT 1")
            result = cursor.fetchone()
            cursor.close()
            connection.close()
            return True, "Database connection successful"
        return False, "Could not establish connection"
    except Error as e:
        return False, f"Database error: {e}"
```

### Memory Management for Large Datasets

```python
def chunked_load(df, chunk_size, table_name, db_config):
    """Load large dataframes in chunks to avoid memory issues"""
    total_chunks = len(df) // chunk_size + (1 if len(df) % chunk_size != 0 else 0)
    
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        chunk_num = i // chunk_size + 1
        
        try:
            connection = mysql.connector.connect(**db_config)
            chunk.to_sql(table_name, connection, if_exists='append', index=False)
            print(f"Loaded chunk {chunk_num}/{total_chunks}")
        except Exception as e:
            print(f"Error loading chunk {chunk_num}: {e}")
        finally:
            if connection.is_connected():
                connection.close()
```

### Handling Missing Data

```python
def clean_artifact_data(df):
    """Clean and handle missing values in artifact data"""
    # Fill numeric nulls with 0
    numeric_cols = df.select_dtypes(include=['int64', 'float64']).columns
    df[numeric_cols] = df[numeric_cols].fillna(0)
    
    # Fill string nulls with 'Unknown'
    string_cols = df.select_dtypes(include=['object']).columns
    df[string_cols] = df[string_cols].fillna('Unknown')
    
    # Remove duplicates based on objectid
    df = df.drop_duplicates(subset=['objectid'], keep='first')
    
    return df
```

This skill provides AI agents with the knowledge to build complete ETL pipelines for museum data, implement SQL analytics, and create interactive dashboards using the Harvard Art Museums API architecture.
