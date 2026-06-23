---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and SQL
  - process Harvard artifacts collection data
  - set up data engineering pipeline for art museum data
  - query and visualize Harvard museum artifacts
  - implement batch data loading from Harvard API
  - analyze art collection data with SQL queries
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytics queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application:
- Fetches artifact data from Harvard Art Museums API with pagination
- Performs ETL operations to transform nested JSON into relational tables
- Stores data in MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and color data
- Visualizes results through interactive Plotly charts in Streamlit dashboards

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the database and tables:

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    medium VARCHAR(500),
    provenance TEXT,
    creditline TEXT,
    division VARCHAR(200)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT PRIMARY KEY,
    base_url VARCHAR(500),
    media_type VARCHAR(50),
    format VARCHAR(50),
    description TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Commands

### Run the Streamlit App

```bash
streamlit run app.py
```

### Run ETL Pipeline Programmatically

```python
import os
from dotenv import load_dotenv
import requests
import pandas as pd
import mysql.connector

# Load environment variables
load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')

# Fetch data from API
def fetch_artifacts(num_records=100, page_size=100):
    """Fetch artifacts with pagination handling"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    
    while len(all_records) < num_records:
        params = {
            'apikey': API_KEY,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code != 200:
            break
            
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
    return all_records[:num_records]

# Transform data
def transform_metadata(records):
    """Extract artifact metadata from API response"""
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'medium': record.get('medium'),
            'provenance': record.get('provenance'),
            'creditline': record.get('creditline'),
            'division': record.get('division')
        })
    
    return pd.DataFrame(metadata)

def transform_media(records):
    """Extract media information from artifacts"""
    media_data = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for img in images:
            media_data.append({
                'artifact_id': artifact_id,
                'media_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'media_type': img.get('format'),
                'format': img.get('technique'),
                'description': img.get('description')
            })
    
    return pd.DataFrame(media_data)

def transform_colors(records):
    """Extract color data from artifacts"""
    color_data = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_data)

# Load data to database
def load_to_database(df, table_name):
    """Batch insert DataFrame into MySQL table"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Prepare insert query
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return len(data_tuples)

# Run complete ETL pipeline
def run_etl_pipeline(num_records=100):
    """Execute full ETL workflow"""
    # Extract
    print("Extracting data from API...")
    records = fetch_artifacts(num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(records)
    media_df = transform_media(records)
    colors_df = transform_colors(records)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, 'artifactmetadata')
    load_to_database(media_df, 'artifactmedia')
    load_to_database(colors_df, 'artifactcolors')
    
    print(f"ETL complete: {len(records)} artifacts processed")
```

## Analytics Query Examples

### Execute SQL Queries

```python
def execute_query(query):
    """Run SQL query and return results as DataFrame"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df

# Example: Top 10 cultures by artifact count
query_top_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

results = execute_query(query_top_cultures)
print(results)
```

### Common Analytics Queries

```python
# Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Media availability analysis
query_media_stats = """
SELECT 
    COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
    COUNT(am.media_id) as total_media_files,
    AVG(media_per_artifact) as avg_media_per_artifact
FROM artifactmedia am
JOIN (
    SELECT artifact_id, COUNT(*) as media_per_artifact
    FROM artifactmedia
    GROUP BY artifact_id
) sub ON am.artifact_id = sub.artifact_id
"""

# Color distribution
query_top_colors = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY frequency DESC
LIMIT 15
"""

# Department-wise classification
query_dept_classification = """
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC
"""
```

## Streamlit Visualization Patterns

### Create Interactive Dashboard

```python
import streamlit as st
import plotly.express as px

st.title("Harvard Artifacts Analytics Dashboard")

# Query selector
query_options = {
    "Top Cultures": query_top_cultures,
    "Artifacts by Century": query_by_century,
    "Color Distribution": query_top_colors
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))

# Execute and display
if st.button("Run Query"):
    results = execute_query(query_options[selected_query])
    
    st.dataframe(results)
    
    # Auto-generate visualization
    if len(results.columns) == 2:
        fig = px.bar(
            results, 
            x=results.columns[0], 
            y=results.columns[1],
            title=selected_query
        )
        st.plotly_chart(fig)
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(num_records, requests_per_second=10):
    """Fetch data with rate limiting"""
    delay = 1.0 / requests_per_second
    all_records = []
    
    for page in range(1, (num_records // 100) + 2):
        response = requests.get(url, params={'page': page, 'apikey': API_KEY})
        all_records.extend(response.json().get('records', []))
        time.sleep(delay)
    
    return all_records[:num_records]
```

### Handle Missing Data

```python
def safe_get(record, *keys, default=None):
    """Safely extract nested values from JSON"""
    for key in keys:
        if isinstance(record, dict):
            record = record.get(key, default)
        else:
            return default
    return record

# Usage
culture = safe_get(record, 'culture', default='Unknown')
image_url = safe_get(record, 'images', 0, 'baseimageurl', default='')
```

## Troubleshooting

**API Key Issues:**
- Ensure `.env` file is in project root
- Verify API key is valid at https://harvardartmuseums.org/collections/api
- Check rate limits (default: 2500 requests/day)

**Database Connection Errors:**
- Verify database credentials in `.env`
- Ensure MySQL/TiDB Cloud instance is accessible
- Check firewall rules for remote connections

**Empty Query Results:**
- Confirm tables are populated: `SELECT COUNT(*) FROM artifactmetadata`
- Run ETL pipeline first to load data
- Check for NULL values in filter conditions

**Streamlit Performance:**
- Use `@st.cache_data` decorator for query results
- Limit initial record counts during development
- Implement pagination for large result sets

```python
@st.cache_data
def cached_query(query_text):
    return execute_query(query_text)
```
