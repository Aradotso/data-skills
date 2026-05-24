---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - set up SQL analytics for Harvard artifacts collection
  - build a Streamlit dashboard for art museum data
  - extract and transform Harvard Art Museums API data
  - analyze museum artifacts with SQL queries
  - visualize art collection data with Plotly
  - implement data engineering pipeline for museum collections
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases, and provides interactive analytics through Streamlit dashboards.

## What It Does

- **API Integration**: Connects to Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms into normalized tables (metadata, media, colors)
- **SQL Storage**: Stores structured data in MySQL/TiDB Cloud with proper foreign keys
- **Analytics**: Executes predefined SQL queries for insights (culture distribution, media analysis, color patterns)
- **Visualization**: Generates interactive Plotly charts from query results

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or configure in your app:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Connection

Set up MySQL or TiDB Cloud connection parameters:

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

### 3. Database Schema

Create tables before running ETL:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    description TEXT,
    creditline TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_artifacts=100):
    """Extract artifacts with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    per_page = 100
    
    while len(all_records) < num_artifacts:
        params = {
            'apikey': api_key,
            'page': page,
            'size': per_page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_records.extend(records)
            
            if not records or len(all_records) >= num_artifacts:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_artifacts]
```

### Transform: Normalize JSON to DataFrames

```python
def transform_artifacts(raw_data):
    """Transform nested JSON into relational tables"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Metadata table
        metadata_list.append({
            'id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'description': artifact.get('description'),
            'creditline': artifact.get('creditline')
        })
        
        # Media table
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media_list.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': primary_image,
                'iiifbaseuri': artifact.get('baseimageurl', '')
            })
        
        # Colors table
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into SQL Database

```python
def load_to_database(conn, df_metadata, df_media, df_colors):
    """Batch insert data with error handling"""
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, division, dated, url, description, creditline)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE 
        title=VALUES(title), culture=VALUES(culture)
    """
    
    for _, row in df_metadata.iterrows():
        try:
            cursor.execute(metadata_query, tuple(row))
        except Exception as e:
            print(f"Error inserting metadata: {e}")
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_type, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
    """
    
    for _, row in df_media.iterrows():
        try:
            cursor.execute(media_query, tuple(row))
        except Exception as e:
            print(f"Error inserting media: {e}")
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
    """
    
    for _, row in df_colors.iterrows():
        try:
            cursor.execute(colors_query, tuple(row))
        except Exception as e:
            print(f"Error inserting colors: {e}")
    
    conn.commit()
    cursor.close()
```

## Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

load_dotenv()

def run_etl_pipeline(num_artifacts=100):
    """Execute full ETL pipeline"""
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    raw_data = fetch_artifacts(api_key, num_artifacts)
    print(f"Extracted {len(raw_data)} artifacts")
    
    # Transform
    df_metadata, df_media, df_colors = transform_artifacts(raw_data)
    print(f"Transformed into {len(df_metadata)} metadata, "
          f"{len(df_media)} media, {len(df_colors)} color records")
    
    # Load
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    conn = mysql.connector.connect(**db_config)
    load_to_database(conn, df_metadata, df_media, df_colors)
    conn.close()
    print("Data loaded successfully")

# Run the pipeline
run_etl_pipeline(200)
```

## Analytics Queries

### Query: Artifacts by Culture

```python
def get_artifacts_by_culture(conn, limit=10):
    """Get top cultures with most artifacts"""
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT %s
    """
    df = pd.read_sql(query, conn, params=(limit,))
    return df
```

### Query: Color Distribution Analysis

```python
def get_color_distribution(conn):
    """Analyze color usage across artifacts"""
    query = """
        SELECT 
            c.color,
            COUNT(DISTINCT c.artifact_id) as artifact_count,
            AVG(c.percent) as avg_percent
        FROM artifactcolors c
        GROUP BY c.color
        ORDER BY artifact_count DESC
        LIMIT 15
    """
    df = pd.read_sql(query, conn)
    return df
```

### Query: Artifacts with Media by Department

```python
def get_media_by_department(conn):
    """Count artifacts with images per department"""
    query = """
        SELECT 
            m.department,
            COUNT(DISTINCT m.id) as total_artifacts,
            COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
            ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / 
                  COUNT(DISTINCT m.id), 2) as media_percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia media ON m.id = media.artifact_id
        WHERE m.department IS NOT NULL
        GROUP BY m.department
        ORDER BY artifacts_with_media DESC
    """
    df = pd.read_sql(query, conn)
    return df
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Sidebar configuration
st.sidebar.title("Configuration")
api_key = st.sidebar.text_input("API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))

# Main dashboard
st.title("Harvard Art Museums Data Analytics")

# ETL Section
if st.button("Run ETL Pipeline"):
    with st.spinner("Running ETL..."):
        run_etl_pipeline(100)
        st.success("ETL completed successfully!")

# Analytics Section
st.header("Analytics Dashboard")

conn = mysql.connector.connect(**db_config)

# Culture distribution
st.subheader("Top Cultures")
df_culture = get_artifacts_by_culture(conn, 10)
fig = px.bar(df_culture, x='culture', y='artifact_count',
             title='Artifacts by Culture',
             labels={'artifact_count': 'Count', 'culture': 'Culture'})
st.plotly_chart(fig, use_container_width=True)

# Color analysis
st.subheader("Color Distribution")
df_colors = get_color_distribution(conn)
fig = px.bar(df_colors, x='color', y='artifact_count',
             title='Most Common Colors',
             color='avg_percent',
             labels={'artifact_count': 'Artifact Count', 'color': 'Color'})
st.plotly_chart(fig, use_container_width=True)

# Media availability
st.subheader("Media by Department")
df_media = get_media_by_department(conn)
st.dataframe(df_media, use_container_width=True)

conn.close()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, delay=0.5):
    """Fetch data with rate limiting"""
    artifacts = []
    for page in range(1, 6):
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page, 'size': 100}
        )
        if response.status_code == 200:
            artifacts.extend(response.json()['records'])
        time.sleep(delay)  # Respect API rate limits
    return artifacts
```

### Error Handling in ETL

```python
def safe_extract_field(artifact, field, default=None):
    """Safely extract nested fields"""
    try:
        return artifact.get(field, default)
    except (KeyError, AttributeError):
        return default
```

### Incremental Loading

```python
def get_last_artifact_id(conn):
    """Get the last loaded artifact ID"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(conn, api_key):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(conn)
    # Fetch artifacts with id > last_id
    # Transform and load
```

## Troubleshooting

### API Connection Issues
- Verify API key is valid and active
- Check rate limits (Harvard API: ~2500 requests/day)
- Use `hasimage=1` parameter to filter artifacts with images

### Database Connection Errors
- Ensure MySQL/TiDB is running and accessible
- Check firewall rules for cloud databases
- Verify credentials in environment variables

### Memory Issues with Large Datasets
- Use batch processing with `chunksize` parameter
- Process data in smaller page sizes
- Clear DataFrames after loading: `del df_metadata`

### Empty Query Results
- Check if ETL has completed successfully
- Verify foreign key relationships are correct
- Use `IS NOT NULL` filters for optional fields

### Streamlit Performance
- Cache database connections: `@st.cache_resource`
- Cache query results: `@st.cache_data`
- Limit initial data loads to 100-200 records
