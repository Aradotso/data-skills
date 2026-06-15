---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create analytics dashboard for museum artifact data
  - fetch and store Harvard artifacts in SQL database
  - visualize museum collection data with Streamlit
  - set up Harvard Art Museums API data pipeline
  - query and analyze art museum artifacts with SQL
  - build end-to-end museum data engineering project
  - integrate Harvard API with database and visualization
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It implements a complete ETL pipeline that extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases (MySQL/TiDB), and visualizes insights through an interactive Streamlit dashboard.

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

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Configuration

```python
import os
from dotenv import load_dotenv
import mysql.connector

load_dotenv()

# Database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Database Schema

The project uses three main tables with relational structure:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
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
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        return data
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

# Fetch multiple pages
def fetch_all_artifacts(num_pages=5, page_size=100):
    all_records = []
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=page_size)
        if data and 'records' in data:
            all_records.extend(data['records'])
    return all_records
```

### Transform: Process JSON Data

```python
import pandas as pd

def transform_artifact_metadata(records):
    """
    Transform artifact records into structured metadata
    """
    metadata_list = []
    
    for record in records:
        metadata = {
            'id': record.get('id'),
            'title': record.get('title', 'Unknown')[:500],
            'culture': record.get('culture', 'Unknown')[:255],
            'century': record.get('century', 'Unknown')[:100],
            'classification': record.get('classification', 'Unknown')[:255],
            'department': record.get('department', 'Unknown')[:255],
            'division': record.get('division', 'Unknown')[:255],
            'technique': record.get('technique', '')[:500],
            'medium': record.get('medium', '')[:500],
            'dated': record.get('dated', '')[:255],
            'period': record.get('period', '')[:255],
            'accession_number': record.get('accessionyear', '')[:100],
            'url': record.get('url', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(records):
    """
    Extract media information from artifacts
    """
    media_list = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'base_url': img.get('baseimageurl', ''),
                'format': img.get('format', 'unknown'),
                'height': img.get('height', 0),
                'width': img.get('width', 0)
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(records):
    """
    Extract color information from artifacts
    """
    color_list = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Insert Data into SQL

```python
def load_metadata_to_db(df, connection):
    """
    Batch insert metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, 
     technique, medium, dated, period, accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media_to_db(df, connection):
    """
    Batch insert media data into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, base_url, format, height, width)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")

def load_colors_to_db(df, connection):
    """
    Batch insert color data into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color_hex, color_percent)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")
```

## Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Starting ETL Pipeline...")
    records = fetch_all_artifacts(num_pages=num_pages)
    print(f"Extracted {len(records)} records")
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(records)
    df_media = transform_artifact_media(records)
    df_colors = transform_artifact_colors(records)
    
    # Load
    print("Loading data to database...")
    connection = get_db_connection()
    
    try:
        load_metadata_to_db(df_metadata, connection)
        load_media_to_db(df_media, connection)
        load_colors_to_db(df_colors, connection)
        print("ETL Pipeline completed successfully!")
    except Exception as e:
        print(f"Error during load: {e}")
        connection.rollback()
    finally:
        connection.close()

# Run the pipeline
if __name__ == "__main__":
    run_etl_pipeline(num_pages=10)
```

## Analytics Queries

### Sample SQL Queries for Insights

```python
# Predefined analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN media_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(DISTINCT am.id) as count
        FROM artifactmetadata am
        LEFT JOIN artifactmedia md ON am.id = md.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Classification Breakdown": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query_name):
    """
    Execute analytical query and return results as DataFrame
    """
    connection = get_db_connection()
    cursor = connection.cursor()
    
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        return None
    
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    connection.close()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(query_name)
            
            if df is not None and not df.empty:
                st.subheader(f"Results: {query_name}")
                
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
                    st.plotly_chart(fig)
                
                # Download option
                csv = df.to_csv(index=False)
                st.download_button(
                    label="Download CSV",
                    data=csv,
                    file_name=f"{query_name.replace(' ', '_')}.csv",
                    mime="text/csv"
                )
            else:
                st.warning("No results found")
    
    # ETL Pipeline Section
    st.sidebar.header("ETL Pipeline")
    num_pages = st.sidebar.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(num_pages=num_pages)
            st.success("ETL Pipeline completed!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, size=100, delay=0.5):
    """
    Fetch data with rate limiting to avoid API throttling
    """
    data = fetch_artifacts(page=page, size=size)
    time.sleep(delay)  # Delay between requests
    return data
```

### Error Handling for API Calls

```python
def safe_fetch_artifacts(page, retries=3):
    """
    Fetch with retry logic
    """
    for attempt in range(retries):
        try:
            return fetch_artifacts(page=page)
        except requests.exceptions.RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt < retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

### Incremental Data Loading

```python
def get_latest_artifact_id(connection):
    """
    Get the latest artifact ID from database for incremental loads
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(new_records):
    """
    Only load new artifacts not already in database
    """
    connection = get_db_connection()
    latest_id = get_latest_artifact_id(connection)
    
    new_records_filtered = [r for r in new_records if r.get('id', 0) > latest_id]
    print(f"Found {len(new_records_filtered)} new records")
    
    # Transform and load new records
    df_metadata = transform_artifact_metadata(new_records_filtered)
    load_metadata_to_db(df_metadata, connection)
    
    connection.close()
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')

if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")

# Test API connection
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': api_key, 'size': 1}
)
print(f"API Status: {response.status_code}")
```

### Database Connection Issues

```python
# Test database connection
try:
    connection = get_db_connection()
    cursor = connection.cursor()
    cursor.execute("SELECT 1")
    print("Database connection successful")
    connection.close()
except Exception as e:
    print(f"Database connection failed: {e}")
```

### Handling NULL Values

```python
# Clean data before insertion
def clean_value(value, max_length=None):
    """
    Clean and validate data values
    """
    if value is None or value == '':
        return 'Unknown'
    
    value_str = str(value)
    if max_length:
        return value_str[:max_length]
    return value_str
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(num_pages, batch_size=5):
    """
    Process large datasets in batches to manage memory
    """
    for start_page in range(1, num_pages + 1, batch_size):
        end_page = min(start_page + batch_size, num_pages + 1)
        print(f"Processing pages {start_page} to {end_page - 1}")
        
        records = fetch_all_artifacts(num_pages=batch_size, start_page=start_page)
        
        # Transform and load batch
        df_metadata = transform_artifact_metadata(records)
        connection = get_db_connection()
        load_metadata_to_db(df_metadata, connection)
        connection.close()
        
        # Clear memory
        del records, df_metadata
```

This skill enables AI coding agents to help developers build complete ETL pipelines for museum artifact data, implement SQL analytics, and create interactive visualizations using the Harvard Art Museums API.
