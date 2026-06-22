---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline with Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data to SQL
  - analyze art museum collections with Python
  - build Streamlit app for artifact visualization
  - set up data pipeline for Harvard Art Museums
  - query and visualize museum artifact metadata
  - create relational database from Harvard API
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

This project is an end-to-end data engineering application that demonstrates professional ETL (Extract, Transform, Load) patterns using the Harvard Art Museums API. It extracts artifact data, transforms nested JSON into relational tables, loads into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through a Streamlit dashboard with Plotly visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Getting Harvard API Key

1. Visit: https://www.harvardartmuseums.org/collections/api
2. Request API access
3. Add key to `.env` file

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

# Create tables
cursor = conn.cursor()

# Artifact Metadata table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    division VARCHAR(255),
    medium VARCHAR(500),
    period VARCHAR(255),
    technique VARCHAR(500),
    url TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
)
""")

# Artifact Media table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    renditionnumber VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
)
""")

# Artifact Colors table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    css3 VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    spectrum VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
)
""")

conn.commit()
cursor.close()
conn.close()
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100, page_size=100):
    """
    Extract artifact data from Harvard Art Museums API
    Handles pagination automatically
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    pages_needed = (num_records + page_size - 1) // page_size
    
    for page in range(1, pages_needed + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            if len(all_artifacts) >= num_records:
                break
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts[:num_records]
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """
    Transform nested JSON into three relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        object_id = artifact.get('objectid')
        
        # Extract metadata
        metadata = {
            'objectid': object_id,
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'department': artifact.get('department', ''),
            'classification': artifact.get('classification', ''),
            'dated': artifact.get('dated', ''),
            'division': artifact.get('division', ''),
            'medium': artifact.get('medium', ''),
            'period': artifact.get('period', ''),
            'technique': artifact.get('technique', ''),
            'url': artifact.get('url', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        images = artifact.get('images', [])
        for img in images:
            media = {
                'objectid': object_id,
                'baseimageurl': img.get('baseimageurl', ''),
                'iiifbaseuri': img.get('iiifbaseuri', ''),
                'renditionnumber': img.get('renditionnumber', ''),
                'height': img.get('height', 0),
                'width': img.get('width', 0)
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'objectid': object_id,
                'color': color.get('color', ''),
                'css3': color.get('css3', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0),
                'spectrum': color.get('spectrum', '')
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list) if media_list else pd.DataFrame()
    df_colors = pd.DataFrame(colors_list) if colors_list else pd.DataFrame()
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into SQL Database

```python
def load_to_database(df_metadata, df_media, df_colors):
    """
    Load transformed dataframes into SQL database
    Uses batch inserts for performance
    """
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, department, classification, 
     dated, division, medium, period, technique, url, 
     totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    for _, row in df_metadata.iterrows():
        cursor.execute(metadata_query, tuple(row))
    
    # Insert media
    if not df_media.empty:
        media_query = """
        INSERT INTO artifactmedia 
        (objectid, baseimageurl, iiifbaseuri, renditionnumber, height, width)
        VALUES (%s, %s, %s, %s, %s, %s)
        """
        for _, row in df_media.iterrows():
            cursor.execute(media_query, tuple(row))
    
    # Insert colors
    if not df_colors.empty:
        colors_query = """
        INSERT INTO artifactcolors 
        (objectid, color, css3, hue, percent, spectrum)
        VALUES (%s, %s, %s, %s, %s, %s)
        """
        for _, row in df_colors.iterrows():
            cursor.execute(colors_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return len(df_metadata), len(df_media), len(df_colors)
```

## Analytics Queries

### Example SQL Analytics

```python
def run_analytics_query(query_name):
    """
    Execute predefined analytical queries
    """
    queries = {
        'top_cultures': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'century_distribution': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'department_stats': """
            SELECT department, 
                   COUNT(*) as total_artifacts,
                   AVG(totalpageviews) as avg_pageviews
            FROM artifactmetadata
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        
        'color_analysis': """
            SELECT c.color, c.spectrum, 
                   COUNT(*) as frequency,
                   AVG(c.percent) as avg_percentage
            FROM artifactcolors c
            GROUP BY c.color, c.spectrum
            ORDER BY frequency DESC
            LIMIT 20
        """,
        
        'media_availability': """
            SELECT 
                COUNT(DISTINCT m.objectid) as artifacts_with_images,
                COUNT(*) as total_images,
                AVG(m.width) as avg_width,
                AVG(m.height) as avg_height
            FROM artifactmedia m
        """
    }
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "Analytics Dashboard", "Database Stats"]
    )
    
    if menu == "ETL Pipeline":
        show_etl_page()
    elif menu == "Analytics Dashboard":
        show_analytics_page()
    elif menu == "Database Stats":
        show_stats_page()

def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, 
                                   value=100, step=10)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = fetch_artifacts(num_records)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            counts = load_to_database(df_meta, df_media, df_colors)
            st.success(f"Loaded: {counts[0]} metadata, {counts[1]} media, {counts[2]} colors")

def show_analytics_page():
    st.header("📊 Analytics Dashboard")
    
    query_type = st.selectbox(
        "Select Analysis",
        ["Top Cultures", "Century Distribution", "Department Stats", 
         "Color Analysis", "Media Availability"]
    )
    
    query_map = {
        "Top Cultures": "top_cultures",
        "Century Distribution": "century_distribution",
        "Department Stats": "department_stats",
        "Color Analysis": "color_analysis",
        "Media Availability": "media_availability"
    }
    
    if st.button("Run Analysis"):
        df = run_analytics_query(query_map[query_type])
        
        # Display table
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_type)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Run the Application

```bash
streamlit run app.py
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(url, params, delay=0.5):
    """Add delay between API calls to respect rate limits"""
    response = requests.get(url, params=params)
    time.sleep(delay)
    return response
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=''):
    """Safely extract values with defaults"""
    value = dictionary.get(key, default)
    return value if value is not None else default
```

### Batch Processing

```python
def batch_insert(cursor, query, data, batch_size=1000):
    """Insert data in batches for better performance"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i+batch_size]
        cursor.executemany(query, batch)
```

## Troubleshooting

### API Key Issues
- Verify `HARVARD_API_KEY` is set in `.env`
- Check API key is valid at Harvard portal
- Ensure no rate limit exceeded (429 errors)

### Database Connection Errors
- Verify all `DB_*` environment variables are set
- Check database server is running
- Confirm firewall allows connection on DB_PORT
- Test connection with `mysql-connector-python`

### Data Loading Failures
- Check foreign key constraints are satisfied
- Verify data types match table schema
- Use `ON DUPLICATE KEY UPDATE` for re-runs
- Handle NULL values appropriately

### Streamlit Performance
- Limit query result sizes with SQL LIMIT
- Use `@st.cache_data` for expensive operations
- Optimize database indexes on frequently queried columns

### Memory Issues with Large Datasets
```python
# Process in chunks
chunk_size = 100
for i in range(0, total_records, chunk_size):
    batch = fetch_artifacts(chunk_size)
    process_and_load(batch)
```
