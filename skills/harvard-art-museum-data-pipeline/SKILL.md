---
name: harvard-art-museum-data-pipeline
description: ETL pipeline and analytics application for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - analyze Harvard Art Museums collection
  - create a Streamlit data analytics dashboard
  - extract data from Harvard Art Museums API
  - set up museum artifact data warehouse
  - visualize art collection metadata with SQL
  - implement ETL for cultural heritage data
  - build end-to-end art museum analytics app
---

# Harvard Art Museum Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through a Streamlit dashboard.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Core Dependencies:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

Create a `.env` file or configure environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The pipeline creates three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensiontext TEXT,
    provenance TEXT,
    creditline TEXT,
    accessionyear INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    imageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py
```

The application will be available at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)

total_records = data['info']['totalrecords']
artifacts = data['records']
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def extract_artifact_metadata(artifact):
    """Extract metadata from artifact JSON"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', '')[:500],
        'culture': artifact.get('culture', '')[:255],
        'century': artifact.get('century', '')[:100],
        'classification': artifact.get('classification', '')[:255],
        'division': artifact.get('division', '')[:255],
        'department': artifact.get('department', '')[:255],
        'dated': artifact.get('dated', '')[:255],
        'medium': artifact.get('medium', '')[:500],
        'technique': artifact.get('technique', '')[:500],
        'dimensiontext': artifact.get('dimensions'),
        'provenance': artifact.get('provenance'),
        'creditline': artifact.get('creditline'),
        'accessionyear': artifact.get('accessionyear')
    }

def extract_artifact_media(artifact):
    """Extract media/image data from artifact"""
    media_list = []
    artifact_id = artifact.get('id')
    
    images = artifact.get('images', [])
    for img in images:
        media_list.append({
            'artifact_id': artifact_id,
            'imageurl': img.get('baseimageurl'),
            'iiifbaseuri': img.get('iiifbaseuri')
        })
    
    return media_list

def extract_artifact_colors(artifact):
    """Extract color data from artifact"""
    colors_list = []
    artifact_id = artifact.get('id')
    
    colors = artifact.get('colors', [])
    for color in colors:
        colors_list.append({
            'artifact_id': artifact_id,
            'color': color.get('color'),
            'spectrum': color.get('spectrum'),
            'percent': color.get('percent')
        })
    
    return colors_list

def transform_artifacts(artifacts):
    """Transform raw artifact data into structured dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        metadata_list.append(extract_artifact_metadata(artifact))
        media_list.extend(extract_artifact_media(artifact))
        colors_list.extend(extract_artifact_colors(artifact))
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_metadata_to_db(df_metadata, connection):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, division, department, 
     dated, medium, technique, dimensiontext, provenance, creditline, accessionyear)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    records = df_metadata.to_records(index=False)
    data = [tuple(record) for record in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_media_to_db(df_media, connection):
    """Batch insert artifact media"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, imageurl, iiifbaseuri)
    VALUES (%s, %s, %s)
    """
    
    records = df_media.to_records(index=False)
    data = [tuple(record) for record in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_colors_to_db(df_colors, connection):
    """Batch insert artifact colors"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df_colors.to_records(index=False)
    data = [tuple(record) for record in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
```

### 4. Streamlit Analytics Dashboard

```python
import streamlit as st
import plotly.express as px

def run_analytics_query(query, connection):
    """Execute SQL query and return results as DataFrame"""
    return pd.read_sql(query, connection)

# Streamlit UI
st.title("Harvard Art Museum Analytics Dashboard")

# Sidebar for query selection
queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 20
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
    "Top Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors 
        WHERE color IS NOT NULL 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 15
    """,
    "Artifacts with Images": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'With Images' ELSE 'Without Images' END as status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY status
    """
}

selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute query
if st.button("Run Analysis"):
    connection = get_db_connection()
    
    if connection:
        df_result = run_analytics_query(queries[selected_query], connection)
        
        st.subheader(f"Results: {selected_query}")
        st.dataframe(df_result)
        
        # Visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(
                df_result, 
                x=df_result.columns[0], 
                y=df_result.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig)
        
        connection.close()
```

## Common Patterns

### Full ETL Pipeline Execution

```python
def run_full_etl_pipeline(num_pages=5):
    """Execute complete ETL pipeline"""
    api_key = os.getenv('HARVARD_API_KEY')
    connection = get_db_connection()
    
    if not connection:
        print("Failed to connect to database")
        return
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = fetch_artifacts(api_key, size=100, page=page)
        artifacts = data['records']
        
        # Transform
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        # Load
        load_metadata_to_db(df_metadata, connection)
        load_media_to_db(df_media, connection)
        load_colors_to_db(df_colors, connection)
        
        print(f"Loaded {len(artifacts)} artifacts from page {page}")
    
    connection.close()
    print("ETL pipeline completed successfully")

# Execute
run_full_etl_pipeline(num_pages=10)
```

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(api_key, total_pages, delay=1):
    """Fetch data with rate limiting"""
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, size=100, page=page)
        all_artifacts.extend(data['records'])
        
        time.sleep(delay)  # Respect API rate limits
        
        if page % 10 == 0:
            print(f"Fetched {len(all_artifacts)} artifacts so far...")
    
    return all_artifacts
```

## Troubleshooting

### API Key Issues
- Verify API key is valid at Harvard Art Museums developer portal
- Check `.env` file is in project root and properly loaded
- Ensure `HARVARD_API_KEY` environment variable is set

### Database Connection Errors
- Verify database credentials in environment variables
- Check firewall rules allow connection to database host
- For TiDB Cloud, ensure IP whitelist is configured
- Test connection with: `mysql -h HOST -u USER -p`

### Memory Issues with Large Datasets
```python
# Process in smaller chunks
def chunked_etl(total_pages, chunk_size=10):
    for i in range(0, total_pages, chunk_size):
        end = min(i + chunk_size, total_pages)
        run_full_etl_pipeline(num_pages=end-i)
        print(f"Completed chunk {i}-{end}")
```

### Streamlit Performance
- Cache database connections: Use `@st.cache_resource`
- Cache query results: Use `@st.cache_data`
- Limit initial data load size

```python
@st.cache_resource
def get_cached_connection():
    return get_db_connection()

@st.cache_data(ttl=3600)
def cached_query(query):
    conn = get_cached_connection()
    return pd.read_sql(query, conn)
```
