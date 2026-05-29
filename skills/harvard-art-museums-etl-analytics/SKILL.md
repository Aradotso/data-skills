---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - set up Harvard artifacts collection analytics app
  - create museum data engineering workflow with Streamlit
  - analyze Harvard Art Museums API data with SQL
  - implement artifact metadata ETL and visualization
  - build interactive analytics dashboard for museum collections
  - extract and transform Harvard museum artifacts data
  - create data pipeline for art collection analytics
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering and analytics solution for the Harvard Art Museums API. It demonstrates:

- **API Integration**: Paginated data collection from Harvard Art Museums with rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data
- **SQL Database**: Relational schema with proper foreign key relationships
- **Analytics**: 20+ predefined SQL queries for insights
- **Visualization**: Interactive Streamlit dashboards with Plotly charts

**Architecture**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Key Setup

1. Get your Harvard Art Museums API key from: https://docs.harvardartmuseums.org/
2. Store in environment variable:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

Set up database connection using environment variables:

```bash
export DB_HOST="your_database_host"
export DB_PORT="4000"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

For TiDB Cloud or MySQL:

```python
import os
import mysql.connector

def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        ssl_ca='/path/to/ca.pem' if os.getenv('DB_SSL_CA') else None
    )
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    period VARCHAR(255),
    accessionyear INT,
    totalpageviews INT DEFAULT 0,
    totaluniquepageviews INT DEFAULT 0
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    imagecount INT DEFAULT 0,
    videocount INT DEFAULT 0,
    primaryimageurl VARCHAR(1000),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': min(page_size, num_records - len(all_records)),
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            if not records:
                break
                
            all_records.extend(records)
            print(f"Fetched page {page}: {len(records)} records")
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching data: {e}")
            break
    
    return all_records[:num_records]
```

### Transform: Process JSON to Dataframes

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Metadata
        metadata_list.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
        
        # Media
        media_list.append({
            'objectid': artifact.get('objectid'),
            'imagecount': len(artifact.get('images', [])),
            'videocount': len(artifact.get('videos', [])),
            'primaryimageurl': artifact.get('primaryimageurl', '')[:1000]
        })
        
        # Colors
        for color_data in artifact.get('colors', []):
            colors_list.append({
                'objectid': artifact.get('objectid'),
                'color': color_data.get('color', '')[:50],
                'spectrum': color_data.get('spectrum', '')[:50],
                'hue': color_data.get('hue', '')[:50],
                'percent': color_data.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df, connection):
    """
    Load dataframes into SQL database using batch inserts
    """
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, classification, department, 
         division, technique, medium, dated, period, accessionyear, 
         totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (objectid, imagecount, videocount, primaryimageurl)
        VALUES (%s, %s, %s, %s)
    """
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

### Complete ETL Function

```python
def run_etl_pipeline(api_key, num_records=100):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Loading data to database...")
    connection = get_db_connection()
    load_to_database(metadata_df, media_df, colors_df, connection)
    connection.close()
    
    print("ETL pipeline completed successfully!")
    return metadata_df, media_df, colors_df
```

## Analytics Queries

### Sample SQL Analytics

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN imagecount > 0 THEN 'With Images'
                ELSE 'No Images'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Color Analysis": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Most Viewed Artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """
}

def execute_analytics_query(query_name, connection):
    """
    Execute analytics query and return results as dataframe
    """
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        return None
    
    df = pd.read_sql(query, connection)
    return df
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("End-to-end Data Engineering & Analytics Pipeline")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # ETL Section
    st.header("📥 ETL Pipeline")
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        if not api_key:
            st.error("Please provide API key")
        else:
            with st.spinner("Running ETL pipeline..."):
                metadata_df, media_df, colors_df = run_etl_pipeline(api_key, num_records)
                st.success(f"Successfully processed {len(metadata_df)} artifacts!")
                st.dataframe(metadata_df.head())
    
    # Analytics Section
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analytics Query", 
                              list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        connection = get_db_connection()
        results_df = execute_analytics_query(query_name, connection)
        connection.close()
        
        if results_df is not None and not results_df.empty:
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(results_df, 
                           x=results_df.columns[0], 
                           y=results_df.columns[1],
                           title=query_name)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Run Application

```bash
streamlit run app.py
```

## Common Patterns

### Batch Processing Large Datasets

```python
def batch_process_artifacts(api_key, total_records=1000, batch_size=100):
    """
    Process large datasets in batches to avoid memory issues
    """
    connection = get_db_connection()
    
    for offset in range(0, total_records, batch_size):
        print(f"Processing batch: {offset} to {offset + batch_size}")
        raw_data = fetch_artifacts(api_key, batch_size)
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        load_to_database(metadata_df, media_df, colors_df, connection)
        time.sleep(1)  # Rate limiting between batches
    
    connection.close()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_execution(api_key, num_records):
    """
    ETL with comprehensive error handling
    """
    try:
        raw_data = fetch_artifacts(api_key, num_records)
        if not raw_data:
            logger.warning("No data fetched from API")
            return None
        
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        
        connection = get_db_connection()
        load_to_database(metadata_df, media_df, colors_df, connection)
        connection.close()
        
        logger.info(f"ETL completed: {len(metadata_df)} records processed")
        return metadata_df
        
    except Exception as e:
        logger.error(f"ETL failed: {str(e)}")
        raise
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues

For SSL/TLS connection problems with TiDB Cloud:

```python
import ssl

def get_db_connection_with_ssl():
    ssl_context = ssl.create_default_context()
    ssl_context.check_hostname = False
    ssl_context.verify_mode = ssl.CERT_NONE
    
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 4000)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        ssl_disabled=False
    )
```

### Memory Issues with Large Datasets

Use chunked processing:

```python
def fetch_artifacts_chunked(api_key, total_records, chunk_size=50):
    """
    Generator function for memory-efficient processing
    """
    for page in range(1, (total_records // chunk_size) + 2):
        params = {
            'apikey': api_key,
            'size': chunk_size,
            'page': page
        }
        response = requests.get("https://api.harvardartmuseums.org/object", params=params)
        records = response.json().get('records', [])
        if not records:
            break
        yield records
```

### Missing or Null Data

Handle missing fields gracefully:

```python
def safe_get(dictionary, key, default='', max_length=None):
    """
    Safely extract values from nested dictionaries
    """
    value = dictionary.get(key, default)
    if value is None:
        value = default
    if max_length and isinstance(value, str):
        value = value[:max_length]
    return value
```
