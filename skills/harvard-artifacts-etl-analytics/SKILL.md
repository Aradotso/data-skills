---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data engineering pipeline with Harvard artifacts
  - build analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums data
  - set up SQL database for artifact metadata
  - visualize museum artifact data with Streamlit
  - query Harvard Art Museums API with pagination
  - analyze art collection data with SQL
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

An end-to-end data engineering and analytics application for the Harvard Art Museums API. This project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational SQL tables
- **SQL Analytics**: Runs 20+ predefined analytical queries on structured artifact data
- **Interactive Dashboards**: Visualizes query results using Streamlit and Plotly
- **Database Design**: Implements normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

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

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

```python
# Store in environment variable
export HARVARD_API_KEY="your_api_key_here"
```

### 2. Database Configuration

Set up MySQL/TiDB Cloud connection:

```python
# Environment variables for database
export DB_HOST="your_host"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
export DB_PORT="3306"
```

### 3. Streamlit Secrets (Alternative)

Create `.streamlit/secrets.toml`:

```toml
[api]
harvard_key = "your_api_key"

[database]
host = "your_host"
user = "your_username"
password = "your_password"
database = "harvard_artifacts"
port = 3306
```

## Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    primaryimageurl TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    artifact_id INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(20),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts with pagination from Harvard Art Museums API"""
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {info['totalrecords']}")
```

### Transform: Parse and Structure Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact JSON into metadata dataframe"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Transform nested media data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl'),
                'iiifbaseuri': image.get('iiifbaseuri'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Transform color data"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            port=int(os.getenv('DB_PORT', 3306))
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df_metadata):
    """Load metadata into database with batch insert"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     dated, url, primaryimageurl, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    cursor.close()
    connection.close()
    
    return len(data_tuples)

def load_media(df_media):
    """Load media data"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, iiifbaseuri, format, height, width)
    VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    cursor.close()
    connection.close()
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, num_pages=5):
    """Run complete ETL pipeline"""
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        artifacts, info = fetch_artifacts(api_key, page=page, size=100)
        all_artifacts.extend(artifacts)
    
    print(f"Extracted {len(all_artifacts)} artifacts")
    
    # Transform
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    
    print(f"Transformed into {len(df_metadata)} metadata records")
    
    # Load
    loaded_count = load_metadata(df_metadata)
    load_media(df_media)
    
    print(f"Loaded {loaded_count} records into database")
    
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'color_count': len(df_colors)
    }
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

# Main app structure
st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for API configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            results = run_etl_pipeline(api_key, num_pages=3)
            st.success(f"Loaded {results['metadata_count']} artifacts")

# Analytics queries
st.header("📊 SQL Analytics")

queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    "Top Viewed Artifacts": """
        SELECT title, culture, totalpageviews 
        FROM artifactmetadata 
        ORDER BY totalpageviews DESC 
        LIMIT 10
    """,
    "Color Distribution": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent 
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY count DESC 
        LIMIT 15
    """
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    connection = get_db_connection()
    df_result = pd.read_sql(queries[selected_query], connection)
    connection.close()
    
    # Display results
    st.dataframe(df_result, use_container_width=True)
    
    # Auto-generate visualization
    if len(df_result.columns) >= 2:
        fig = px.bar(df_result, 
                     x=df_result.columns[0], 
                     y=df_result.columns[1],
                     title=selected_query)
        st.plotly_chart(fig, use_container_width=True)
```

## Common Analytical Queries

```python
# Department distribution
"""
SELECT department, COUNT(*) as artifact_count 
FROM artifactmetadata 
GROUP BY department 
ORDER BY artifact_count DESC
"""

# Artifacts with most images
"""
SELECT a.title, a.culture, COUNT(m.artifact_id) as image_count
FROM artifactmetadata a
JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.id, a.title, a.culture
ORDER BY image_count DESC
LIMIT 10
"""

# Color prevalence by culture
"""
SELECT a.culture, c.color, AVG(c.percent) as avg_percent
FROM artifactmetadata a
JOIN artifactcolors c ON a.id = c.artifact_id
WHERE a.culture IS NOT NULL
GROUP BY a.culture, c.color
ORDER BY avg_percent DESC
"""

# Classification breakdown
"""
SELECT classification, century, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND century IS NOT NULL
GROUP BY classification, century
ORDER BY count DESC
"""
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_artifacts_with_retry(api_key, page=1, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            artifacts, info = fetch_artifacts(api_key, page)
            return artifacts, info
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
# Test connection
def test_db_connection():
    """Verify database connectivity"""
    try:
        conn = get_db_connection()
        if conn and conn.is_connected():
            cursor = conn.cursor()
            cursor.execute("SELECT 1")
            result = cursor.fetchone()
            cursor.close()
            conn.close()
            return True
    except Error as e:
        st.error(f"Database error: {e}")
        return False
```

### Handle Missing Data

```python
def safe_transform(artifact, field, default=None):
    """Safely extract nested fields"""
    return artifact.get(field, default)

# Apply in transforms
metadata = {
    'id': safe_transform(artifact, 'id'),
    'title': safe_transform(artifact, 'title', 'Unknown'),
    'culture': safe_transform(artifact, 'culture', 'Unknown')
}
```

## Best Practices

1. **Pagination**: Always handle API pagination for large datasets
2. **Batch Inserts**: Use `executemany()` for performance
3. **Error Handling**: Implement retry logic for API calls
4. **Environment Variables**: Never hardcode credentials
5. **Data Validation**: Check for null values before database insert
6. **Normalization**: Keep related data in separate tables with foreign keys

This skill enables AI agents to help developers build production-ready ETL pipelines for museum collection data with proper database design and analytics capabilities.
