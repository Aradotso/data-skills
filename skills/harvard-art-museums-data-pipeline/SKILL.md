---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - create a data pipeline for Harvard Art Museums API
  - build an ETL pipeline with Streamlit visualization
  - analyze Harvard artifacts collection data
  - set up SQL analytics for museum artifact data
  - extract and visualize art museum metadata
  - build a museum data engineering application
  - create interactive dashboards for artifact collections
  - process Harvard Art Museums API with Python
---

# Harvard Art Museums Data Pipeline & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics application that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database structures
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries
- Visualizes insights using Streamlit and Plotly

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from: https://www.harvardartmuseums.org/collections/api

Set it as an environment variable:
```bash
export HARVARD_API_KEY="your_api_key_here"
```

### 2. Database Configuration

Set up MySQL/TiDB Cloud credentials:
```bash
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit app will launch at `http://localhost:8501`

## Database Schema

### Core Tables

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    technique VARCHAR(500),
    url TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    alttext TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Extract artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
records = data.get('records', [])
```

### Transform: Processing JSON to Relational Format

```python
import pandas as pd

def transform_artifact_metadata(records):
    """Transform artifact records into metadata DataFrame"""
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'department': record.get('department'),
            'classification': record.get('classification'),
            'dated': record.get('dated'),
            'technique': record.get('technique'),
            'url': record.get('url'),
            'totalpageviews': record.get('totalpageviews', 0),
            'totaluniquepageviews': record.get('totaluniquepageviews', 0)
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(records):
    """Transform media data into relational format"""
    media = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for image in images:
            media.append({
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl'),
                'alttext': image.get('alttext')
            })
    
    return pd.DataFrame(media)

def transform_artifact_colors(records):
    """Transform color data into relational format"""
    colors = []
    
    for record in records:
        artifact_id = record.get('id')
        color_data = record.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### Load: Inserting Data into MySQL

```python
import mysql.connector
import os

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df):
    """Load artifact metadata into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, department, classification, dated, technique, url, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    for _, row in df.iterrows():
        cursor.execute(insert_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()

def load_media(df):
    """Load artifact media into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, baseimageurl, alttext)
    VALUES (%s, %s, %s)
    """
    
    for _, row in df.iterrows():
        cursor.execute(insert_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Artifacts by Culture
query_culture = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Query 2: Artifacts by Century
query_century = """
SELECT century, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY artifact_count DESC
"""

# Query 3: Most Viewed Artifacts
query_pageviews = """
SELECT title, culture, totalpageviews
FROM artifactmetadata
ORDER BY totalpageviews DESC
LIMIT 20
"""

# Query 4: Color Distribution
query_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15
"""

# Query 5: Media Availability
query_media = """
SELECT 
    CASE WHEN media_count > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
    COUNT(*) as artifact_count
FROM (
    SELECT m.id, COUNT(md.media_id) as media_count
    FROM artifactmetadata m
    LEFT JOIN artifactmedia md ON m.id = md.artifact_id
    GROUP BY m.id
) as subquery
GROUP BY media_status
"""
```

### Executing Queries with Streamlit

```python
import streamlit as st
import plotly.express as px

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Streamlit app example
st.title("Harvard Artifacts Analytics Dashboard")

query_option = st.selectbox(
    "Select Analysis",
    ["Artifacts by Culture", "Artifacts by Century", "Color Distribution"]
)

if query_option == "Artifacts by Culture":
    df = execute_query(query_culture)
    st.dataframe(df)
    
    fig = px.bar(df, x='culture', y='count', title='Artifacts by Culture')
    st.plotly_chart(fig)
```

## Common Patterns

### Pattern 1: Paginated Data Collection

```python
def collect_all_artifacts(api_key, max_pages=10):
    """Collect multiple pages of artifacts"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page, size=100)
        records = data.get('records', [])
        all_records.extend(records)
        
        # Check if more pages available
        if len(records) < 100:
            break
    
    return all_records
```

### Pattern 2: Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, num_pages=5):
    """Run complete ETL pipeline"""
    # Extract
    st.info("Extracting data from API...")
    records = collect_all_artifacts(api_key, max_pages=num_pages)
    
    # Transform
    st.info("Transforming data...")
    metadata_df = transform_artifact_metadata(records)
    media_df = transform_artifact_media(records)
    colors_df = transform_artifact_colors(records)
    
    # Load
    st.info("Loading data to database...")
    load_metadata(metadata_df)
    load_media(media_df)
    load_colors(colors_df)
    
    st.success(f"ETL completed! Loaded {len(records)} artifacts")
```

### Pattern 3: Interactive Visualization Dashboard

```python
import plotly.graph_objects as go

def create_dashboard():
    """Create multi-chart analytics dashboard"""
    st.header("📊 Harvard Artifacts Analytics")
    
    col1, col2 = st.columns(2)
    
    with col1:
        df_culture = execute_query(query_culture)
        fig = px.pie(df_culture, values='count', names='culture', 
                     title='Distribution by Culture')
        st.plotly_chart(fig)
    
    with col2:
        df_century = execute_query(query_century)
        fig = px.bar(df_century, x='century', y='artifact_count',
                     title='Artifacts by Century')
        st.plotly_chart(fig)
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
            else:
                raise e
```

### Database Connection Issues
```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Exception as e:
        st.error(f"Database connection failed: {e}")
        return False
```

### Handling Missing Data
```python
def safe_get(record, key, default=None):
    """Safely extract nested values"""
    value = record.get(key, default)
    return value if value is not None else default

# Usage in transformation
metadata.append({
    'title': safe_get(record, 'title', 'Unknown'),
    'culture': safe_get(record, 'culture', 'Unknown')
})
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement batch inserts** for better database performance
3. **Handle pagination properly** to avoid missing data
4. **Add error handling** around API calls and database operations
5. **Use Streamlit caching** for expensive operations:

```python
@st.cache_data
def load_cached_query(query):
    return execute_query(query)
```
