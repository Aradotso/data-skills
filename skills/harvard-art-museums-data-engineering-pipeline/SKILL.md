---
name: harvard-art-museums-data-engineering-pipeline
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL for museum artifacts collection
  - create analytics dashboard for Harvard Art Museums data
  - extract and transform Harvard artifacts API data
  - build Streamlit app for museum collection analytics
  - query and visualize Harvard Art Museums database
  - implement SQL analytics for artifact collections
  - design relational schema for museum API data
---

# Harvard Art Museums Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a production-grade data engineering workflow that:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards

The application showcases real-world ETL patterns, database design, and data visualization techniques used in analytics engineering roles.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Create .env file for configuration
echo "HARVARD_API_KEY=your_api_key_here" > .env
echo "DB_HOST=your_db_host" >> .env
echo "DB_USER=your_db_user" >> .env
echo "DB_PASSWORD=your_db_password" >> .env
echo "DB_NAME=harvard_artifacts" >> .env
```

### Required Dependencies

```txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### Database Configuration

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
cursor = connection.cursor()
```

## Database Schema Design

### Create Tables

```python
def create_tables(cursor):
    """Create normalized relational schema for artifacts data"""
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            objectnumber VARCHAR(100),
            url TEXT,
            lastupdate DATETIME,
            INDEX idx_culture (culture),
            INDEX idx_century (century),
            INDEX idx_classification (classification)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl TEXT,
            primaryimageurl TEXT,
            totalpageviews INT,
            totaluniquepageviews INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
                ON DELETE CASCADE,
            INDEX idx_artifact (artifact_id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
                ON DELETE CASCADE,
            INDEX idx_artifact (artifact_id),
            INDEX idx_color (color)
        )
    """)
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """Extract artifacts with pagination and rate limiting"""
    artifacts = []
    
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
                artifacts.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} records")
            
            # Rate limiting - respect API limits
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return artifacts
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into normalized dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'objectnumber': artifact.get('objectnumber', '')[:100],
            'url': artifact.get('url', ''),
            'lastupdate': artifact.get('lastupdate')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        media_records.append(media)
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color_data in artifact['colors']:
                color = {
                    'artifact_id': artifact.get('id'),
                    'color': color_data.get('color', '')[:50],
                    'spectrum': color_data.get('spectrum', '')[:50],
                    'hue': color_data.get('hue', '')[:50],
                    'percent': color_data.get('percent', 0.0)
                }
                color_records.append(color)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into Database

```python
def load_to_database(df_metadata, df_media, df_colors, connection):
    """Load transformed data with batch inserts"""
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, objectnumber, url, lastupdate)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, 
             totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        if not df_colors.empty:
            color_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(color_query, df_colors.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Exception as e:
        connection.rollback()
        print(f"Error loading data: {e}")
        raise
    finally:
        cursor.close()
```

## Analytical SQL Queries

### Pre-built Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
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
    
    "Top Classifications": """
        SELECT classification, COUNT(*) as total
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY total DESC
        LIMIT 12
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(*) as total_artifacts,
            SUM(CASE WHEN primaryimageurl IS NOT NULL AND primaryimageurl != '' 
                THEN 1 ELSE 0 END) as with_images,
            ROUND(100.0 * SUM(CASE WHEN primaryimageurl IS NOT NULL 
                AND primaryimageurl != '' THEN 1 ELSE 0 END) / COUNT(*), 2) 
                as image_coverage_percent
        FROM artifactmedia
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Popular Artifacts": """
        SELECT m.title, m.culture, m.century, 
               a.totalpageviews, a.totaluniquepageviews
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        WHERE a.totalpageviews > 0
        ORDER BY a.totalpageviews DESC
        LIMIT 20
    """
}

def execute_analytics_query(query_name, connection):
    """Execute analytical query and return results"""
    cursor = connection.cursor()
    
    try:
        query = ANALYTICS_QUERIES[query_name]
        cursor.execute(query)
        
        columns = [desc[0] for desc in cursor.description]
        results = cursor.fetchall()
        
        df = pd.DataFrame(results, columns=columns)
        return df
        
    except Exception as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        cursor.close()
```

## Streamlit Dashboard

### Main Application

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
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualization_page()

def show_data_collection_page():
    """ETL pipeline execution page"""
    st.header("📥 Data Collection & ETL")
    
    num_pages = st.number_input("Number of pages to fetch", 
                                  min_value=1, max_value=20, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(API_KEY, num_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            connection = mysql.connector.connect(**db_config)
            load_to_database(df_meta, df_media, df_colors, connection)
            connection.close()
            st.success("Data loaded to database")
        
        st.dataframe(df_meta.head(10))

def show_analytics_page():
    """SQL analytics execution page"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analytics Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        connection = mysql.connector.connect(**db_config)
        df_result = execute_analytics_query(query_name, connection)
        connection.close()
        
        if not df_result.empty:
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) == 2:
                fig = px.bar(df_result, 
                            x=df_result.columns[0], 
                            y=df_result.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No results found")

if __name__ == "__main__":
    main()
```

### Run the Application

```bash
streamlit run app.py
```

## Common Patterns

### Error Handling with Retry Logic

```python
from functools import wraps
import time

def retry_on_failure(max_retries=3, delay=2):
    """Decorator for API calls with retry logic"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except requests.exceptions.RequestException as e:
                    if attempt == max_retries - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}")
                    time.sleep(delay * (attempt + 1))
            return None
        return wrapper
    return decorator

@retry_on_failure(max_retries=3)
def fetch_with_retry(url, params):
    response = requests.get(url, params=params, timeout=30)
    response.raise_for_status()
    return response.json()
```

### Incremental Data Loading

```python
def get_last_update_timestamp(connection):
    """Get the latest artifact timestamp from database"""
    cursor = connection.cursor()
    cursor.execute("""
        SELECT MAX(lastupdate) FROM artifactmetadata
    """)
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else None

def fetch_incremental_artifacts(api_key, last_update):
    """Fetch only new/updated artifacts"""
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'lastupdate',
        'sortorder': 'desc'
    }
    
    if last_update:
        params['after'] = last_update
    
    response = requests.get(BASE_URL, params=params)
    return response.json().get('records', [])
```

### Data Quality Checks

```python
def validate_artifact_data(df):
    """Perform data quality checks"""
    checks = {
        'missing_ids': df['id'].isna().sum(),
        'duplicate_ids': df['id'].duplicated().sum(),
        'missing_titles': df['title'].isna().sum(),
        'invalid_dates': (df['lastupdate'].isna()).sum()
    }
    
    for check, count in checks.items():
        if count > 0:
            print(f"⚠️ Data Quality Issue: {check} = {count}")
    
    return all(count == 0 for count in checks.values())
```

## Troubleshooting

### API Rate Limiting

```python
# Issue: 429 Too Many Requests
# Solution: Implement exponential backoff

import time
from requests.exceptions import HTTPError

def fetch_with_backoff(url, params, max_retries=5):
    for retry in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = (2 ** retry) + 1
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
```

### Database Connection Issues

```python
# Issue: Lost connection to MySQL server
# Solution: Connection pooling

from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    pool_reset_session=True,
    **db_config
)

def get_connection():
    return db_pool.get_connection()
```

### Memory Issues with Large Datasets

```python
# Issue: Memory error with large API responses
# Solution: Process data in chunks

def process_artifacts_in_chunks(artifacts, chunk_size=100):
    """Process artifacts in batches to manage memory"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        df_meta, df_media, df_colors = transform_artifacts(chunk)
        
        connection = get_connection()
        load_to_database(df_meta, df_media, df_colors, connection)
        connection.close()
        
        # Clear memory
        del df_meta, df_media, df_colors
```

### Empty or Malformed API Responses

```python
def safe_extract_value(data, key, default=''):
    """Safely extract values from nested JSON"""
    try:
        value = data.get(key, default)
        return value if value is not None else default
    except (AttributeError, TypeError):
        return default

# Usage in transform function
metadata = {
    'id': safe_extract_value(artifact, 'id', 0),
    'title': safe_extract_value(artifact, 'title', 'Unknown')[:500],
    'culture': safe_extract_value(artifact, 'culture', 'Unknown')[:255]
}
```

## Performance Optimization

### Batch Insert Optimization

```python
def optimized_batch_insert(df, table_name, connection, batch_size=1000):
    """Insert data in optimized batches"""
    cursor = connection.cursor()
    
    for start in range(0, len(df), batch_size):
        batch = df.iloc[start:start + batch_size]
        placeholders = ', '.join(['%s'] * len(batch.columns))
        query = f"INSERT INTO {table_name} VALUES ({placeholders})"
        
        cursor.executemany(query, batch.values.tolist())
        connection.commit()
    
    cursor.close()
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards with the Harvard Art Museums API, covering all aspects from data extraction to visualization.
