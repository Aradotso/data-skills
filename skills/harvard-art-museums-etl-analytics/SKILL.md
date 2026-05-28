---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build a data pipeline for Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - set up analytics dashboard with Streamlit and museum data
  - extract and transform Harvard Art Museums API data
  - query and visualize artifact collection data
  - implement museum data engineering pipeline
  - analyze Harvard Art Museums collection with SQL
  - build interactive museum artifact analytics
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization using Streamlit.

## What It Does

The application:
- Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations to transform nested JSON into relational database structure
- Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly charts

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+ required
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

### Database Schema Setup

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    objectnumber VARCHAR(100),
    division VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500)
);

CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    height INT,
    width INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            print(f"Error: {response.status_code}")
            break
        
        data = response.json()
        artifacts.extend(data.get('records', []))
        
        if len(data.get('records', [])) == 0:
            break
        
        page += 1
    
    return artifacts[:num_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'dated': artifact.get('dated', '')[:200],
            'department': artifact.get('department', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'objectnumber': artifact.get('objectnumber', '')[:100],
            'division': artifact.get('division', '')[:200],
            'period': artifact.get('period', '')[:200],
            'technique': artifact.get('technique', '')[:500]
        })
        
        # Extract media/images
        if artifact.get('primaryimageurl'):
            media_records.append({
                'mediaid': artifact.get('id'),
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('primaryimageurl', '')[:500],
                'height': artifact.get('height', 0),
                'width': artifact.get('width', 0),
                'format': 'image'
            })
        
        # Extract colors
        for color_data in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color_data.get('color', '')[:50],
                'spectrum': color_data.get('spectrum', '')[:50],
                'percent': color_data.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert dataframes into SQL database"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, department, classification, 
         objectnumber, division, period, technique)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_sql, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_sql = """
            INSERT INTO artifactmedia 
            (mediaid, artifact_id, baseimageurl, height, width, format)
            VALUES (%s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE baseimageurl=VALUES(baseimageurl)
            """
            cursor.executemany(media_sql, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            color_sql = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(color_sql, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. SQL Analytics Queries

```python
def execute_analytics_query(query_name):
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century 
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                COUNT(DISTINCT am.id) as total_artifacts,
                COUNT(DISTINCT media.artifact_id) as with_media,
                ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
            FROM artifactmetadata am
            LEFT JOIN artifactmedia media ON am.id = media.artifact_id
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL AND department != ''
            GROUP BY department
            ORDER BY artifact_count DESC
        """
    }
    
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    try:
        cursor.execute(queries[query_name])
        results = cursor.fetchall()
        return pd.DataFrame(results)
    finally:
        cursor.close()
        conn.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for ETL operations
    st.sidebar.header("Data Collection")
    num_records = st.sidebar.number_input("Number of artifacts to fetch", 
                                          min_value=10, max_value=1000, value=100)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Transformation complete")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded successfully")
    
    # Analytics section
    st.header("📊 Analytics Queries")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Media Availability": "media_availability",
        "Top Colors": "top_colors",
        "Department Distribution": "department_distribution"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            results_df = execute_analytics_query(query_options[selected_query])
            
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df) > 0 and len(results_df.columns) >= 2:
                fig = px.bar(results_df, 
                            x=results_df.columns[0], 
                            y=results_df.columns[1],
                            title=selected_query)
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID in database for incremental loading"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def fetch_new_artifacts(last_id):
    """Fetch only artifacts newer than last_id"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&after={last_id}"
    # Implementation continues...
```

### Error Handling and Retry Logic

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_retry_session():
    """Create requests session with retry logic"""
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

## Troubleshooting

### API Rate Limiting

If you encounter 429 errors:
```python
import time

def fetch_with_rate_limit(url, params, delay=1):
    """Add delay between API calls"""
    time.sleep(delay)
    response = requests.get(url, params=params)
    return response
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        print("Database connection successful")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(total_records, batch_size=100):
    """Process artifacts in batches to avoid memory issues"""
    for offset in range(0, total_records, batch_size):
        artifacts = fetch_artifacts_paginated(offset, batch_size)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Processed {offset + len(artifacts)}/{total_records}")
```

## Advanced Analytics Examples

### Complex Join Query

```python
advanced_query = """
SELECT 
    am.culture,
    am.century,
    COUNT(DISTINCT am.id) as artifact_count,
    COUNT(DISTINCT media.mediaid) as with_images,
    AVG(ac.percent) as avg_color_dominance
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.id = media.artifact_id
LEFT JOIN artifactcolors ac ON am.id = ac.artifact_id
WHERE am.culture IS NOT NULL
GROUP BY am.culture, am.century
HAVING artifact_count > 5
ORDER BY artifact_count DESC
"""
```
