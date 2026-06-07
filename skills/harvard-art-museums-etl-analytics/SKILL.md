---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard Art Museums API
  - extract and transform museum artifact data
  - set up SQL database for art collection analysis
  - visualize Harvard Art Museums data with Streamlit
  - query and analyze museum artifacts with SQL
  - integrate Harvard Art Museums API into data pipeline
  - build museum data engineering application
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data
- **SQL Storage**: Structured relational database with proper foreign key relationships
- **Analytics Queries**: 20+ predefined SQL queries for insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# MySQL or TiDB Cloud instance
```

### Setup Steps

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Configure environment variables
# Create .env file or set environment variables:
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_database_host
# DB_USER=your_database_user
# DB_PASSWORD=your_database_password
# DB_NAME=your_database_name
```

### Dependencies

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
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    division VARCHAR(255),
    dated VARCHAR(255),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10):
    """Fetch artifacts with pagination from Harvard Art Museums API"""
    artifacts = []
    base_url = "https://api.harvardartmuseums.org/object"
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'page': page,
            'size': 100  # Max results per page
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
            break
    
    return artifacts
```

### Transform: Process JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform nested JSON into relational dataframes"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Metadata extraction
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', '')
        })
        
        # Media extraction
        images = artifact.get('images', [])
        for image in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl', ''),
                'iiifbaseuri': image.get('iiifbaseuri', '')
            })
        
        # Color extraction
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into SQL Database

```python
def load_to_database(df_metadata, df_media, df_colors, connection):
    """Batch insert dataframes into MySQL tables"""
    cursor = connection.cursor()
    
    # Load metadata
    for _, row in df_metadata.iterrows():
        sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, technique, division, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), period=VALUES(period)
        """
        cursor.execute(sql, tuple(row))
    
    # Load media
    for _, row in df_media.iterrows():
        sql = """
        INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    # Load colors
    for _, row in df_colors.iterrows():
        sql = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    connection.commit()
    cursor.close()
```

## Analytics Queries

### Example SQL Analytics

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Query 2: Artifacts with images
query_images = """
SELECT 
    am.classification,
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT media.artifact_id) as artifacts_with_images,
    ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.id = media.artifact_id
GROUP BY am.classification
ORDER BY total_artifacts DESC
LIMIT 10
"""

# Query 3: Color distribution
query_colors = """
SELECT 
    spectrum,
    COUNT(*) as frequency,
    ROUND(AVG(percent), 2) as avg_percent
FROM artifactcolors
GROUP BY spectrum
ORDER BY frequency DESC
"""

# Query 4: Artifacts by century
query_century = """
SELECT 
    century,
    COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC
LIMIT 15
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
import os

st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")

# Sidebar configuration
st.sidebar.title("Configuration")
api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                 value=os.getenv('HARVARD_API_KEY', ''))

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Main tabs
tab1, tab2, tab3 = st.tabs(["ETL Pipeline", "SQL Analytics", "Visualizations"])

with tab1:
    st.header("ETL Pipeline")
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(api_key, num_pages=5)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = get_db_connection()
            load_to_database(df_meta, df_media, df_colors, conn)
            conn.close()
            st.success("Data loaded to database")

with tab2:
    st.header("SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Image Coverage": query_images,
        "Color Distribution": query_colors,
        "Artifacts by Century": query_century
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        conn = get_db_connection()
        df_result = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig, use_container_width=True)

with tab3:
    st.header("Interactive Visualizations")
    # Add custom visualizations
```

### Run the Application

```bash
streamlit run app.py
```

## Common Patterns

### Incremental ETL

```python
def get_last_synced_id(connection):
    """Get the highest artifact ID already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_fetch(api_key, last_id):
    """Fetch only new artifacts since last sync"""
    # Add filter parameter to API call
    params = {
        'apikey': api_key,
        'q': f'id:>{last_id}',
        'size': 100
    }
    # Continue with fetch logic
```

### Error Handling

```python
def safe_etl_pipeline(api_key, connection):
    """ETL with comprehensive error handling"""
    try:
        artifacts = fetch_artifacts(api_key)
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        load_to_database(df_meta, df_media, df_colors, connection)
        return {"status": "success", "records": len(artifacts)}
    except requests.exceptions.RequestException as e:
        return {"status": "error", "message": f"API error: {str(e)}"}
    except mysql.connector.Error as e:
        return {"status": "error", "message": f"Database error: {str(e)}"}
    except Exception as e:
        return {"status": "error", "message": f"Unexpected error: {str(e)}"}
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            if response.status_code == 429:  # Too many requests
                wait_time = 2 ** attempt
                time.sleep(wait_time)
                continue
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
    return None
```

### Database Connection Issues
```python
# Connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return db_pool.get_connection()
```

### Memory Issues with Large Datasets
```python
# Batch processing
def batch_load(df, connection, batch_size=1000):
    """Load data in batches to avoid memory issues"""
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        load_to_database(batch, connection)
        connection.commit()
```

## Key Commands

```bash
# Run Streamlit app
streamlit run app.py

# Run ETL pipeline independently
python etl_pipeline.py

# Run SQL analytics
python analytics.py

# Export query results
python export_results.py --query "cultures" --format csv
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement pagination** when fetching large datasets from the API
3. **Use transactions** for database operations to ensure data consistency
4. **Cache database connections** in Streamlit with `@st.cache_resource`
5. **Validate data** before inserting into database
6. **Log ETL operations** for debugging and monitoring
7. **Create indexes** on frequently queried columns in SQL tables
