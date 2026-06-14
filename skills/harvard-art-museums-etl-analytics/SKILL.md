---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifacts
  - set up data engineering pipeline with API and SQL
  - visualize Harvard museum collection data
  - extract and analyze art museum metadata
  - build Streamlit app for artifact analytics
  - design SQL schema for museum collections
  - implement batch data ingestion from Harvard API
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipeline construction, SQL database design, analytical querying, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms into relational format, loads into SQL database
- **Database Design**: Three-table schema (metadata, media, colors) with foreign key relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

## Architecture Flow

```
Harvard API → Extract (Python/Requests) → Transform (Pandas) → Load (MySQL/TiDB) → Analytics (SQL) → Visualize (Streamlit/Plotly)
```

## Installation

### Prerequisites

```bash
# Python 3.8+
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get API key from: https://docs.google.com/forms/d/1Fe1H4nOhFkrLpaeBpLAnSrIMYvcAxnYWm0IU9a6IkFA/viewform

### Database Setup

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    technique VARCHAR(255),
    dated VARCHAR(255),
    accession_year INT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    caption TEXT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard API with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

def collect_all_artifacts(max_pages=10):
    """Collect multiple pages of artifact data"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        
        if page >= info['pages']:
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(artifacts):
    """Transform nested JSON to flat DataFrames"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'accession_year': artifact.get('accessionyear'),
            'url': artifact.get('url')
        })
        
        # Extract media (images)
        if artifact.get('images'):
            for img in artifact['images']:
                media_records.append({
                    'objectid': artifact.get('objectid'),
                    'image_url': img.get('baseimageurl'),
                    'media_type': 'image',
                    'caption': img.get('caption')
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'objectid': artifact.get('objectid'),
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load DataFrames to MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in df_metadata.iterrows():
            sql = """INSERT INTO artifactmetadata 
                     (objectid, title, culture, period, century, division, 
                      department, classification, technique, dated, accession_year, url)
                     VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                     ON DUPLICATE KEY UPDATE title=VALUES(title)"""
            cursor.execute(sql, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            sql = """INSERT INTO artifactmedia 
                     (objectid, image_url, media_type, caption)
                     VALUES (%s, %s, %s, %s)"""
            cursor.execute(sql, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            sql = """INSERT INTO artifactcolors 
                     (objectid, color_hex, color_percent)
                     VALUES (%s, %s, %s)"""
            cursor.execute(sql, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
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
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors Used": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.objectid, m.title, COUNT(a.image_url) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.objectid = a.objectid
        GROUP BY m.objectid, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Accession Year Trends": """
        SELECT accession_year, COUNT(*) as artifacts_acquired
        FROM artifactmetadata
        WHERE accession_year IS NOT NULL
        GROUP BY accession_year
        ORDER BY accession_year DESC
    """
}

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Select Analysis")
query_name = st.sidebar.selectbox(
    "Choose a query:",
    list(ANALYTICAL_QUERIES.keys())
)

# Execute selected query
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        query = ANALYTICAL_QUERIES[query_name]
        df_result = execute_query(query)
        
        st.subheader(f"Results: {query_name}")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(
                df_result,
                x=df_result.columns[0],
                y=df_result.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

# ETL Section
st.sidebar.header("Data Collection")
if st.sidebar.button("Fetch New Data"):
    with st.spinner("Fetching from API..."):
        artifacts = collect_all_artifacts(max_pages=5)
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        load_to_database(df_meta, df_media, df_colors)
        st.success(f"Loaded {len(df_meta)} artifacts!")
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(df, table_name, batch_size=1000):
    """Insert data in batches for better performance"""
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch logic
        connection.commit()
    
    cursor.close()
    connection.close()
```

### Error Handling for API Calls

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise e
```

## Troubleshooting

**API Rate Limiting**: Harvard API has rate limits. Add delays between requests:
```python
time.sleep(0.5)  # 500ms between requests
```

**Database Connection Errors**: Verify credentials and network access to database host.

**Missing Data in Visualizations**: Check for NULL values in SQL queries using `WHERE column IS NOT NULL`.

**Streamlit Performance**: Use `@st.cache_data` for expensive operations:
```python
@st.cache_data
def load_cached_query(query):
    return execute_query(query)
```

**Memory Issues**: Process data in chunks when dealing with 10k+ artifacts.
