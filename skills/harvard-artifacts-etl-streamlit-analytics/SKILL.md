---
name: harvard-artifacts-etl-streamlit-analytics
description: End-to-end ETL pipeline for Harvard Art Museums API with SQL analytics and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a Streamlit dashboard for art collection analytics
  - extract and analyze Harvard museum artifacts with SQL
  - set up data engineering pipeline with museum API
  - visualize Harvard art collection data with interactive charts
  - design ETL workflow for museum artifact metadata
  - query and analyze art museum data with Python and SQL
  - build analytics app for cultural heritage collections
---

# Harvard Artifacts ETL & Analytics Application

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a complete data engineering solution that demonstrates real-world ETL (Extract, Transform, Load) patterns using the Harvard Art Museums API. It extracts artifact metadata, media files, and color information, loads them into a relational SQL database, performs analytical queries, and visualizes results through an interactive Streamlit dashboard.

**Architecture Flow**: API → ETL → SQL Database → Analytics Queries → Streamlit Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit app
streamlit run app.py
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

Register for a free API key at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

```python
# Use environment variables for secure configuration
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = 'https://api.harvardartmuseums.org'
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

conn = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three main tables with relational design:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    century VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    description TEXT,
    provenance TEXT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    renditionnumber VARCHAR(100),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    css3 VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## Core ETL Patterns

### Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = f'https://api.harvardartmuseums.org/object'
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage with pagination
artifacts = []
for page in range(1, 6):  # Fetch 5 pages
    records, info = fetch_artifacts(API_KEY, size=100, page=page)
    artifacts.extend(records)
    print(f"Fetched page {page}, Total: {info['totalrecords']}")
```

### Transform: Process Nested JSON

```python
def transform_artifact_metadata(artifacts):
    """
    Transform artifact JSON into flat DataFrame for metadata
    """
    transformed = []
    
    for artifact in artifacts:
        record = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'century': artifact.get('century'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        }
        transformed.append(record)
    
    return pd.DataFrame(transformed)

def transform_artifact_media(artifacts):
    """
    Extract media information from nested structure
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_record = {
                'artifact_id': artifact_id,
                'baseimageurl': img.get('baseimageurl'),
                'renditionnumber': img.get('renditionnumber'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            }
            media_list.append(media_record)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color palette data
    """
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent'),
                'css3': color.get('css3')
            }
            colors_list.append(color_record)
    
    return pd.DataFrame(colors_list)
```

### Load: Batch Insert into SQL

```python
def load_to_sql(df, table_name, conn):
    """
    Batch insert DataFrame into SQL table
    """
    cursor = conn.cursor()
    
    # Generate INSERT query dynamically
    cols = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Batch insert
    data = [tuple(row) for row in df.values]
    cursor.executemany(query, data)
    conn.commit()
    
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
    cursor.close()

# Complete ETL execution
artifacts, info = fetch_artifacts(API_KEY, size=100)

metadata_df = transform_artifact_metadata(artifacts)
media_df = transform_artifact_media(artifacts)
colors_df = transform_artifact_colors(artifacts)

conn = mysql.connector.connect(**db_config)
load_to_sql(metadata_df, 'artifactmetadata', conn)
load_to_sql(media_df, 'artifactmedia', conn)
load_to_sql(colors_df, 'artifactcolors', conn)
conn.close()
```

## Analytics Queries

### Common SQL Analytics Patterns

