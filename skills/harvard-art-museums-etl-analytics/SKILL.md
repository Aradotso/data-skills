---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Harvard Art Museums API
  - extract and transform art collection data
  - set up SQL database for museum artifacts
  - visualize art museum data with Streamlit
  - implement data engineering workflow for cultural heritage data
  - query and analyze Harvard Art Museums collection
  - build artifact metadata pipeline with Python
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering and analytics application for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit. Use this skill to build similar data pipelines for cultural heritage data or adapt the patterns for other RESTful APIs.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with batch inserts for performance
- **Analyzes** data using 20+ predefined SQL queries for insights
- **Visualizes** results through interactive Plotly charts in a Streamlit dashboard

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)

### Setup

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

### Run the Application

```bash
streamlit run app.py
```

The Streamlit app will open at `http://localhost:8501`

## Configuration

### API Configuration

The Harvard Art Museums API requires authentication. Configure in your Streamlit app or Python scripts:

```python
import os
import requests

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

# API request with pagination
def fetch_artifacts(page=1, size=100):
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size
    }
    response = requests.get(BASE_URL, params=params)
    return response.json()
```

### Database Configuration

Set up MySQL/TiDB connection:

```python
import mysql.connector
import os

def get_db_connection():
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return conn
```

### Database Schema

Create three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    period VARCHAR(200),
    dated VARCHAR(200),
    url TEXT
);

-- Artifact media/images table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def extract_artifacts(num_pages=10):
    """Extract artifact data with rate limiting"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            response = requests.get(
                BASE_URL,
                params={
                    'apikey': os.getenv('HARVARD_API_KEY'),
                    'page': page,
                    'size': 100
                }
            )
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            
            # Rate limiting
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error fetching page {page}: {e}")
            
    return all_artifacts
```

### Transform: Process JSON Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into dataframes for each table"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl'),
                'media_type': 'image'
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Batch Insert into SQL

```python
def load_to_database(metadata_df, media_df, colors_df):
    """Batch load dataframes into MySQL"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Load metadata
    metadata_values = [tuple(row) for row in metadata_df.values]
    cursor.executemany("""
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, technique, period, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """, metadata_values)
    
    # Load media
    media_values = [tuple(row) for row in media_df.values]
    cursor.executemany("""
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
    """, media_values)
    
    # Load colors
    colors_values = [tuple(row) for row in colors_df.values]
    cursor.executemany("""
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
    """, colors_values)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=10):
    """Run complete ETL process"""
    print("Starting ETL pipeline...")
    
    # Extract
    print(f"Extracting {num_pages} pages of artifacts...")
    raw_data = extract_artifacts(num_pages)
    print(f"Extracted {len(raw_data)} artifacts")
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("ETL pipeline complete!")
    return len(raw_data)
```

## Analytics Queries

### Sample SQL Queries

```python
# Query definitions for analytics
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
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
    
    "Departments with Most Artifacts": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN am.artifact_id IS NOT NULL THEN 'With Media' ELSE 'No Media' END as status,
            COUNT(DISTINCT a.id) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia am ON a.id = am.artifact_id
        GROUP BY status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("Data engineering and analytics pipeline for cultural heritage data")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "Analytics Dashboard", "Data Explorer"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    elif page == "Data Explorer":
        show_explorer_page()

def show_etl_page():
    """ETL pipeline interface"""
    st.header("ETL Pipeline")
    
    num_pages = st.number_input("Number of pages to extract", 1, 50, 10)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            count = run_etl_pipeline(num_pages)
            st.success(f"✅ Successfully processed {count} artifacts!")

def show_analytics_page():
    """Analytics dashboard with visualizations"""
    st.header("Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        df = execute_query(query_name)
        
        # Display table
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

def show_explorer_page():
    """Raw data explorer"""
    st.header("Data Explorer")
    
    table = st.selectbox("Select Table", ["artifactmetadata", "artifactmedia", "artifactcolors"])
    
    conn = get_db_connection()
    df = pd.read_sql(f"SELECT * FROM {table} LIMIT 100", conn)
    conn.close()
    
    st.dataframe(df)
    st.write(f"Showing 100 rows from {table}")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Handling API Rate Limits

```python
import time
from functools import wraps

def rate_limited(max_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / max_per_second
    
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        return wrapper
    return decorator

@rate_limited(max_per_second=2)
def fetch_artifact_by_id(artifact_id):
    response = requests.get(
        f"{BASE_URL}/{artifact_id}",
        params={'apikey': os.getenv('HARVARD_API_KEY')}
    )
    return response.json()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_extract(num_pages):
    """Extract with error handling"""
    artifacts = []
    failed_pages = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(page)
            artifacts.extend(data.get('records', []))
            logger.info(f"Successfully fetched page {page}")
        except requests.RequestException as e:
            logger.error(f"Failed to fetch page {page}: {e}")
            failed_pages.append(page)
        except Exception as e:
            logger.error(f"Unexpected error on page {page}: {e}")
            failed_pages.append(page)
    
    if failed_pages:
        logger.warning(f"Failed pages: {failed_pages}")
    
    return artifacts, failed_pages
```

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    cursor.close()
    conn.close()
    return max_id

def incremental_etl():
    """Only load new artifacts"""
    max_existing_id = get_max_artifact_id()
    
    raw_data = extract_artifacts(num_pages=5)
    new_artifacts = [a for a in raw_data if a.get('id', 0) > max_existing_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        logger.info(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        logger.info("No new artifacts to load")
```

## Troubleshooting

### API Key Issues
```python
# Test API connection
def test_api_connection():
    try:
        response = requests.get(
            BASE_URL,
            params={'apikey': os.getenv('HARVARD_API_KEY'), 'size': 1}
        )
        if response.status_code == 200:
            print("✅ API connection successful")
            return True
        else:
            print(f"❌ API returned status {response.status_code}")
            return False
    except Exception as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Database Connection Issues
```python
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Memory Issues with Large Datasets
```python
def chunked_load(artifacts, chunk_size=1000):
    """Load data in chunks to avoid memory issues"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_to_database(metadata_df, media_df, colors_df)
        logger.info(f"Loaded chunk {i//chunk_size + 1}")
```

### Empty/Null Data Handling
```python
def clean_dataframe(df):
    """Clean dataframe before loading"""
    # Replace empty strings with None
    df = df.replace('', None)
    
    # Handle text length limits
    for col in df.select_dtypes(include=['object']).columns:
        df[col] = df[col].str[:500]  # Truncate long strings
    
    return df
```

This skill enables AI agents to help developers build production-ready ETL pipelines for cultural heritage data using the Harvard Art Museums API, with patterns applicable to any REST API data engineering workflow.
