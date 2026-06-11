---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and Harvard API
  - query Harvard museum collection database
  - visualize artifact data with Plotly
  - set up MySQL database for museum artifacts
  - transform Harvard API JSON to relational tables
  - analyze art collection data by culture and century
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into structured relational tables, loads it into MySQL/TiDB, and provides interactive analytics dashboards using Streamlit and Plotly.

**Architecture:** `API → ETL → SQL → Analytics → Visualization`

The application handles:
- Paginated API data collection with rate limiting
- Transformation of nested JSON to normalized relational schema
- Batch SQL inserts for performance
- 20+ predefined analytical queries
- Interactive visualizations with Plotly

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Requirements

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

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform
2. Register for free API access
3. Add key to `.env` file

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    period VARCHAR(255),
    technique VARCHAR(255),
    people TEXT,
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit app will open at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

# Fetch multiple pages
def fetch_all_artifacts(max_pages=10):
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        
        print(f"Fetched page {page}/{info['pages']}")
        
        if page >= info['pages']:
            break
    
    return all_artifacts
```

### 2. ETL Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data to metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'dated': artifact.get('dated', ''),
            'medium': artifact.get('medium', ''),
            'period': artifact.get('period', ''),
            'technique': artifact.get('technique', ''),
            'people': ','.join([p['name'] for p in artifact.get('people', [])]),
            'url': artifact.get('url', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Extract media information"""
    media = []
    
    for artifact in artifacts:
        media.append({
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'iiifbaseuri': artifact.get('iiifbaseuri', ''),
            'primaryimageurl': artifact.get('primaryimageurl', '')
        })
    
    return pd.DataFrame(media)

def transform_artifact_colors(artifacts):
    """Extract color information"""
    colors = []
    
    for artifact in artifacts:
        for color in artifact.get('colors', []):
            colors.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### 3. Load to Database

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

def load_metadata(df):
    """Batch insert metadata"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, dated, 
     medium, period, technique, people, url, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(df)} metadata records")

def load_media(df):
    """Batch insert media records"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()

def load_colors(df):
    """Batch insert color records"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Analytical Queries

```python
# Sample analytical queries

QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL AND culture != ''
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
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as count 
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE department IS NOT NULL 
        GROUP BY department 
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(*) as total_artifacts,
            SUM(CASE WHEN primaryimageurl != '' THEN 1 ELSE 0 END) as with_images,
            ROUND(100.0 * SUM(CASE WHEN primaryimageurl != '' THEN 1 ELSE 0 END) / COUNT(*), 2) as image_percentage
        FROM artifactmedia
    """
}

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Select Analysis")
selected_query = st.sidebar.selectbox("Choose Query", list(QUERIES.keys()))

# Execute query
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        df = execute_query(QUERIES[selected_query])
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
        
        # Download option
        csv = df.to_csv(index=False)
        st.download_button("Download CSV", csv, "results.csv", "text/csv")
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_etl_pipeline(max_pages=5):
    """Execute full ETL workflow"""
    print("Starting ETL pipeline...")
    
    # Extract
    artifacts = fetch_all_artifacts(max_pages=max_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    load_metadata(metadata_df)
    load_media(media_df)
    load_colors(colors_df)
    
    print("ETL pipeline completed successfully!")

# Run the pipeline
if __name__ == "__main__":
    run_etl_pipeline(max_pages=10)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handle Missing Data

```python
def safe_get(artifact, key, default=''):
    """Safely extract nested values"""
    value = artifact.get(key, default)
    return value if value is not None else default
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Batch database operations** for performance (use `executemany()`)
3. **Handle API pagination** properly to get complete datasets
4. **Implement retry logic** for API calls
5. **Normalize data** into separate tables with foreign keys
6. **Index frequently queried columns** in your database
7. **Cache query results** in Streamlit with `@st.cache_data`
