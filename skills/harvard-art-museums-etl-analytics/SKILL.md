---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - set up Harvard artifacts data engineering project
  - create analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums data
  - build Streamlit app with museum API data
  - query and visualize Harvard artifacts with SQL
  - implement ETL pipeline for art collection data
  - analyze museum artifacts with Python and SQL
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases (MySQL/TiDB), and visualizing insights through a Streamlit dashboard. It showcases real-world ETL patterns, pagination handling, batch processing, and analytical SQL queries.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit application
streamlit run app.py
```

## Configuration

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store it as an environment variable:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or configure within the Streamlit app's sidebar (not recommended for production).

### Database Configuration

The project supports MySQL and TiDB Cloud. Set up your database connection:

```python
import os
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

The ETL pipeline creates three relational tables:

```sql
-- Artifact metadata
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url VARCHAR(500)
);

-- Artifact media/images
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    media_id VARCHAR(100),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Color analysis
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core ETL Patterns

### Extract: API Data Collection

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, records_per_page=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': records_per_page,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_records.extend(records)
            page += 1
            
            # Rate limiting
            time.sleep(0.5)
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_records]
```

### Transform: JSON to Relational Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media (one-to-many)
        for media in artifact.get('images', []):
            media_list.append({
                'objectid': artifact.get('objectid'),
                'media_id': media.get('imageid'),
                'baseimageurl': media.get('baseimageurl'),
                'iiifbaseuri': media.get('iiifbaseuri'),
                'publiccaption': media.get('publiccaption')
            })
        
        # Extract colors (one-to-many)
        for color in artifact.get('colors', []):
            colors_list.append({
                'objectid': artifact.get('objectid'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return {
        'metadata': pd.DataFrame(metadata_list),
        'media': pd.DataFrame(media_list),
        'colors': pd.DataFrame(colors_list)
    }
```

### Load: Batch Insert to SQL

```python
from mysql.connector import Error

def load_to_database(dataframes, db_config):
    """
    Batch insert dataframes into SQL database
    """
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        
        # Insert metadata
        metadata_df = dataframes['metadata']
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (objectid, title, culture, century, dated, classification, 
                 department, division, technique, medium, dimensions, 
                 creditline, accession_number, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        media_df = dataframes['media']
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (objectid, media_id, baseimageurl, iiifbaseuri, publiccaption)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        colors_df = dataframes['colors']
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (objectid, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        if conn.is_connected():
            cursor.close()
            conn.close()
```

## Analytical SQL Queries

### Example Queries

```python
# Top 10 cultures by artifact count
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts with images
query_media = """
SELECT 
    COUNT(DISTINCT m.objectid) as artifacts_with_images,
    COUNT(*) as total_images
FROM artifactmedia m
"""

# Color distribution
query_colors = """
SELECT 
    color, 
    COUNT(*) as occurrences,
    AVG(percent) as avg_percentage
FROM artifactcolors
GROUP BY color
ORDER BY occurrences DESC
LIMIT 15
"""

# Department analysis
query_department = """
SELECT 
    department,
    COUNT(*) as total_artifacts,
    COUNT(DISTINCT classification) as unique_classifications
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC
"""
```

## Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px

def run_analytics_dashboard():
    st.title("Harvard Art Museums Analytics")
    
    # Query selector
    queries = {
        "Artifacts by Culture": query_culture,
        "Media Analysis": query_media,
        "Color Distribution": query_colors,
        "Department Breakdown": query_department
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        # Execute query
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        # Display results
        st.dataframe(df)
        
        # Visualize
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig)
```

## Complete ETL Workflow

```python
def run_complete_etl():
    """
    End-to-end ETL pipeline execution
    """
    # 1. Extract
    api_key = os.getenv('HARVARD_API_KEY')
    raw_data = fetch_artifacts(api_key, num_records=500)
    
    # 2. Transform
    dataframes = transform_artifacts(raw_data)
    
    # 3. Load
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    load_to_database(dataframes, db_config)
    
    print("ETL pipeline completed successfully!")
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from functools import wraps

def rate_limit(delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            time.sleep(delay)
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limit(delay=0.5)
def fetch_single_page(url, params):
    return requests.get(url, params=params)
```

### Database Connection Issues

```python
def get_db_connection(max_retries=3):
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(**db_config)
            return conn
        except Error as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
            else:
                raise e
```

### Handling Missing Data

```python
def safe_extract(data, key, default=None):
    """Safely extract nested data with fallback"""
    try:
        return data.get(key, default)
    except (AttributeError, KeyError):
        return default
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement pagination** for large datasets
3. **Add rate limiting** to respect API quotas
4. **Use batch inserts** for database performance
5. **Handle NULL values** explicitly in SQL queries
6. **Create indexes** on foreign keys for query optimization
7. **Log ETL progress** for monitoring and debugging
