---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and load Harvard API data to SQL
  - visualize museum collection analytics with Streamlit
  - implement artifact data engineering pipeline
  - query and analyze Harvard Art Museums database
  - set up museum collection data warehouse
  - process Harvard API artifacts into relational database
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for the Harvard Art Museums API. It demonstrates:

- **ETL Pipeline**: Extract artifact data from Harvard API, transform JSON to relational format, load into SQL database
- **Database Design**: Three-table schema (metadata, media, colors) with foreign key relationships
- **Analytics**: 20+ predefined SQL queries for artifact insights
- **Visualization**: Streamlit dashboard with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### Environment Variables

Set up your credentials:

```bash
# Harvard Art Museums API
export HARVARD_API_KEY="your_api_key_here"

# MySQL/TiDB Database
export DB_HOST="your_db_host"
export DB_PORT="3306"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
```

### Database Setup

Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    description TEXT,
    provenance TEXT,
    accessionyear INT
);

-- Media/images table
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Colors table
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def extract_all_artifacts(api_key, max_pages=10):
    """Extract multiple pages with pagination"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
        
        # Rate limiting
        import time
        time.sleep(1)
    
    return all_artifacts
```

### Data Transformation

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifacts to metadata DataFrame"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'accessionyear': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Transform media/images data"""
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'objectid': objectid,
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Transform color data"""
    color_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_list)
```

### Data Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """Load metadata to database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, classification, department, 
     division, technique, dated, description, provenance, accessionyear)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    records = df_metadata.to_records(index=False)
    cursor.executemany(insert_query, records.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(df_metadata)} metadata records")

def load_media(df_media):
    """Load media data to database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (objectid, baseimageurl, format, height, width)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df_media.to_records(index=False)
    cursor.executemany(insert_query, records.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(df_media)} media records")

def load_colors(df_colors):
    """Load color data to database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df_colors.to_records(index=False)
    cursor.executemany(insert_query, records.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
    print(f"Loaded {len(df_colors)} color records")
```

## Complete ETL Execution

```python
def run_etl_pipeline():
    """Execute complete ETL pipeline"""
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Extract
    print("Starting extraction...")
    artifacts = extract_all_artifacts(api_key, max_pages=5)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_metadata(artifacts)
    df_media = transform_media(artifacts)
    df_colors = transform_colors(artifacts)
    
    # Load
    print("Loading to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline()
```

## Analytics Queries

### Sample SQL Analytics

```python
ANALYTICS_QUERIES = {
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
    
    "top_departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "artifacts_with_most_images": """
        SELECT m.objectid, m.title, COUNT(a.mediaid) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.objectid = a.objectid
        GROUP BY m.objectid, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "classification_by_department": """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND classification IS NOT NULL
        GROUP BY department, classification
        ORDER BY department, count DESC
    """
}

def execute_query(query_name):
    """Execute analytics query and return results"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return pd.DataFrame(results)
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics")
    st.markdown("Data Engineering & Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_option = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(query_option)
            
            st.subheader(f"Results: {query_option}")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=f"{query_option.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig)
    
    # ETL Controls
    st.sidebar.header("ETL Pipeline")
    if st.sidebar.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline()
            st.success("ETL completed successfully!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline separately
python etl_pipeline.py

# Access dashboard at http://localhost:8501
```

## Common Patterns

### Incremental Loading

```python
def get_max_objectid():
    """Get highest objectid in database for incremental loads"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Load only new artifacts"""
    max_id = get_max_objectid()
    # Fetch artifacts with objectid > max_id
    # Continue ETL process
```

### Error Handling

```python
def safe_load(df, load_function):
    """Load with error handling"""
    try:
        load_function(df)
    except Error as e:
        print(f"Database error: {e}")
        # Implement retry logic or logging
    except Exception as e:
        print(f"Unexpected error: {e}")
```

## Troubleshooting

**API Rate Limits**: Add `time.sleep(1)` between requests or use exponential backoff.

**Database Connection Issues**: Verify environment variables and network connectivity.

**Memory Issues with Large Datasets**: Process in batches using pagination and chunked loading.

**Missing Data**: Handle None values in transformations with `.get()` and default values.

**Duplicate Keys**: Use `ON DUPLICATE KEY UPDATE` or check existing records before insert.