```python
# Query 1: Artifacts by culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Century distribution
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Color analysis
query_colors = """
SELECT ac.color, COUNT(DISTINCT ac.artifact_id) as artifacts,
       ROUND(AVG(ac.percent), 2) as avg_percent
FROM artifactcolors ac
GROUP BY ac.color
HAVING COUNT(DISTINCT ac.artifact_id) >= 5
ORDER BY artifacts DESC
LIMIT 15;
"""

# Query 4: Media availability
query_media = """
SELECT am.classification,
       COUNT(DISTINCT am.artifact_id) as total_artifacts,
       COUNT(DISTINCT amedia.artifact_id) as with_media,
       ROUND(COUNT(DISTINCT amedia.artifact_id) * 100.0 / COUNT(DISTINCT am.artifact_id), 2) as media_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia amedia ON am.artifact_id = amedia.artifact_id
GROUP BY am.classification
ORDER BY total_artifacts DESC
LIMIT 10;
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
with st.sidebar:
    st.header("Configuration")
    
    # Database connection test
    if st.button("Test DB Connection"):
        try:
            conn = mysql.connector.connect(**db_config)
            st.success("✅ Connected to database")
            conn.close()
        except Exception as e:
            st.error(f"❌ Connection failed: {e}")

# ETL Section
st.header("📥 ETL Pipeline")

col1, col2 = st.columns(2)

with col1:
    num_records = st.number_input("Records to fetch", 10, 1000, 100)
    
with col2:
    if st.button("Run ETL"):
        with st.spinner("Fetching data..."):
            artifacts, info = fetch_artifacts(API_KEY, size=num_records)
            
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            
            conn = mysql.connector.connect(**db_config)
            load_to_sql(metadata_df, 'artifactmetadata', conn)
            load_to_sql(media_df, 'artifactmedia', conn)
            load_to_sql(colors_df, 'artifactcolors', conn)
            conn.close()
            
            st.success(f"✅ Loaded {len(metadata_df)} artifacts")

# Analytics Section
st.header("📊 SQL Analytics")

queries = {
    "Artifacts by Culture": query_culture,
    "Century Distribution": query_century,
    "Color Analysis": query_colors,
    "Media Availability": query_media
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    conn = mysql.connector.connect(**db_config)
    df_result = pd.read_sql(queries[selected_query], conn)
    conn.close()
    
    # Display results
    st.dataframe(df_result, use_container_width=True)
    
    # Auto-generate visualization
    if len(df_result) > 0:
        x_col = df_result.columns[0]
        y_col = df_result.columns[1]
        
        fig = px.bar(df_result, x=x_col, y=y_col, 
                     title=selected_query,
                     color=y_col,
                     color_continuous_scale='Viridis')
        
        st.plotly_chart(fig, use_container_width=True)
```

## Advanced Patterns

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(api_key, total_records=1000, rate_limit=0.5):
    """
    Fetch data with rate limiting to respect API limits
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < total_records:
        try:
            records, info = fetch_artifacts(api_key, size=size, page=page)
            artifacts.extend(records)
            
            print(f"Fetched {len(records)} records (Total: {len(artifacts)})")
            
            if len(records) < size:
                break
                
            page += 1
            time.sleep(rate_limit)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return artifacts
```

### Incremental ETL Pattern

```python
def get_last_artifact_id(conn):
    """
    Get the last loaded artifact ID for incremental loading
    """
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl(api_key, conn):
    """
    Only load new artifacts since last run
    """
    last_id = get_last_artifact_id(conn)
    
    # Fetch only new artifacts
    url = f'https://api.harvardartmuseums.org/object'
    params = {
        'apikey': api_key,
        'size': 100,
        'after': last_id  # Only fetch IDs greater than last_id
    }
    
    response = requests.get(url, params=params)
    new_artifacts = response.json()['records']
    
    if new_artifacts:
        metadata_df = transform_artifact_metadata(new_artifacts)
        load_to_sql(metadata_df, 'artifactmetadata', conn)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### Common Issues

**API Rate Limiting**
```python
# Use exponential backoff
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            if response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
                continue
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

**Database Connection Issues**
```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="artifact_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return db_pool.get_connection()
```

**Memory Issues with Large Datasets**
```python
# Process in chunks
def chunked_load(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df = transform_artifact_metadata(chunk)
        load_to_sql(df, 'artifactmetadata', conn)
        print(f"Processed chunk {i//chunk_size + 1}")
```

## Environment Variables Template

Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_db_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

This project demonstrates production-ready ETL practices including error handling, rate limiting, incremental loading, and interactive analytics visualization.
