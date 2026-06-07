---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines using the Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL pipeline for museum artifact data
  - create analytics dashboard for Harvard art collection
  - extract and analyze Harvard museum data
  - build Streamlit app with museum API data
  - implement SQL analytics for art artifacts
  - configure Harvard Art Museums data engineering project
  - visualize museum collection data with Python
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build complete data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, relational database design, SQL analytics, and interactive visualization with Streamlit.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Fetches artifact data from Harvard Art Museums API with pagination
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on museum collections
- Visualizes insights using Plotly charts in Streamlit dashboards

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

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store credentials in environment variables or `.env` file:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Configuration

Create the required database schema:

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    rank INT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url VARCHAR(500),
    image_count INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
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

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into normalized tables"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'dated': artifact.get('dated', ''),
            'accession_number': artifact.get('accessionnumber', ''),
            'rank': artifact.get('rank', 0),
            'url': artifact.get('url', '')
        })
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'base_image_url': artifact.get('primaryimageurl', ''),
                'image_count': len(artifact.get('images', []))
            })
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percentage': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_data_to_db(metadata_df, media_df, colors_df):
    """Load transformed data into SQL database"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch insert)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, dated, accession_number, rank, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, base_image_url, image_count)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        color_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percentage)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(color_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Streamlit Analytics Dashboard

```python
import streamlit as st
import plotly.express as px

def run_analytics_query(query_name, sql_query):
    """Execute analytical SQL query and return results"""
    conn = get_db_connection()
    df = pd.read_sql(sql_query, conn)
    conn.close()
    return df

# Streamlit app structure
st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
queries = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top 10 Cultures": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Century Distribution": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN image_count > 0 THEN 'Has Images' ELSE 'No Images' END as media_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY media_status
    """
}

query_choice = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute and display
if st.button("Run Analysis"):
    with st.spinner("Querying database..."):
        result_df = run_analytics_query(query_choice, queries[query_choice])
        
        st.subheader(f"Results: {query_choice}")
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(
                result_df,
                x=result_df.columns[0],
                y=result_df.columns[1],
                title=query_choice
            )
            st.plotly_chart(fig)
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(max_pages=5):
    """Run complete ETL pipeline"""
    
    # Extract
    st.info("📥 Extracting data from API...")
    artifacts = fetch_all_artifacts(max_pages=max_pages)
    
    # Transform
    st.info("🔄 Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Load
    st.info("💾 Loading data to database...")
    load_data_to_db(metadata_df, media_df, colors_df)
    
    st.success(f"✅ ETL Complete! Processed {len(metadata_df)} artifacts")
```

### Error Handling with Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(page, max_retries=3):
    """Fetch data with exponential backoff"""
    
    for attempt in range(max_retries):
        try:
            records, info = fetch_artifacts(page=page)
            return records, info
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                st.warning(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

## Troubleshooting

**API Key Issues:**
- Verify your API key at https://www.harvardartmuseums.org/collections/api
- Check environment variables are loaded correctly
- Ensure `.env` file is in project root

**Database Connection Errors:**
```python
# Test connection
try:
    conn = get_db_connection()
    print("✓ Database connected successfully")
    conn.close()
except Error as e:
    print(f"✗ Connection failed: {e}")
```

**Missing Data in Queries:**
- Some artifacts may have null values for culture, period, or century
- Add `WHERE field IS NOT NULL` filters in queries
- Use `COALESCE()` for default values

**Streamlit Performance:**
- Use `@st.cache_data` decorator for expensive queries
- Limit initial page loads with smaller datasets
- Implement pagination for large result sets

```python
@st.cache_data(ttl=3600)
def cached_query(sql):
    conn = get_db_connection()
    df = pd.read_sql(sql, conn)
    conn.close()
    return df
```
