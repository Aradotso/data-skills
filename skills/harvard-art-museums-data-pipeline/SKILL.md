---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - build a data pipeline for museum artifacts
  - create an ETL workflow with Harvard Art Museums API
  - analyze art museum collection data with SQL
  - build a Streamlit dashboard for artifact analytics
  - extract and transform museum API data
  - design a data engineering pipeline for cultural artifacts
  - visualize museum collection insights with Plotly
  - create analytics queries for art museum data
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It implements ETL pipelines that extract artifact data, transform nested JSON into relational tables, load into SQL databases, and provide interactive analytics dashboards using Streamlit.

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

### API Key Setup

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store credentials in environment variables or `.env` file:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required database and tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    dated VARCHAR(255),
    accession_number VARCHAR(100),
    division VARCHAR(255)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    base_url TEXT,
    image_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

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
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def extract_all_artifacts(api_key, total_records=1000):
    """
    Extract multiple pages of artifact data
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < total_records:
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=size)
        
        if not data.get('records'):
            break
            
        artifacts.extend(data['records'])
        page += 1
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return artifacts[:total_records]
```

### 2. Data Transformation

```python
import pandas as pd

def transform_metadata(artifacts):
    """
    Transform artifact JSON into structured metadata
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'dated': artifact.get('dated'),
            'accession_number': artifact.get('accessionyear'),
            'division': artifact.get('division')
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """
    Extract and flatten media/image data
    """
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Extract primary image
        if artifact.get('primaryimageurl'):
            media_records.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'base_url': artifact.get('baseimageurl'),
                'image_url': artifact.get('primaryimageurl')
            })
        
        # Extract additional images
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'base_url': image.get('baseimageurl'),
                'image_url': image.get('iiifbaseuri')
            })
    
    return pd.DataFrame(media_records)

def transform_colors(artifacts):
    """
    Extract color palette data
    """
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df):
    """
    Bulk insert metadata into database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     technique, period, dated, accession_number, division)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(df)} metadata records")

def load_media(df):
    """
    Insert media records
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, media_type, base_url, image_url)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(df)} media records")

def load_colors(df):
    """
    Insert color data
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
    VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(df)} color records")
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def run_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Sample Analytics Queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 15
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE department IS NOT NULL 
        GROUP BY department 
        ORDER BY count DESC
    """,
    "Top Colors Used": """
        SELECT color_hex, COUNT(*) as frequency, AVG(color_percent) as avg_percent
        FROM artifactcolors 
        GROUP BY color_hex 
        ORDER BY frequency DESC 
        LIMIT 20
    """,
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(med.id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia med ON m.id = med.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """
}

# Dashboard UI
st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Query Selector
selected_query = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        query = ANALYTICS_QUERIES[selected_query]
        results = run_query(query)
        
        st.subheader("Results")
        st.dataframe(results)
        
        # Auto-visualization
        if len(results.columns) == 2:
            fig = px.bar(results, x=results.columns[0], y=results.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

### 5. Complete ETL Pipeline

```python
def run_etl_pipeline():
    """
    Execute full ETL pipeline
    """
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    print("Extracting artifacts...")
    artifacts = extract_all_artifacts(api_key, total_records=500)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Loading to database...")
    load_metadata(metadata_df)
    load_media(media_df)
    load_colors(colors_df)
    
    print("ETL Pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline()
```

## Running the Application

```bash
# Run ETL pipeline
python etl_pipeline.py

# Launch Streamlit dashboard
streamlit run app.py
```

## Common Analytics Patterns

### Join Queries

```python
# Artifacts with color and media data
query = """
SELECT 
    m.title, 
    m.culture, 
    c.color_hex, 
    c.color_percent,
    COUNT(med.id) as image_count
FROM artifactmetadata m
LEFT JOIN artifactcolors c ON m.id = c.artifact_id
LEFT JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY m.id, c.color_hex
HAVING image_count > 0
ORDER BY c.color_percent DESC
"""
```

### Aggregation Analysis

```python
# Century and classification cross-analysis
query = """
SELECT 
    century, 
    classification, 
    COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND classification IS NOT NULL
GROUP BY century, classification
ORDER BY count DESC
LIMIT 50
"""
```

## Troubleshooting

**API Rate Limiting:**
- Add `time.sleep()` between requests
- Use batch processing with smaller page sizes

**Database Connection Errors:**
- Verify credentials in `.env` file
- Check firewall rules for cloud databases (TiDB Cloud)
- Ensure database exists before running ETL

**Memory Issues with Large Datasets:**
- Process in smaller batches
- Use chunking for DataFrame operations
- Clear variables after loading

**Streamlit Performance:**
- Use `@st.cache_data` for expensive queries
- Limit initial data loads
- Add pagination for large result sets

```python
@st.cache_data(ttl=3600)
def cached_query(query):
    return run_query(query)
```
