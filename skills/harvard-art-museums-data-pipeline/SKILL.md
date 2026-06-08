---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for Harvard Art Museums
  - create ETL pipeline with Harvard API
  - analyze Harvard art collection data
  - set up museum artifact data engineering
  - visualize Harvard museum data with Streamlit
  - query Harvard art museums database
  - extract and transform museum API data
  - build analytics dashboard for art collections
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering & Analytics App is an end-to-end data pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB), and provides interactive analytics through a Streamlit dashboard. The project demonstrates production-grade ETL patterns, SQL analytics, and data visualization.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

### Required Dependencies

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
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

def get_db_connection():
    return mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    object_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    copyright VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact Media Table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    media_type VARCHAR(50),
    media_url TEXT,
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);

-- Artifact Colors Table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);
```

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    """
    all_records = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(BASE_URL, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_records.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} records")
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_records
```

### Transform: Parse and Structure Data

```python
import pandas as pd

def transform_artifacts(records):
    """
    Transform raw API records into structured dataframes.
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        object_id = record.get('objectid')
        
        # Metadata
        metadata_list.append({
            'object_id': object_id,
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'medium': record.get('medium'),
            'technique': record.get('technique'),
            'dimensions': record.get('dimensions'),
            'copyright': record.get('copyright'),
            'creditline': record.get('creditline'),
            'accession_number': record.get('accessionyear'),
            'url': record.get('url')
        })
        
        # Media
        if 'images' in record and record['images']:
            for img in record['images']:
                media_list.append({
                    'object_id': object_id,
                    'media_type': 'image',
                    'media_url': img.get('baseimageurl')
                })
        
        # Colors
        if 'colors' in record and record['colors']:
            for color in record['colors']:
                colors_list.append({
                    'object_id': object_id,
                    'color_hex': color.get('hex'),
                    'color_name': color.get('color'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert dataframes into SQL database.
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_tuples = [tuple(x) for x in metadata_df.to_numpy()]
        insert_metadata_query = """
            INSERT IGNORE INTO artifactmetadata 
            (object_id, title, culture, century, dated, classification, 
             department, division, medium, technique, dimensions, 
             copyright, creditline, accession_number, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        cursor.executemany(insert_metadata_query, metadata_tuples)
        
        # Insert media
        if not media_df.empty:
            media_tuples = [tuple(x) for x in media_df.to_numpy()]
            insert_media_query = """
                INSERT INTO artifactmedia (object_id, media_type, media_url)
                VALUES (%s, %s, %s)
            """
            cursor.executemany(insert_media_query, media_tuples)
        
        # Insert colors
        if not colors_df.empty:
            colors_tuples = [tuple(x) for x in colors_df.to_numpy()]
            insert_colors_query = """
                INSERT INTO artifactcolors (object_id, color_hex, color_name, percentage)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(insert_colors_query, colors_tuples)
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Exception as e:
        conn.rollback()
        print(f"Error loading data: {e}")
        raise
    finally:
        cursor.close()
        conn.close()
```

## Analytics Queries

### Sample SQL Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.object_id) as artifacts_with_media,
            COUNT(DISTINCT meta.object_id) as total_artifacts
        FROM artifactmetadata meta
        LEFT JOIN artifactmedia am ON meta.object_id = am.object_id
    """,
    
    "Most Common Colors": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """
}

def execute_query(query_name):
    """Execute an analytics query and return results as DataFrame."""
    conn = get_db_connection()
    try:
        df = pd.read_sql(ANALYTICS_QUERIES[query_name], conn)
        return df
    finally:
        conn.close()
```

## Streamlit Dashboard

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar
    st.sidebar.header("Configuration")
    
    # ETL Section
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            records = fetch_artifacts(os.getenv('HARVARD_API_KEY'), num_pages=3)
            st.success(f"Fetched {len(records)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(records)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded successfully!")
    
    # Analytics Section
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            result_df = execute_query(query_name)
            
            st.subheader("Query Results")
            st.dataframe(result_df, use_container_width=True)
            
            # Auto-generate visualization
            if len(result_df.columns) >= 2:
                fig = px.bar(
                    result_df, 
                    x=result_df.columns[0], 
                    y=result_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Error Handling for API Requests

```python
def safe_api_call(url, params, max_retries=3):
    """Make API call with retry logic."""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=30)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.Timeout:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise
        except requests.exceptions.RequestException as e:
            st.error(f"API Error: {e}")
            return None
```

### Batch Processing for Large Datasets

```python
def batch_insert(cursor, query, data, batch_size=1000):
    """Insert data in batches for better performance."""
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(query, batch)
        print(f"Inserted batch {i//batch_size + 1}")
```

## Troubleshooting

### API Rate Limiting
- Add `time.sleep(0.5)` between requests
- Implement exponential backoff for retries
- Monitor API key quota

### Database Connection Issues
```python
# Test connection
try:
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    print("Database connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
```

### Memory Issues with Large Datasets
- Process data in chunks
- Use `pd.read_sql` with `chunksize` parameter
- Clear dataframes after loading: `del df; gc.collect()`

### Streamlit Caching
```python
@st.cache_data(ttl=3600)
def cached_query(query_name):
    """Cache query results for 1 hour."""
    return execute_query(query_name)
```

This skill enables AI agents to help developers build complete data engineering pipelines using the Harvard Art Museums API, from data extraction to interactive visualization.
