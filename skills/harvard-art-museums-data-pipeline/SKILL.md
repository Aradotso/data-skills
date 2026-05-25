---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards for the Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - build a data pipeline for Harvard Art Museums
  - create an ETL process for art collection data
  - analyze Harvard artifacts with SQL
  - set up a museum data analytics dashboard
  - extract and transform Harvard Art Museums API data
  - build a Streamlit app for art collection analytics
  - query Harvard museum collection data
  - create visualizations from art museum metadata
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates:

- **API Integration**: Fetch artifact data from Harvard Art Museums with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data
- **SQL Storage**: Store normalized data in MySQL/TiDB with proper relationships
- **Analytics**: Run predefined SQL queries for insights
- **Visualization**: Interactive Streamlit dashboard with Plotly charts

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Required Dependencies

```txt
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
mysql-connector-python>=8.0.0
plotly>=5.11.0
python-dotenv>=0.19.0
```

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    imageurl VARCHAR(1000),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Configuration

### API Setup

Get your API key from [Harvard Art Museums API](https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z4PFYCoMg/viewform).

Store in environment variables or `.env` file:

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Connection

```python
import mysql.connector
import os

def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """Fetch artifacts with pagination and rate limiting."""
    all_records = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_records.extend(data.get('records', []))
            print(f"Fetched page {page}/{num_pages}")
            time.sleep(1)  # Rate limiting
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_records
```

### Transform: Normalize JSON to Tables

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into normalized DataFrames."""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in raw_data:
        # Metadata
        metadata_list.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'dated': record.get('dated'),
            'accessionyear': record.get('accessionyear'),
            'technique': record.get('technique'),
            'medium': record.get('medium')
        })
        
        # Media
        for image in record.get('images', []):
            media_list.append({
                'artifact_id': record.get('id'),
                'imageurl': image.get('baseimageurl'),
                'caption': image.get('caption')
            })
        
        # Colors
        for color in record.get('colors', []):
            colors_list.append({
                'artifact_id': record.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into SQL Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert DataFrames into SQL tables."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, accessionyear, technique, medium)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, imageurl, caption)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

## Analytics Queries

### Sample SQL Queries for Insights

```python
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
        GROUP BY department 
        ORDER BY count DESC
    """,
    
    "Color Usage Patterns": """
        SELECT color, 
               COUNT(*) as frequency, 
               AVG(percentage) as avg_percentage 
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_images
        FROM artifactmedia m
    """
}

def run_analytics_query(query_name):
    """Execute an analytics query and return results."""
    conn = get_db_connection()
    df = pd.read_sql(ANALYTICS_QUERIES[query_name], conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics Dashboard", "Database Info"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_database_info()

def show_analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = run_analytics_query(query_name)
            
            st.subheader("Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Run the Dashboard

```bash
streamlit run app.py
```

## Common Patterns

### Full ETL Pipeline Execution

```python
def run_etl_pipeline(api_key, num_pages=5):
    """Complete ETL workflow."""
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_artifacts(api_key, num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print(f"ETL complete! Processed {len(metadata_df)} artifacts.")
```

### Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the highest artifact ID in database."""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_etl(api_key):
    """Load only new artifacts."""
    latest_id = get_latest_artifact_id()
    
    # Fetch with ID filter
    params = {
        'apikey': api_key,
        'q': f'id:>{latest_id}'
    }
    response = requests.get(BASE_URL, params=params)
    # Process new records...
```

## Troubleshooting

### API Rate Limiting

```python
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    """Create session with automatic retries."""
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

### Database Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_pooled_connection():
    return db_pool.get_connection()
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=None):
    """Safely extract nested dictionary values."""
    value = dictionary.get(key, default)
    return value if value not in [None, '', 'null'] else default

# Use in transformation
metadata_list.append({
    'id': record.get('id'),
    'title': safe_get(record, 'title', 'Untitled'),
    'culture': safe_get(record, 'culture', 'Unknown'),
    'accessionyear': safe_get(record, 'accessionyear', 0)
})
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement rate limiting** when fetching from the API (sleep between requests)
3. **Use batch inserts** for better database performance
4. **Handle NULL values** explicitly in SQL schema and transformations
5. **Index foreign keys** for query performance
6. **Cache API responses** for development to avoid unnecessary API calls
7. **Log ETL progress** for monitoring and debugging

This skill enables AI agents to build production-ready data pipelines for museum collection analysis with proper ETL practices, SQL analytics, and interactive visualizations.
