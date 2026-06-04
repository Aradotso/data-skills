---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create ETL workflow for museum artifacts data
  - analyze Harvard museum collection with SQL
  - set up artifact data analytics dashboard
  - extract and transform museum API data
  - build Streamlit analytics app for art collections
  - create museum data engineering project
  - query Harvard artifacts with SQL analytics
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

This project provides an end-to-end data engineering solution for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production ETL patterns, relational database design, SQL analytics, and interactive visualization using Streamlit.

**Architecture Flow:** API → ETL → SQL Database → Analytics → Streamlit Dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. API Key Setup

Obtain your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store it securely in environment variables or Streamlit secrets:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
```

Or using Streamlit secrets (`.streamlit/secrets.toml`):
```toml
[api]
harvard_key = "your_api_key_here"
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
# Database credentials in .env
DB_HOST=your_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Extract artifact data from Harvard API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages with rate limiting
import time

def fetch_all_artifacts(total_records=1000):
    """Fetch artifacts across multiple pages"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        records, info = fetch_artifacts(page=page, size=size)
        all_artifacts.extend(records)
        
        if page >= info['pages']:
            break
            
        page += 1
        time.sleep(0.5)  # Rate limiting
    
    return all_artifacts[:total_records]
```

### 2. ETL Pipeline - Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform nested JSON to flat metadata table"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'artifact_id': artifact.get('id'),
            'object_number': artifact.get('objectnumber'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description', '')[:500]  # Truncate
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Extract media/image information"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height'),
                'copyright': img.get('copyright')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color palette data"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### 3. Load - Database Operations

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL/TiDB connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        raise Exception(f"Database connection failed: {e}")

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            object_number VARCHAR(100),
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            description TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            copyright TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_name VARCHAR(100),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()

def batch_insert_metadata(connection, df):
    """Batch insert metadata with performance optimization"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, object_number, title, culture, period, century, 
         classification, department, dated, description)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
```

### 4. Streamlit Analytics Dashboard

```python
import streamlit as st
import plotly.express as px

def run_analytics_query(connection, query_name):
    """Execute predefined analytical queries"""
    
    queries = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "top_colors": """
            SELECT color_name, COUNT(*) as frequency,
                   AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 15
        """,
        
        "department_distribution": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        
        "image_dimensions": """
            SELECT 
                CASE 
                    WHEN width < 500 THEN 'Small'
                    WHEN width < 1000 THEN 'Medium'
                    ELSE 'Large'
                END as size_category,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY size_category
        """
    }
    
    cursor = connection.cursor()
    cursor.execute(queries[query_name])
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    return pd.DataFrame(results, columns=columns)

# Streamlit app structure
def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # ETL Controls
    st.sidebar.header("Data Pipeline")
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Extracting data..."):
            artifacts = fetch_all_artifacts(total_records=500)
            
        with st.spinner("Transforming data..."):
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            
        with st.spinner("Loading to database..."):
            conn = create_database_connection()
            create_tables(conn)
            batch_insert_metadata(conn, metadata_df)
            # Similar for media and colors
            
        st.success("ETL Pipeline completed!")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    query_choice = st.selectbox(
        "Select Analysis",
        ["artifacts_by_culture", "artifacts_by_century", 
         "top_colors", "department_distribution"]
    )
    
    if st.button("Run Analysis"):
        conn = create_database_connection()
        results_df = run_analytics_query(conn, query_choice)
        
        st.dataframe(results_df)
        
        # Auto-generate visualization
        if len(results_df.columns) == 2:
            fig = px.bar(results_df, x=results_df.columns[0], 
                        y=results_df.columns[1],
                        title=query_choice.replace('_', ' ').title())
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental ETL

```python
def incremental_etl(connection, last_run_timestamp):
    """Only fetch artifacts updated since last run"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'modifiedafter': last_run_timestamp
    }
    # Fetch and process only new/updated records
```

### Pattern 2: Error Handling in Pipeline

```python
def safe_etl_pipeline():
    """ETL with comprehensive error handling"""
    try:
        artifacts = fetch_all_artifacts()
    except Exception as e:
        st.error(f"Extraction failed: {e}")
        return
    
    try:
        df = transform_artifact_metadata(artifacts)
    except Exception as e:
        st.error(f"Transformation failed: {e}")
        return
    
    try:
        conn = create_database_connection()
        batch_insert_metadata(conn, df)
    except Exception as e:
        st.error(f"Load failed: {e}")
        return
    
    st.success("Pipeline completed successfully!")
```

## Troubleshooting

**API Rate Limiting:**
- Add `time.sleep(0.5)` between requests
- Use `hasimage=1` parameter to reduce data volume

**Database Connection Issues:**
- Verify credentials in `.env` file
- Check firewall rules for TiDB Cloud
- Use connection pooling for production

**Memory Issues with Large Datasets:**
- Process data in batches of 100-500 records
- Use `chunksize` parameter in pandas operations

**Streamlit Performance:**
- Use `@st.cache_data` for expensive operations
- Store connection in `st.session_state`

```python
@st.cache_data
def load_cached_artifacts():
    return fetch_all_artifacts(total_records=1000)
```
