---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and museum data
  - query Harvard artifacts collection with SQL
  - visualize museum artifact data with Plotly
  - set up data engineering pipeline for art collection
  - transform Harvard API JSON to SQL database
  - analyze art museum metadata with Python
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for Harvard Art Museums data. It demonstrates ETL pipeline construction, SQL database design, analytical queries, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

**Key capabilities:**
- Dynamic data collection from Harvard Art Museums API with pagination
- ETL transformation of nested JSON to relational database schema
- SQL analytics with 20+ predefined queries
- Interactive Streamlit dashboard with Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
# Create .env or export variables:
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

## Configuration

### API Setup

Obtain an API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
import requests

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

# Basic API request with pagination
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

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error: {e}")
        return None
```

## Database Schema

The project uses three main tables with relational structure:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    imagecount INT,
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract Phase

```python
import requests
import time

def extract_artifacts(num_records=1000, batch_size=100):
    """Extract artifacts from Harvard API with pagination"""
    artifacts = []
    pages = num_records // batch_size
    
    for page in range(1, pages + 1):
        try:
            response = requests.get(
                BASE_URL,
                params={
                    'apikey': API_KEY,
                    'page': page,
                    'size': batch_size
                }
            )
            response.raise_for_status()
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return artifacts
```

### Transform Phase

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifacts into metadata DataFrame"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        })
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Transform artifacts into media DataFrame"""
    media_list = []
    
    for artifact in artifacts:
        media_list.append({
            'objectid': artifact.get('objectid'),
            'imagecount': artifact.get('totalpageviews', 0),
            'primaryimageurl': artifact.get('primaryimageurl')
        })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Transform nested color data"""
    colors_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            colors_list.append({
                'objectid': object_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent')
            })
    
    return pd.DataFrame(colors_list)
```

### Load Phase

```python
def load_to_database(df, table_name, connection):
    """Load DataFrame to MySQL database"""
    cursor = connection.cursor()
    
    # Create insert query dynamically
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    try:
        cursor.executemany(query, df.values.tolist())
        connection.commit()
        print(f"Loaded {len(df)} records into {table_name}")
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
    finally:
        cursor.close()

# Complete ETL execution
def run_etl_pipeline(num_records=1000):
    """Execute complete ETL pipeline"""
    # Extract
    print("Extracting data from API...")
    artifacts = extract_artifacts(num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Loading to database...")
    connection = create_connection()
    if connection:
        load_to_database(metadata_df, 'artifactmetadata', connection)
        load_to_database(media_df, 'artifactmedia', connection)
        load_to_database(colors_df, 'artifactcolors', connection)
        connection.close()
```

## Analytical SQL Queries

```python
# Sample analytical queries for the dashboard

ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Images Availability": """
        SELECT 
            CASE 
                WHEN imagecount > 0 THEN 'With Images'
                ELSE 'No Images'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as acquisitions
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 20
    """
}

def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Artifacts Analytics Dashboard")
    st.markdown("**ETL Pipeline & SQL Analytics for Harvard Art Museums**")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # ETL Section
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(num_records=500)
            st.success("ETL pipeline completed successfully!")
    
    # Analytics Section
    st.header("SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis Query",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        connection = create_connection()
        if connection:
            query = ANALYTICAL_QUERIES[query_name]
            
            # Display SQL query
            st.code(query, language='sql')
            
            # Execute and display results
            df = execute_query(query, connection)
            st.dataframe(df)
            
            # Auto-visualization
            if len(df.columns) == 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig)
            
            connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access the dashboard at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_latest_objectid(connection):
    """Get the latest objectid to avoid duplicates"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_etl(connection):
    """Load only new artifacts"""
    latest_id = get_latest_objectid(connection)
    
    # Fetch artifacts with objectid > latest_id
    params = {
        'apikey': API_KEY,
        'q': f'objectid:>{latest_id}',
        'size': 100
    }
    response = requests.get(BASE_URL, params=params)
    new_artifacts = response.json().get('records', [])
    
    # Transform and load
    if new_artifacts:
        metadata_df = transform_metadata(new_artifacts)
        load_to_database(metadata_df, 'artifactmetadata', connection)
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_call(url, params, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                logger.error("Max retries reached")
                raise
            time.sleep(2 ** attempt)
```

## Troubleshooting

**API Rate Limiting:**
- Add `time.sleep()` between requests (0.5-1 second recommended)
- Use batch processing with smaller page sizes
- Implement exponential backoff on errors

**Database Connection Issues:**
- Verify environment variables are set correctly
- Check firewall rules for cloud databases (TiDB)
- Use connection pooling for production:

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

connection = db_pool.get_connection()
```

**Memory Issues with Large Datasets:**
- Process data in chunks using generators
- Use `df.to_sql()` with `chunksize` parameter

```python
def load_large_dataset(df, table_name, engine, chunk_size=1000):
    """Load large DataFrame in chunks"""
    df.to_sql(
        table_name, 
        engine, 
        if_exists='append', 
        index=False,
        chunksize=chunk_size
    )
```

**Streamlit Performance:**
- Use `@st.cache_data` for expensive operations
- Implement pagination for large result sets

```python
@st.cache_data
def cached_query(query_text):
    connection = create_connection()
    df = execute_query(query_text, connection)
    connection.close()
    return df
```
