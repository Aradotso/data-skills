---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I set up an ETL pipeline for Harvard Art Museums data
  - create a data pipeline from Harvard Art Museums API to SQL
  - build analytics dashboard for museum artifact data
  - extract and transform Harvard Art Museums API data
  - set up Streamlit app for museum collection analytics
  - query and visualize Harvard Art Museums data with SQL
  - implement batch data ingestion from Harvard API
  - create relational database schema for museum artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates:

- API data extraction with pagination and rate limiting
- ETL pipeline implementation using Python and Pandas
- Relational database design for museum artifact metadata
- SQL analytics queries for cultural heritage insights
- Interactive Streamlit dashboards with Plotly visualizations

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
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### 2. Database Setup

Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    copyright VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    provenance TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_type VARCHAR(50),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color_name VARCHAR(100),
    color_hex VARCHAR(10),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Usage Patterns

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch first page
data = fetch_artifacts(page=1, size=50)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifact_data(raw_artifacts: List[Dict]) -> tuple:
    """Transform nested JSON into relational dataframes"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'copyright': artifact.get('copyright'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionnumber'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        })
        
        # Extract media
        if 'images' in artifact and artifact['images']:
            for img in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'media_url': img.get('baseimageurl')
                })
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color_name': color.get('color'),
                    'color_hex': color.get('hex'),
                    'color_percentage': color.get('percent')
                })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
def load_to_database(df_metadata, df_media, df_colors):
    """Batch insert data into MySQL database"""
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
        (id, title, culture, classification, century, dated, department, 
         division, technique, medium, dimensions, copyright, creditline, 
         accession_number, provenance, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, df_metadata.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, media_url)
        VALUES (%s, %s, %s)
    """
    if not df_media.empty:
        cursor.executemany(media_query, df_media.values.tolist())
    
    # Insert colors
    color_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color_name, color_hex, color_percentage)
        VALUES (%s, %s, %s, %s)
    """
    if not df_colors.empty:
        cursor.executemany(color_query, df_colors.values.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5, page_size=50):
    """Execute complete ETL pipeline"""
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        
        # Extract
        data = fetch_artifacts(page=page, size=page_size)
        artifacts = data['records']
        
        # Transform
        df_meta, df_media, df_colors = transform_artifact_data(artifacts)
        
        all_metadata.append(df_meta)
        all_media.append(df_media)
        all_colors.append(df_colors)
    
    # Combine all pages
    combined_metadata = pd.concat(all_metadata, ignore_index=True)
    combined_media = pd.concat(all_media, ignore_index=True)
    combined_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load
    load_to_database(combined_metadata, combined_media, combined_colors)
    
    print(f"ETL Complete: {len(combined_metadata)} artifacts loaded")
```

## Analytics Queries

### Sample SQL Analytics

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts with images by department
query_dept_images = """
    SELECT m.department, COUNT(DISTINCT m.id) as artifacts_with_images
    FROM artifactmetadata m
    JOIN artifactmedia a ON m.id = a.artifact_id
    GROUP BY m.department
    ORDER BY artifacts_with_images DESC
"""

# Query 3: Most common colors across artifacts
query_colors = """
    SELECT color_name, COUNT(*) as frequency, AVG(color_percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_name
    ORDER BY frequency DESC
    LIMIT 15
"""

# Query 4: Artifacts by century
query_centuries = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
    LIMIT 20
"""
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Collection Analytics")

# Sidebar for configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                             value=os.getenv('HARVARD_API_KEY', ''))
    
    st.header("ETL Controls")
    num_pages = st.slider("Number of pages to fetch", 1, 20, 5)
    page_size = st.selectbox("Records per page", [10, 25, 50, 100], index=2)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            run_etl_pipeline(num_pages=num_pages, page_size=page_size)
            st.success("ETL Complete!")

# Analytics section
st.header("📊 Analytics Dashboard")

analytics_option = st.selectbox(
    "Select Analysis",
    [
        "Top Cultures by Artifact Count",
        "Departments with Most Images",
        "Color Distribution Analysis",
        "Artifacts by Century",
        "Classification Breakdown"
    ]
)

if st.button("Run Query"):
    # Execute query based on selection
    conn = mysql.connector.connect(...)
    
    if analytics_option == "Top Cultures by Artifact Count":
        df = pd.read_sql(query_cultures, conn)
        
        # Display table
        st.dataframe(df)
        
        # Visualization
        fig = px.bar(df, x='culture', y='artifact_count',
                     title='Top 10 Cultures by Artifact Count')
        st.plotly_chart(fig, use_container_width=True)
    
    conn.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, size, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page, size)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
def get_db_connection(max_retries=3):
    """Get database connection with retry logic"""
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                autocommit=True
            )
            return conn
        except mysql.connector.Error as e:
            if attempt < max_retries - 1:
                time.sleep(2)
            else:
                raise e
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=''):
    """Safely extract values from nested dictionaries"""
    value = dictionary.get(key, default)
    return value if value is not None else default

# Use in transformation
metadata_records.append({
    'title': safe_get(artifact, 'title'),
    'culture': safe_get(artifact, 'culture'),
    'classification': safe_get(artifact, 'classification')
})
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

The dashboard will provide interactive controls for ETL execution, query selection, and real-time visualization of Harvard Art Museums collection data.
