---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - analyze Harvard artifacts data with SQL queries
  - create a Streamlit dashboard for museum artifact analytics
  - extract and transform Harvard Art Museums collection data
  - build data engineering pipeline for art museum artifacts
  - visualize Harvard museum data with Plotly and Streamlit
  - setup SQL database for Harvard artifacts collection
  - implement batch data ingestion from museum API
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-ready ETL patterns including API pagination, nested JSON transformation into relational tables, batch SQL operations, and interactive Streamlit dashboards with Plotly visualizations.

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Connection

```python
import mysql.connector
import os
from sqlalchemy import create_engine

# MySQL Connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# SQLAlchemy Engine for Pandas
def get_engine():
    db_url = f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    return create_engine(db_url)
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Key Components

### 1. API Data Extraction

```python
import requests
import time

def fetch_artifacts(api_key, size=100, max_pages=10):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    
    for page in range(1, max_pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                artifacts.extend(data['records'])
                
            # Rate limiting
            time.sleep(0.5)
            
            # Check if more pages available
            if data.get('info', {}).get('next') is None:
                break
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts
```

### 2. ETL Pipeline - Transform Nested JSON

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into three relational tables
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Artifact Metadata Table
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_records.append(metadata)
        
        # Artifact Media Table
        images = artifact.get('images', [])
        for image in images:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'width': image.get('width'),
                'height': image.get('height')
            }
            media_records.append(media)
        
        # Artifact Colors Table
        colors = artifact.get('colors', [])
        for color in colors:
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

### 3. Database Schema Creation

```python
def create_tables(connection):
    """
    Create relational database schema
    """
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            dated VARCHAR(255),
            classification VARCHAR(255),
            medium TEXT,
            department VARCHAR(255),
            division VARCHAR(255),
            creditline TEXT,
            accessionyear INT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl TEXT,
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Batch Data Loading

```python
def load_to_database(df_metadata, df_media, df_colors, engine):
    """
    Batch insert data using pandas to_sql
    """
    # Load metadata (replace if exists)
    df_metadata.to_sql(
        'artifactmetadata',
        con=engine,
        if_exists='replace',
        index=False,
        method='multi',
        chunksize=1000
    )
    
    # Load media
    df_media.to_sql(
        'artifactmedia',
        con=engine,
        if_exists='replace',
        index=False,
        method='multi',
        chunksize=1000
    )
    
    # Load colors
    df_colors.to_sql(
        'artifactcolors',
        con=engine,
        if_exists='replace',
        index=False,
        method='multi',
        chunksize=1000
    )
```

### 5. Analytical Queries

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
    
    "Artifacts with Images": """
        SELECT 
            COUNT(DISTINCT am.id) as with_images,
            (SELECT COUNT(*) FROM artifactmetadata) as total,
            ROUND(COUNT(DISTINCT am.id) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmetadata am
        INNER JOIN artifactmedia media ON am.id = media.artifact_id
    """,
    
    "Most Common Color Spectrum": """
        SELECT spectrum, COUNT(*) as frequency
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY frequency DESC
    """,
    
    "Average Colors per Artifact": """
        SELECT AVG(color_count) as avg_colors
        FROM (
            SELECT artifact_id, COUNT(*) as color_count
            FROM artifactcolors
            GROUP BY artifact_id
        ) as color_counts
    """,
    
    "Top 10 Classifications": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_query(query, connection):
    """
    Execute analytical query and return DataFrame
    """
    return pd.read_sql(query, connection)
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    
    # ETL Section
    st.header("1. Data Collection & ETL")
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(api_key, size=100, max_pages=5)
            st.success(f"Fetched {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(raw_data)
            st.success(f"Transformed into {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} color records")
        
        with st.spinner("Loading to database..."):
            engine = get_engine()
            load_to_database(df_metadata, df_media, df_colors, engine)
            st.success("Data loaded successfully!")
    
    # Analytics Section
    st.header("2. SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        connection = get_db_connection()
        query = ANALYTICAL_QUERIES[query_name]
        
        st.code(query, language="sql")
        
        df_result = execute_query(query, connection)
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2 and len(df_result) > 0:
            fig = px.bar(
                df_result,
                x=df_result.columns[0],
                y=df_result.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)
        
        connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time
from functools import wraps

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
def fetch_single_page(api_key, page):
    # API call implementation
    pass
```

### Handling Missing Data

```python
def clean_dataframe(df):
    """Clean and prepare DataFrame for SQL insertion"""
    # Replace NaN with None for SQL NULL
    df = df.where(pd.notnull(df), None)
    
    # Truncate long text fields
    text_columns = ['title', 'medium', 'creditline']
    for col in text_columns:
        if col in df.columns:
            df[col] = df[col].astype(str).str[:1000]
    
    return df
```

## Troubleshooting

### API Key Issues

```python
# Test API connection
def test_api_connection(api_key):
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1},
            timeout=10
        )
        return response.status_code == 200
    except:
        return False
```

### Database Connection Errors

```python
# Retry logic for database operations
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=2, max=10))
def connect_with_retry():
    return get_db_connection()
```

### Memory Management for Large Datasets

```python
def fetch_artifacts_chunked(api_key, total_pages=100, chunk_size=10):
    """Process data in chunks to manage memory"""
    for start_page in range(1, total_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size, total_pages + 1)
        chunk_data = []
        
        for page in range(start_page, end_page):
            # Fetch and process
            artifacts = fetch_artifacts(api_key, size=100, max_pages=1)
            chunk_data.extend(artifacts)
        
        # Transform and load chunk
        df_metadata, df_media, df_colors = transform_artifacts(chunk_data)
        load_to_database(df_metadata, df_media, df_colors, get_engine())
        
        # Clear memory
        del chunk_data
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement pagination** for API requests to handle large datasets
3. **Use batch inserts** with `method='multi'` and `chunksize` for performance
4. **Add indexes** on foreign keys for query performance
5. **Validate data** before insertion to prevent SQL errors
6. **Log ETL operations** for debugging and monitoring
7. **Handle API rate limits** gracefully with sleep intervals
8. **Use transactions** for atomic database operations
