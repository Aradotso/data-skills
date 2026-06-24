---
name: harvard-artifacts-etl-streamlit-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to create a data engineering pipeline with Streamlit
  - help me extract and analyze Harvard art collection data
  - create an analytics dashboard for museum artifact data
  - build a SQL database from Harvard Museums API
  - how to visualize art collection data with Plotly and Streamlit
  - set up an end-to-end data pipeline for API to visualization
  - integrate Harvard Art Museums API with MySQL database
---

# Harvard Artifacts ETL & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill helps you build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL analytics, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection app provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Three normalized tables (artifactmetadata, artifactmedia, artifactcolors)
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations

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

### 1. Harvard Art Museums API Key

Get your free API key from: https://harvardartmuseums.org/collections/api

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Setup (MySQL/TiDB Cloud)

Configure database connection in `.env`:
```bash
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Create the database schema:
```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact Metadata Table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    period VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    credit_line TEXT
);

-- Artifact Media Table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url TEXT,
    image_width INT,
    image_height INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(100),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline Pattern

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Pagination example
def fetch_all_artifacts(max_records=1000):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(page=page, size=size)
        artifacts = data.get('records', [])
        
        if not artifacts:
            break
            
        all_artifacts.extend(artifacts)
        page += 1
        
    return all_artifacts[:max_records]
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata DataFrame"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', ''),
            'credit_line': artifact.get('creditline', '')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Extract media/image data"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        primary_image = artifact.get('primaryimageurl')
        
        if primary_image:
            media_records.append({
                'artifact_id': artifact_id,
                'base_image_url': primary_image,
                'image_width': artifact.get('imagewidth'),
                'image_height': artifact.get('imageheight'),
                'format': 'jpg'
            })
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color data from artifacts"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color_obj in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color_obj.get('color', 'Unknown'),
                'spectrum': color_obj.get('spectrum', 'Unknown'),
                'hue': color_obj.get('hue', 'Unknown'),
                'percent': color_obj.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_records)
```

### Load: Insert into SQL Database

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

def load_metadata(df_metadata):
    """Batch insert metadata into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, technique, period, dated, url, credit_line)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_media(df_media):
    """Insert media records"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, base_image_url, image_width, image_height, format)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_colors(df_colors):
    """Insert color records"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_colors.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(num_records=500):
    """Execute full ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # EXTRACT
    print(f"Extracting {num_records} artifacts from API...")
    artifacts = fetch_all_artifacts(max_records=num_records)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # TRANSFORM
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    print(f"Metadata records: {len(df_metadata)}")
    print(f"Media records: {len(df_media)}")
    print(f"Color records: {len(df_colors)}")
    
    # LOAD
    print("Loading data into database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL Pipeline completed successfully!")
    
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'color_count': len(df_colors)
    }
```

## SQL Analytics Queries

### Top Cultures by Artifact Count

```python
def query_top_cultures(limit=10):
    """Get top cultures by artifact count"""
    conn = get_db_connection()
    
    query = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture != 'Unknown'
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT %s
    """
    
    df = pd.read_sql(query, conn, params=(limit,))
    conn.close()
    return df
```

### Artifacts by Century

