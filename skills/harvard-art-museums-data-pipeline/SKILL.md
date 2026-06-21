---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch artifacts from Harvard Art Museums API
  - build an ETL pipeline for museum data
  - create analytics dashboard with Streamlit and museum data
  - query Harvard Art Museums API with pagination
  - transform nested JSON art data into SQL tables
  - visualize museum artifact data with Plotly
  - set up TiDB database for art collections
  - analyze artifact metadata by culture and century
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates real-world ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract nested JSON, transform into relational format, load into SQL databases
- **SQL Analytics**: Pre-built analytical queries for culture, century, media, and color analysis
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for data exploration

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

Get your free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

Create a `.env` file or configure in your Streamlit app:

```python
import os

# Set API key
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

Set up MySQL or TiDB Cloud connection:

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

Create the following tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    copyright TEXT,
    creditline TEXT,
    totalpageviews INT,
    totaluniquepageviews INT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

## API Data Extraction

### Fetch Artifacts with Pagination

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    size = 100  # Max per page
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': size,
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

### Extract Specific Fields

```python
def extract_artifact_metadata(artifact):
    """
    Extract metadata fields from artifact JSON
    """
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title'),
        'culture': artifact.get('culture'),
        'period': artifact.get('period'),
        'century': artifact.get('century'),
        'dated': artifact.get('dated'),
        'classification': artifact.get('classification'),
        'division': artifact.get('division'),
        'department': artifact.get('department'),
        'technique': artifact.get('technique'),
        'medium': artifact.get('medium'),
        'dimensions': artifact.get('dimensions'),
        'copyright': artifact.get('copyright'),
        'creditline': artifact.get('creditline'),
        'totalpageviews': artifact.get('totalpageviews', 0),
        'totaluniquepageviews': artifact.get('totaluniquepageviews', 0),
        'url': artifact.get('url')
    }
```

## ETL Transform Operations

### Transform Nested Color Data

```python
import pandas as pd

def transform_colors(artifacts):
    """
    Extract and flatten color data from artifacts
    """
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

### Transform Media/Image Data

```python
def transform_media(artifacts):
    """
    Extract image and media URLs
    """
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        media_records.append({
            'artifact_id': artifact_id,
            'iiifbaseuri': artifact.get('iiifbaseuri'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl')
        })
    
    return pd.DataFrame(media_records)
```

## Load Data into SQL

### Batch Insert with Error Handling

```python
def load_metadata_to_sql(df, connection):
    """
    Load artifact metadata into SQL database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, dated, classification, 
     division, department, technique, medium, dimensions, 
     copyright, creditline, totalpageviews, totaluniquepageviews, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} records")
    except Exception as e:
        print(f"Error inserting data: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Analytics Queries

### Culture Distribution Analysis

```python
def get_culture_distribution(connection):
    """
    Query artifacts grouped by culture
    """
    query = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 20
    """
    
    return pd.read_sql(query, connection)
```

### Century Analysis

```python
def get_century_distribution(connection):
    """
    Analyze artifact distribution by century
    """
    query = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY century
    """
    
    return pd.read_sql(query, connection)
```

### Color Analysis

```python
def get_dominant_colors(connection):
    """
    Find most common colors across artifacts
    """
    query = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 15
    """
    
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for API key input
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection
    conn = get_database_connection()
    
    # Analytics section
    st.header("📊 Analytics")
    
    analysis_type = st.selectbox(
        "Select Analysis",
        ["Culture Distribution", "Century Timeline", "Color Analysis", 
         "Department Statistics", "Media Availability"]
    )
    
    if analysis_type == "Culture Distribution":
        df = get_culture_distribution(conn)
        fig = px.bar(df, x='culture', y='count', 
                     title='Artifacts by Culture')
        st.plotly_chart(fig)
    
    elif analysis_type == "Color Analysis":
        df = get_dominant_colors(conn)
        fig = px.bar(df, x='color', y='frequency',
                     title='Most Common Colors in Collection')
        st.plotly_chart(fig)
```

### ETL Pipeline in Streamlit

```python
def run_etl_pipeline(api_key, num_records, connection):
    """
    Execute full ETL pipeline from Streamlit UI
    """
    with st.spinner(f"Fetching {num_records} artifacts..."):
        artifacts = fetch_artifacts(api_key, num_records)
    
    st.success(f"Fetched {len(artifacts)} artifacts")
    
    with st.spinner("Transforming data..."):
        # Transform metadata
        metadata_list = [extract_artifact_metadata(a) for a in artifacts]
        df_metadata = pd.DataFrame(metadata_list)
        
        # Transform colors
        df_colors = transform_colors(artifacts)
        
        # Transform media
        df_media = transform_media(artifacts)
    
    st.success("Data transformation complete")
    
    with st.spinner("Loading to database..."):
        load_metadata_to_sql(df_metadata, connection)
        # Load colors and media similarly
    
    st.success("✅ ETL Pipeline Complete!")
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Error Handling for API Calls

```python
def safe_api_call(url, params, max_retries=3):
    """
    API call with retry logic
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Null Handling in Transforms

```python
def safe_get(data, key, default=None):
    """
    Safely extract values with null handling
    """
    value = data.get(key, default)
    return value if value not in [None, '', 'null'] else default
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests using `time.sleep(0.5)`

**Database Connection Issues**: Verify credentials and network access to TiDB/MySQL

**Memory Issues with Large Datasets**: Process data in batches rather than loading all at once

**Missing Data Fields**: Use `.get()` with defaults instead of direct dictionary access

**Encoding Errors**: Ensure UTF-8 encoding when writing to database: `connection.set_charset_collation('utf8mb4')`
