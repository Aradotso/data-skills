---
name: harvard-art-museums-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard Art Museums API
  - set up analytics dashboard for museum artifacts
  - extract and transform Harvard museum collection data
  - build a Streamlit app for art museum analytics
  - design SQL schema for museum artifact data
  - query and visualize Harvard Art Museums collection
  - implement pagination for Harvard Art Museums API
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

The Harvard Art Museums ETL Analytics App:
- Fetches artifact data from Harvard Art Museums API with pagination
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on museum collections
- Visualizes results using Plotly in a Streamlit dashboard

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your-api-key-here"
export DB_HOST="your-database-host"
export DB_USER="your-database-user"
export DB_PASSWORD="your-database-password"
export DB_NAME="harvard_artifacts"
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

Obtain an API key from Harvard Art Museums: https://www.harvardartmuseums.org/collections/api

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
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

## Database Schema

### Create Tables

```python
def create_tables(cursor):
    """Create normalized tables for artifact data"""
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            objectnumber VARCHAR(100),
            dated VARCHAR(255),
            medium VARCHAR(500),
            technique VARCHAR(500),
            creditline TEXT,
            provenance TEXT,
            description TEXT,
            primaryimageurl VARCHAR(1000),
            INDEX idx_culture (culture),
            INDEX idx_century (century),
            INDEX idx_department (department)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl VARCHAR(1000),
            format VARCHAR(50),
            description TEXT,
            technique VARCHAR(255),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
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
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
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
    """Fetch artifacts with pagination and rate limiting"""
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
        
        # Rate limiting - be respectful to API
        time.sleep(0.5)
    
    return artifacts
```

### Transform: Normalize JSON Data

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
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'objectnumber': artifact.get('objectnumber'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'creditline': artifact.get('creditline'),
            'provenance': artifact.get('provenance'),
            'description': artifact.get('description'),
            'primaryimageurl': artifact.get('primaryimageurl')
        }
        metadata_records.append(metadata)
        
        # Extract media (nested array)
        for media in artifact.get('images', []):
            media_record = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format'),
                'description': media.get('description'),
                'technique': media.get('technique')
            }
            media_records.append(media_record)
        
        # Extract colors (nested array)
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Batch Insert into SQL

```python
def load_to_database(cursor, conn, metadata_df, media_df, colors_df):
    """Batch insert data into database tables"""
    
    # Insert metadata
    metadata_query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         division, objectnumber, dated, medium, technique, creditline, 
         provenance, description, primaryimageurl)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    if not media_df.empty:
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, format, description, technique)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    conn.commit()
    print(f"Loaded {len(metadata_df)} artifacts with media and color data")
```

## Analytics Queries

### Sample Analytical Queries

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
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Artifacts with Multiple Images": """
        SELECT m.title, m.culture, COUNT(am.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia am ON m.id = am.artifact_id
        GROUP BY m.id, m.title, m.culture
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """
}

def execute_analytics_query(cursor, query_name):
    """Execute analytical query and return results as dataframe"""
    query = ANALYTICS_QUERIES.get(query_name)
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("*ETL Pipeline & Data Analytics Application*")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        st.sidebar.success("✅ Database Connected")
    except Exception as e:
        st.sidebar.error(f"❌ Database Error: {e}")
        return
    
    # Navigation
    page = st.sidebar.radio("Navigation", ["ETL Pipeline", "Analytics", "Data Explorer"])
    
    if page == "ETL Pipeline":
        show_etl_page(cursor, conn)
    elif page == "Analytics":
        show_analytics_page(cursor)
    elif page == "Data Explorer":
        show_data_explorer(cursor)

def show_etl_page(cursor, conn):
    """ETL pipeline execution page"""
    st.header("📥 ETL Pipeline")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=20, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts from API..."):
            artifacts = fetch_artifacts(API_KEY, num_pages=num_pages)
            st.success(f"✅ Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("✅ Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(cursor, conn, metadata_df, media_df, colors_df)
            st.success("✅ Data loaded to database")

def show_analytics_page(cursor):
    """Analytics dashboard with visualizations"""
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        df = execute_analytics_query(cursor, query_name)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

def show_data_explorer(cursor):
    """Interactive data exploration"""
    st.header("🔍 Data Explorer")
    
    table = st.selectbox("Select Table", ["artifactmetadata", "artifactmedia", "artifactcolors"])
    limit = st.slider("Number of rows", 10, 500, 100)
    
    cursor.execute(f"SELECT * FROM {table} LIMIT {limit}")
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    df = pd.DataFrame(results, columns=columns)
    
    st.dataframe(df)
    st.info(f"Total rows: {len(df)}")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate-Limited API Calls

```python
from time import sleep
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = 1.0 / calls_per_second - elapsed
            if wait_time > 0:
                sleep(wait_time)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_single_artifact(artifact_id):
    response = requests.get(f"{BASE_URL}/{artifact_id}", params={'apikey': API_KEY})
    return response.json()
```

### Error Handling for ETL

```python
def safe_etl_pipeline(api_key, num_pages):
    """ETL pipeline with comprehensive error handling"""
    try:
        artifacts = fetch_artifacts(api_key, num_pages)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        
        load_to_database(cursor, conn, metadata_df, media_df, colors_df)
        
        cursor.close()
        conn.close()
        
        return True, f"Successfully processed {len(artifacts)} artifacts"
    
    except requests.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

## Troubleshooting

### API Issues

**Rate Limiting Errors**: Add delays between requests using `time.sleep()` or implement exponential backoff.

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Too many requests
            wait_time = 2 ** attempt
            time.sleep(wait_time)
        else:
            break
    return None
```

### Database Connection Issues

**Connection Timeouts**: Implement connection pooling or auto-reconnect.

```python
def get_db_connection():
    """Get database connection with retry logic"""
    max_attempts = 3
    for attempt in range(max_attempts):
        try:
            conn = mysql.connector.connect(**db_config)
            return conn
        except mysql.connector.Error as e:
            if attempt < max_attempts - 1:
                time.sleep(2)
            else:
                raise e
```

### Data Quality Issues

**Missing or Null Values**: Handle gracefully during transformation.

```python
def clean_artifact_data(artifact):
    """Clean and validate artifact data"""
    return {
        'id': artifact.get('id', 0),
        'title': artifact.get('title', 'Unknown')[:500],
        'culture': artifact.get('culture', 'Unknown')[:255],
        'century': artifact.get('century', 'Unknown')[:100],
        # Truncate strings to fit database schema
    }
```

### Performance Optimization

**Batch Processing**: Process artifacts in batches to reduce memory usage.

```python
def process_in_batches(artifacts, batch_size=100):
    """Process artifacts in batches"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        load_to_database(cursor, conn, metadata_df, media_df, colors_df)
        print(f"Processed batch {i//batch_size + 1}")
```

This skill covers the complete workflow for building ETL pipelines and analytics applications using the Harvard Art Museums API.