```python
def query_artifacts_by_century():
    """Distribution of artifacts across centuries"""
    conn = get_db_connection()
    
    query = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century != 'Unknown'
    GROUP BY century
    ORDER BY count DESC
    """
    
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### Color Analysis

```python
def query_dominant_colors():
    """Most common colors in artifacts"""
    conn = get_db_connection()
    
    query = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 15
    """
    
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### Media Availability

```python
def query_media_stats():
    """Statistics on artifact images"""
    conn = get_db_connection()
    
    query = """
    SELECT 
        COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
        AVG(m.image_width) as avg_width,
        AVG(m.image_height) as avg_height,
        MAX(m.image_width) as max_width,
        MAX(m.image_height) as max_height
    FROM artifactmedia m
    """
    
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar
    st.sidebar.header("Controls")
    action = st.sidebar.selectbox(
        "Select Action",
        ["Run ETL Pipeline", "View Analytics", "Custom Query"]
    )
    
    if action == "Run ETL Pipeline":
        show_etl_page()
    elif action == "View Analytics":
        show_analytics_page()
    else:
        show_custom_query_page()

def show_etl_page():
    st.header("ETL Pipeline")
    
    num_records = st.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=5000,
        value=500,
        step=50
    )
    
    if st.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            results = run_etl_pipeline(num_records)
            st.success("ETL completed!")
            st.json(results)

def show_analytics_page():
    st.header("Analytics Dashboard")
    
    tab1, tab2, tab3 = st.tabs(["Cultures", "Centuries", "Colors"])
    
    with tab1:
        st.subheader("Top Cultures")
        df = query_top_cultures(15)
        
        col1, col2 = st.columns([2, 1])
        with col1:
            fig = px.bar(df, x='culture', y='artifact_count',
                        title='Artifacts by Culture')
            st.plotly_chart(fig, use_container_width=True)
        with col2:
            st.dataframe(df)
    
    with tab2:
        st.subheader("Century Distribution")
        df = query_artifacts_by_century()
        
        fig = px.bar(df, x='century', y='count',
                    title='Artifacts by Century')
        st.plotly_chart(fig, use_container_width=True)
    
    with tab3:
        st.subheader("Color Analysis")
        df = query_dominant_colors()
        
        fig = px.bar(df, x='color', y='frequency',
                    color='avg_percent',
                    title='Most Common Colors')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Interactive Query Runner

```python
def show_custom_query_page():
    st.header("Custom SQL Queries")
    
    predefined_queries = {
        "All Departments": "SELECT DISTINCT department FROM artifactmetadata WHERE department != 'Unknown'",
        "Images by Department": """
            SELECT m.department, COUNT(med.media_id) as image_count
            FROM artifactmetadata m
            JOIN artifactmedia med ON m.id = med.artifact_id
            GROUP BY m.department
            ORDER BY image_count DESC
        """,
        "Color Diversity": """
            SELECT artifact_id, COUNT(DISTINCT color) as color_count
            FROM artifactcolors
            GROUP BY artifact_id
            ORDER BY color_count DESC
            LIMIT 20
        """
    }
    
    query_name = st.selectbox("Select Query", list(predefined_queries.keys()))
    query = predefined_queries[query_name]
    
    st.code(query, language='sql')
    
    if st.button("Execute Query"):
        conn = get_db_connection()
        df = pd.read_sql(query, conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization if applicable
        if len(df.columns) == 2 and df[df.columns[1]].dtype in ['int64', 'float64']:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig, use_container_width=True)
```

## Running the App

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, size, delay=0.5):
    """Fetch with rate limiting to avoid API throttling"""
    data = fetch_artifacts(page=page, size=size)
    time.sleep(delay)  # Wait between requests
    return data
```

### Error Handling in ETL

```python
def safe_etl_pipeline(num_records=500):
    """ETL with comprehensive error handling"""
    try:
        artifacts = fetch_all_artifacts(max_records=num_records)
    except requests.exceptions.RequestException as e:
        st.error(f"API Error: {e}")
        return None
    
    try:
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
    except Exception as e:
        st.error(f"Transformation Error: {e}")
        return None
    
    try:
        load_metadata(df_metadata)
        load_media(df_media)
        load_colors(df_colors)
    except Error as e:
        st.error(f"Database Error: {e}")
        return None
    
    return True
```

### Incremental Loading

```python
def get_max_artifact_id():
    """Get highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    cursor.close()
    conn.close()
    return max_id

def incremental_etl():
    """Only load new artifacts"""
    max_existing_id = get_max_artifact_id()
    artifacts = fetch_all_artifacts(max_records=1000)
    
    # Filter only new artifacts
    new_artifacts = [a for a in artifacts if a['id'] > max_existing_id]
    
    if new_artifacts:
        # Transform and load only new data
        df_metadata = transform_artifact_metadata(new_artifacts)
        load_metadata(df_metadata)
```

## Troubleshooting

**API Key Issues:**
```python
# Verify API key is loaded
import os
from dotenv import load_dotenv
load_dotenv()
print(f"API Key loaded: {os.getenv('HARVARD_API_KEY') is not None}")
```

**Database Connection Errors:**
```python
# Test connection
try:
    conn = get_db_connection()
    print("Database connected successfully!")
    conn.close()
except Error as e:
    print(f"Connection failed: {e}")
```

**Empty Results:**
```python
# Check API response structure
data = fetch_artifacts(page=1, size=10)
print(f"Total records available: {data.get('info', {}).get('totalrecords')}")
print(f"Records fetched: {len(data.get('records', []))}")
```

**Memory Issues with Large Datasets:**
```python
# Process in chunks
def chunked_etl(total_records=10000, chunk_size=500):
    for i in range(0, total_records, chunk_size):
        artifacts = fetch_all_artifacts(max_records=chunk_size)
        # Process chunk
        df = transform_artifact_metadata(artifacts)
        load_metadata(df)
```
