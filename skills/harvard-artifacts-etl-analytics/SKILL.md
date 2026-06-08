---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - connect to Harvard Art Museums API with Python
  - set up artifact data warehouse with TiDB
  - query and visualize museum collection data
  - implement pagination for Harvard API requests
  - design SQL schema for artifact metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading it into SQL databases (MySQL/TiDB Cloud), and visualizing analytics through Streamlit dashboards.

## What It Does

- **API Data Collection**: Fetches artifact metadata, media, and color data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper foreign key relationships
- **Analytics Engine**: Executes 20+ predefined analytical queries
- **Interactive Visualization**: Generates Plotly charts from SQL query results via Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests pymysql plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Obtain Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to `.env` file

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;

-- Create artifactmetadata table
CREATE TABLE artifactmetadata (
    object_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    url VARCHAR(500)
);

-- Create artifactmedia table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    base_image_url VARCHAR(500),
    alt_text TEXT,
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);

-- Create artifactcolors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    color VARCHAR(100),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);
```

## Core Functionality

### API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages with pagination
def collect_artifacts(num_pages=5):
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        artifacts, info = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(artifacts)
        print(f"Fetched page {page}/{num_pages} - Total: {info['totalrecords']}")
    
    return all_artifacts
```

### ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform API response to metadata DataFrame"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'object_id': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'dimensions': artifact.get('dimensions'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image data"""
    media_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        primary_image = artifact.get('primaryimageurl')
        images = artifact.get('images', [])
        
        if primary_image:
            media_list.append({
                'object_id': object_id,
                'base_image_url': primary_image,
                'alt_text': artifact.get('title')
            })
        
        for img in images:
            media_list.append({
                'object_id': object_id,
                'base_image_url': img.get('baseimageurl'),
                'alt_text': img.get('alttext')
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color data"""
    color_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'object_id': object_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_list)
```

### Database Loading

```python
import pymysql
from sqlalchemy import create_engine

def create_db_connection():
    """Create database connection"""
    connection_string = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    engine = create_engine(connection_string)
    return engine

def load_to_database(df_metadata, df_media, df_colors):
    """Load DataFrames to SQL database"""
    engine = create_db_connection()
    
    # Load metadata (replace if exists)
    df_metadata.to_sql('artifactmetadata', engine, if_exists='replace', index=False)
    print(f"Loaded {len(df_metadata)} rows to artifactmetadata")
    
    # Load media
    df_media.to_sql('artifactmedia', engine, if_exists='replace', index=False)
    print(f"Loaded {len(df_media)} rows to artifactmedia")
    
    # Load colors
    df_colors.to_sql('artifactcolors', engine, if_exists='replace', index=False)
    print(f"Loaded {len(df_colors)} rows to artifactcolors")
    
    engine.dispose()

# Complete ETL pipeline
artifacts = collect_artifacts(num_pages=5)
df_metadata = transform_artifact_metadata(artifacts)
df_media = transform_artifact_media(artifacts)
df_colors = transform_artifact_colors(artifacts)
load_to_database(df_metadata, df_media, df_colors)
```

## Analytical SQL Queries

```python
# Sample analytical queries
QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top 10 Cultures": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Media": """
        SELECT am.department, 
               COUNT(DISTINCT am.object_id) as artifacts_with_images
        FROM artifactmetadata am
        JOIN artifactmedia media ON am.object_id = media.object_id
        GROUP BY am.department
    """,
    
    "Century Distribution": """
        SELECT century, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century, classification
        ORDER BY count DESC
    """
}

def execute_query(query_name):
    """Execute SQL query and return results"""
    engine = create_db_connection()
    query = QUERIES[query_name]
    
    df_result = pd.read_sql(query, engine)
    engine.dispose()
    
    return df_result
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.title("Harvard Artifacts Analytics Dashboard")

# Sidebar for query selection
query_name = st.sidebar.selectbox(
    "Select Analytics Query",
    list(QUERIES.keys())
)

if st.button("Run Query"):
    with st.spinner("Executing query..."):
        df_result = execute_query(query_name)
        
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(
                df_result, 
                x=df_result.columns[0], 
                y=df_result.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

# Data collection interface
st.sidebar.subheader("ETL Pipeline")
num_pages = st.sidebar.number_input("Pages to fetch", min_value=1, max_value=50, value=5)

if st.sidebar.button("Run ETL"):
    with st.spinner("Running ETL pipeline..."):
        artifacts = collect_artifacts(num_pages=num_pages)
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
        load_to_database(df_metadata, df_media, df_colors)
        st.success(f"ETL completed! Loaded {len(df_metadata)} artifacts")
```

### Run the App

```bash
streamlit run app.py
```

## Common Patterns

### Incremental Data Loading

```python
def incremental_load(new_artifacts):
    """Load only new artifacts not already in database"""
    engine = create_db_connection()
    
    # Get existing object IDs
    existing_ids = pd.read_sql("SELECT object_id FROM artifactmetadata", engine)
    existing_set = set(existing_ids['object_id'])
    
    # Filter new artifacts
    new_only = [a for a in new_artifacts if a.get('objectid') not in existing_set]
    
    if new_only:
        df_metadata = transform_artifact_metadata(new_only)
        df_metadata.to_sql('artifactmetadata', engine, if_exists='append', index=False)
        print(f"Added {len(new_only)} new artifacts")
    
    engine.dispose()
```

### Error Handling and Retry Logic

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
import time
time.sleep(0.5)  # 500ms between requests
```

**Database Connection Errors**: Verify credentials and network access
```python
# Test connection
engine = create_db_connection()
with engine.connect() as conn:
    result = conn.execute("SELECT 1")
    print("Database connected successfully")
```

**Missing Data Fields**: Handle None values in transformations
```python
metadata = {
    'object_id': artifact.get('objectid'),
    'title': artifact.get('title') or 'Untitled',
    'culture': artifact.get('culture') or 'Unknown'
}
```

**Large Dataset Memory Issues**: Use batch processing
```python
def batch_load(artifacts, batch_size=1000):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        # Process and load batch
```
