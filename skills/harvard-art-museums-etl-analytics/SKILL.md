---
name: harvard-art-museums-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering app with Harvard Art Museums API
  - set up artifact collection analytics with SQL and Streamlit
  - integrate Harvard Art Museums API into a data pipeline
  - build a museum data analytics dashboard
  - create an end-to-end data engineering project with art museum data
  - analyze Harvard Art Museums collection with Python and SQL
  - build a Streamlit app for artifact data visualization
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Connects to Harvard Art Museums API to fetch artifact data
- **ETL Pipeline**: Extracts artifact metadata, transforms nested JSON, loads into relational SQL tables
- **SQL Analytics**: Pre-built analytical queries for insights on artifacts, cultures, media, and colors
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time analytics
- **Database Design**: Normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

## Installation

### Prerequisites

```bash
# Required Python packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file with required credentials:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration (MySQL or TiDB Cloud)
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema Setup

```sql
-- Create artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(255),
    provenance TEXT,
    dimensions VARCHAR(500),
    copyright TEXT,
    creditline TEXT,
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Create artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    primaryimageurl VARCHAR(500),
    has_images BOOLEAN,
    images_count INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
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
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
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
        return data['records'], data['info']['pages']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, total_pages = fetch_artifacts(api_key, page=1)
print(f"Fetched {len(artifacts)} artifacts from page 1 of {total_pages}")
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform artifact records into metadata DataFrame
    """
    metadata = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'provenance': artifact.get('provenance'),
            'dimensions': artifact.get('dimensions'),
            'copyright': artifact.get('copyright'),
            'creditline': artifact.get('creditline'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """
    Extract media information from artifacts
    """
    media = []
    
    for artifact in artifacts:
        record = {
            'artifact_id': artifact.get('id'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'has_images': 1 if artifact.get('images') else 0,
            'images_count': len(artifact.get('images', []))
        }
        media.append(record)
    
    return pd.DataFrame(media)

def transform_artifact_colors(artifacts):
    """
    Extract color data from artifacts
    """
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color_obj in color_data:
            record = {
                'artifact_id': artifact_id,
                'color': color_obj.get('color'),
                'spectrum': color_obj.get('spectrum'),
                'hue': color_obj.get('hue'),
                'percent': color_obj.get('percent')
            }
            colors.append(record)
    
    return pd.DataFrame(colors)
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection using environment variables
    """
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
    """
    Bulk insert artifact metadata with conflict handling
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, 
         medium, technique, period, provenance, dimensions, copyright, 
         creditline, url, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    print(f"Loaded {cursor.rowcount} metadata records")
    cursor.close()
    conn.close()

def load_media(df_media):
    """
    Insert artifact media data
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, primaryimageurl, has_images, images_count)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    print(f"Loaded {cursor.rowcount} media records")
    cursor.close()
    conn.close()

def load_colors(df_colors):
    """
    Insert artifact color data
    """
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
    
    print(f"Loaded {cursor.rowcount} color records")
    cursor.close()
    conn.close()
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, num_pages=5):
    """
    Execute full ETL pipeline for specified number of pages
    """
    all_artifacts = []
    
    # Extract
    print("Starting extraction...")
    for page in range(1, num_pages + 1):
        artifacts, total_pages = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(artifacts)
        print(f"Extracted page {page}/{num_pages}")
    
    # Transform
    print("Starting transformation...")
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    
    # Load
    print("Starting load...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL pipeline completed successfully!")
    return len(all_artifacts)

# Execute pipeline
api_key = os.getenv('HARVARD_API_KEY')
total_artifacts = run_etl_pipeline(api_key, num_pages=10)
print(f"Total artifacts processed: {total_artifacts}")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Artifacts by century distribution
query_2 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Media availability analysis
query_3 = """
SELECT 
    has_images,
    COUNT(*) as artifact_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) as percentage
FROM artifactmedia
GROUP BY has_images;
"""

