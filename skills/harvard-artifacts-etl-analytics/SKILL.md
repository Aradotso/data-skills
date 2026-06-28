---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, MySQL, and Streamlit
triggers:
  - how do i build an etl pipeline with harvard art museums data
  - create analytics dashboard for museum artifacts
  - extract and load harvard api data to mysql
  - build streamlit app with harvard art museums api
  - query and visualize museum artifact data
  - set up data engineering pipeline for art collections
  - analyze harvard museum artifacts with sql
  - create data pipeline from api to database visualization
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud databases
- Provides 20+ analytical SQL queries for artifact insights
- Visualizes results using Streamlit and Plotly dashboards

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file in the project root:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
MYSQL_HOST=your_mysql_host
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    century VARCHAR(100),
    period VARCHAR(200),
    primaryimageurl TEXT,
    url TEXT
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    mediaid INT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(100),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

def collect_artifacts_batch(num_pages=5):
    """Collect multiple pages of artifacts"""
    api_key = os.getenv('HARVARD_API_KEY')
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """Transform nested JSON into relational dataframes"""
    
    metadata_list = []
    media_list = []
    color_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'period': artifact.get('period', 'Unknown')[:200],
            'primaryimageurl': artifact.get('primaryimageurl'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media
        for media in artifact.get('images', []):
            media_list.append({
                'mediaid': media.get('imageid'),
                'artifact_id': artifact.get('id'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format'),
                'height': media.get('height'),
                'width': media.get('width')
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(color_list)
    )

def load_to_mysql(df_metadata, df_media, df_colors):
    """Load dataframes to MySQL database"""
    
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        port=os.getenv('MYSQL_PORT'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    cursor = conn.cursor()
    
    # Batch insert metadata
    metadata_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, classification, department, dated, century, period, primaryimageurl, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, df_metadata.values.tolist())
    
    # Batch insert media
    if not df_media.empty:
        media_query = """
        INSERT INTO artifactmedia 
        (mediaid, artifact_id, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE baseimageurl=VALUES(baseimageurl)
        """
        cursor.executemany(media_query, df_media.values.tolist())
    
    # Batch insert colors
    if not df_colors.empty:
        color_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(color_query, df_colors.values.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return len(df_metadata)
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
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
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as occurrence, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrence DESC
        LIMIT 10
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Artifacts with Multiple Images": """
        SELECT a.title, a.culture, COUNT(m.mediaid) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title, a.culture
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_coverage
        FROM artifactcolors
        GROUP BY spectrum
        ORDER BY count DESC
    """
}

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results as dataframe"""
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        port=os.getenv('MYSQL_PORT'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for ETL operations
    st.sidebar.header("Data Collection")
    
    num_pages = st.sidebar.number_input("Pages to collect", min_value=1, max_value=50, value=5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Collecting artifacts..."):
            artifacts = collect_artifacts_batch(num_pages)
            st.sidebar.success(f"Collected {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
        
        with st.spinner("Loading to database..."):
            count = load_to_mysql(df_meta, df_media, df_colors)
            st.sidebar.success(f"Loaded {count} artifacts to database")
    
    # Analytics section
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Running query..."):
            results = execute_query(query)
        
        st.subheader("Results")
        st.dataframe(results, use_container_width=True)
        
        # Auto-generate visualization
        if len(results.columns) == 2:
            fig = px.bar(
                results, 
                x=results.columns[0], 
                y=results.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """Fetch with rate limiting to avoid API throttling"""
    time.sleep(delay)
    return fetch_artifacts(api_key, page)
```

### Error Handling in ETL

```python
def safe_transform(raw_data):
    """Transform with error handling"""
    try:
        df_meta, df_media, df_colors = transform_artifacts(raw_data)
        return df_meta, df_media, df_colors
    except Exception as e:
        print(f"Transform error: {e}")
        return pd.DataFrame(), pd.DataFrame(), pd.DataFrame()
```

### Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('MYSQL_HOST'),
    user=os.getenv('MYSQL_USER'),
    password=os.getenv('MYSQL_PASSWORD'),
    database=os.getenv('MYSQL_DATABASE')
)

def get_connection():
    return db_pool.get_connection()
```

## Troubleshooting

### API Key Issues
- Verify `.env` file exists and contains `HARVARD_API_KEY`
- Check API key validity at Harvard Art Museums portal
- Ensure `python-dotenv` is installed

### Database Connection Errors
- Verify MySQL credentials in `.env`
- Check if database `harvard_artifacts` exists
- Ensure firewall allows connection to MySQL port
- For TiDB Cloud, verify SSL/TLS settings

### Data Loading Failures
- Check for duplicate primary keys (artifact IDs)
- Verify foreign key constraints are satisfied
- Ensure data types match table schema
- Review string length limits (VARCHAR constraints)

### Streamlit Performance
```python
# Cache database connections
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(...)

# Cache query results
@st.cache_data(ttl=600)
def cached_query(query):
    return execute_query(query)
```

### Empty Results
- Verify data has been loaded: `SELECT COUNT(*) FROM artifactmetadata`
- Check API response for `hasimage=1` filter
- Ensure pagination is working correctly

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

The dashboard will provide:
1. ETL controls for data collection
2. Query selection dropdown
3. Tabular results display
4. Auto-generated visualizations
