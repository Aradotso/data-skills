---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, MySQL, and Streamlit
triggers:
  - build etl pipeline for harvard art museums api
  - create analytics dashboard with streamlit and museum data
  - extract transform load harvard artifacts data
  - query harvard art museums database with sql
  - visualize museum collection data with plotly
  - set up data engineering pipeline for art museum api
  - analyze harvard art collection with python
  - build museum artifacts data warehouse
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics solution for the Harvard Art Museums API. It demonstrates ETL pipeline construction, relational database design, SQL analytics, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud using batch inserts
- **Analyzes** data with 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly dashboards in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages (typical requirements.txt):
# streamlit
# pandas
# requests
# mysql-connector-python
# plotly
# python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Store in environment variable

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifacts metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    url VARCHAR(500),
    verificationlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifacts media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    type VARCHAR(50),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifacts colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will open in browser at http://localhost:8501
```

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

def extract_all_artifacts(num_pages=10):
    """
    Extract artifacts across multiple pages
    """
    api_key = os.getenv('HARVARD_API_KEY')
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

### Transform: Normalizing JSON to Relational Tables

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'verificationlevel': artifact.get('verificationlevel'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        }
        metadata_list.append(metadata)
        
        # Extract media (nested)
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_id': image.get('imageid'),
                    'type': 'image',
                    'baseimageurl': image.get('baseimageurl'),
                    'iiifbaseuri': image.get('iiifbaseuri'),
                    'publiccaption': image.get('publiccaption')
                }
                media_list.append(media)
        
        # Extract colors (nested array)
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into MySQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """
    Batch insert metadata into database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, division, dated, url, verificationlevel, 
         totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    data = df_metadata.to_records(index=False).tolist()
    
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return cursor.rowcount

def load_all_data(df_metadata, df_media, df_colors):
    """
    Load all transformed data into database
    """
    metadata_count = load_metadata(df_metadata)
    print(f"Loaded {metadata_count} metadata records")
    
    # Similar functions for media and colors
    # load_media(df_media)
    # load_colors(df_colors)
```

## SQL Analytics Queries

### Sample Analytics Functions

```python
def run_sql_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Query 1: Artifacts by Culture
def query_artifacts_by_culture():
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """
    return run_sql_query(query)

# Query 2: Artifacts by Century
def query_artifacts_by_century():
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """
    return run_sql_query(query)

# Query 3: Media Availability
def query_media_availability():
    query = """
        SELECT 
            CASE 
                WHEN media_id IS NOT NULL THEN 'Has Media'
                ELSE 'No Media'
            END as media_status,
            COUNT(DISTINCT m.id) as count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia am ON m.id = am.artifact_id
        GROUP BY media_status
    """
    return run_sql_query(query)

# Query 4: Top Colors Used
def query_top_colors():
    query = """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """
    return run_sql_query(query)

# Query 5: Department Distribution
def query_department_distribution():
    query = """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
    return run_sql_query(query)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_option = st.sidebar.selectbox(
        "Choose Query",
        [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Media Availability",
            "Top Colors",
            "Department Distribution"
        ]
    )
    
    # Execute selected query
    if query_option == "Artifacts by Culture":
        st.header("Artifacts Distribution by Culture")
        df = query_artifacts_by_culture()
        
        # Display table
        st.dataframe(df)
        
        # Visualization
        fig = px.bar(df, x='culture', y='count', 
                     title='Artifacts by Culture',
                     labels={'count': 'Number of Artifacts'})
        st.plotly_chart(fig)
    
    elif query_option == "Top Colors":
        st.header("Most Common Colors in Artifacts")
        df = query_top_colors()
        
        st.dataframe(df)
        
        fig = px.bar(df, x='color', y='frequency',
                     title='Color Frequency',
                     color='avg_percent',
                     labels={'frequency': 'Frequency', 'avg_percent': 'Avg %'})
        st.plotly_chart(fig)
    
    # Add more query options...

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Workflow

```python
def run_etl_pipeline(num_pages=10):
    """
    Complete ETL pipeline execution
    """
    # Extract
    st.info("Extracting data from Harvard API...")
    artifacts = extract_all_artifacts(num_pages)
    
    # Transform
    st.info("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    
    # Load
    st.info("Loading data into database...")
    load_all_data(df_metadata, df_media, df_colors)
    
    st.success(f"ETL complete! Processed {len(artifacts)} artifacts")
```

### Error Handling for API Requests

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """
    Fetch with exponential backoff retry
    """
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(api_key, page)
            return data
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_artifacts_with_rate_limit(api_key, page, delay=1):
    """
    Add delay between requests to avoid rate limiting
    """
    time.sleep(delay)
    return fetch_artifacts(api_key, page)
```

### Database Connection Issues

```python
def test_db_connection():
    """
    Test database connectivity
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handling NULL Values

```python
def clean_dataframe(df):
    """
    Handle NULL values before database insert
    """
    # Replace NaN with None for SQL NULL
    df = df.where(pd.notnull(df), None)
    
    # Truncate long strings
    for col in df.select_dtypes(include=['object']):
        df[col] = df[col].astype(str).str[:500]
    
    return df
```

### Memory Management for Large Datasets

```python
def etl_in_batches(total_pages, batch_size=5):
    """
    Process ETL in batches to manage memory
    """
    for start_page in range(1, total_pages + 1, batch_size):
        end_page = min(start_page + batch_size - 1, total_pages)
        
        artifacts = []
        for page in range(start_page, end_page + 1):
            data = fetch_artifacts(os.getenv('HARVARD_API_KEY'), page)
            artifacts.extend(data.get('records', []))
        
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        load_all_data(df_metadata, df_media, df_colors)
        
        print(f"Processed pages {start_page}-{end_page}")
```