# Top departments by artifact count
query_4 = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY count DESC;
"""

# Color spectrum distribution
query_5 = """
SELECT 
    spectrum,
    COUNT(*) as color_count,
    ROUND(AVG(percent), 2) as avg_percent
FROM artifactcolors
WHERE spectrum IS NOT NULL
GROUP BY spectrum
ORDER BY color_count DESC;
"""

# Most viewed artifacts
query_6 = """
SELECT title, culture, century, totalpageviews
FROM artifactmetadata
WHERE totalpageviews > 0
ORDER BY totalpageviews DESC
LIMIT 20;
"""
```

### Execute Queries

```python
def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Usage
df_cultures = execute_query(query_1)
print(df_cultures)
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    
    queries = {
        "Top 10 Cultures": query_1,
        "Century Distribution": query_2,
        "Media Availability": query_3,
        "Top Departments": query_4,
        "Color Spectrum": query_5,
        "Most Viewed Artifacts": query_6
    }
    
    selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))
    
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df_result = execute_query(queries[selected_query])
            
            st.subheader(f"Results: {selected_query}")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

Run the dashboard:

```bash
streamlit run app.py
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time

def fetch_all_artifacts_with_rate_limit(api_key, max_pages=50, delay=0.5):
    """
    Fetch multiple pages with rate limiting to respect API limits
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            artifacts, total_pages = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(artifacts)
            print(f"Page {page}/{min(max_pages, total_pages)}")
            
            # Rate limit: wait between requests
            time.sleep(delay)
            
            if page >= total_pages:
                break
                
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Incremental ETL Updates

```python
def get_max_artifact_id():
    """
    Get the highest artifact ID already in database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return max_id or 0

def incremental_etl(api_key):
    """
    Only load new artifacts not already in database
    """
    max_id = get_max_artifact_id()
    print(f"Latest artifact ID in DB: {max_id}")
    
    # Fetch new data
    new_artifacts = []
    page = 1
    
    while True:
        artifacts, _ = fetch_artifacts(api_key, page=page)
        new_batch = [a for a in artifacts if a['id'] > max_id]
        
        if not new_batch:
            break
            
        new_artifacts.extend(new_batch)
        page += 1
    
    if new_artifacts:
        print(f"Found {len(new_artifacts)} new artifacts")
        # Transform and load
        df_metadata = transform_artifact_metadata(new_artifacts)
        load_metadata(df_metadata)
    else:
        print("No new artifacts found")
```

## Troubleshooting

### API Key Issues

```python
# Test API connection
def test_api_key(api_key):
    """
    Validate Harvard Art Museums API key
    """
    test_url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
    response = requests.get(test_url)
    
    if response.status_code == 200:
        print("✅ API key is valid")
        return True
    elif response.status_code == 401:
        print("❌ API key is invalid")
        return False
    else:
        print(f"⚠️ Unexpected response: {response.status_code}")
        return False
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection():
    """
    Verify database connectivity and credentials
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        
        if result:
            print("✅ Database connection successful")
            cursor.close()
            conn.close()
            return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=None):
    """
    Safely extract values from nested API responses
    """
    try:
        value = dictionary.get(key, default)
        return value if value else default
    except:
        return default

# Use in transformation
record = {
    'id': safe_get(artifact, 'id', 0),
    'title': safe_get(artifact, 'title', 'Unknown'),
    'culture': safe_get(artifact, 'culture', 'Not Specified')
}
```

### Memory Management for Large Datasets

```python
def batch_etl(api_key, batch_size=10):
    """
    Process data in batches to manage memory
    """
    page = 1
    
    while True:
        artifacts, total_pages = fetch_artifacts(api_key, page=page, size=100)
        
        if not artifacts:
            break
        
        # Process batch immediately
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata(df_metadata)
        
        print(f"Processed batch {page}")
        page += 1
        
        if page > batch_size:
            break
```

This skill enables agents to build production-ready ETL pipelines with the Harvard Art Museums API, implement robust data engineering practices, and create interactive analytics dashboards.
