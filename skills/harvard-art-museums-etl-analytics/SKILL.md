---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data pipelines with Harvard Art Museums API, ETL processes, SQL analytics, and Streamlit visualization
triggers:
  - create a data pipeline with Harvard Art Museums API
  - build ETL pipeline for museum artifact data
  - set up Harvard Art Museums analytics dashboard
  - extract and analyze art collection data
  - build Streamlit app with museum API data
  - create SQL analytics for Harvard artifacts
  - design ETL workflow for museum collections
  - visualize Harvard Art Museums data with Plotly
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact collections.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational database tables
- **SQL Analytics**: Store and query artifact metadata, media, and color information
- **Visualization**: Interactive Streamlit dashboards with Plotly charts
- **Real-world Patterns**: Simulates production data engineering workflows

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
# Create a .env file with:
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_database_host
# DB_USER=your_database_user
# DB_PASSWORD=your_database_password
# DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to your `.env` file

## Running the Application

```bash
# Launch the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Architecture Overview

```
API → Extract → Transform → Load → SQL Database → Analytics → Visualization
```

- **Extract**: Fetch data from Harvard Art Museums API
- **Transform**: Parse nested JSON, normalize data structures
- **Load**: Batch insert into MySQL/TiDB tables
- **Analytics**: Execute SQL queries for insights
- **Visualize**: Render results in Streamlit with Plotly

## Database Schema

### Tables Structure

```python
# artifactmetadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    accessionyear INT
);

# artifactmedia table
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    mediacount INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

# artifactcolors table
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

## Core Code Patterns

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages with pagination
def fetch_all_artifacts(max_records=1000, page_size=100):
    """Fetch artifacts with pagination"""
    all_records = []
    page = 1
    
    while len(all_records) < max_records:
        data = fetch_artifacts(page=page, size=page_size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_records[:max_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(records):
    """Transform raw API data into metadata DataFrame"""
    metadata_list = []
    
    for record in records:
        metadata = {
            'objectid': record.get('objectid'),
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:255],
            'period': record.get('period', '')[:255],
            'century': record.get('century', '')[:255],
            'dated': record.get('dated', '')[:255],
            'classification': record.get('classification', '')[:255],
            'medium': record.get('medium', '')[:500],
            'department': record.get('department', '')[:255],
            'division': record.get('division', '')[:255],
            'technique': record.get('technique', '')[:500],
            'accessionyear': record.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(records):
    """Transform media information"""
    media_list = []
    
    for record in records:
        media = {
            'objectid': record.get('objectid'),
            'baseimageurl': record.get('baseimageurl', ''),
            'primaryimageurl': record.get('primaryimageurl', ''),
            'mediacount': len(record.get('images', []))
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(records):
    """Transform color information from nested arrays"""
    colors_list = []
    
    for record in records:
        objectid = record.get('objectid')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data = {
                'objectid': objectid,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

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
        print(f"Database connection error: {e}")
        return None

def load_metadata(df, connection):
    """Batch insert metadata"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, dated, classification, 
         medium, department, division, technique, accessionyear)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE objectid=objectid
    """
    
    data_tuples = [tuple(x) for x in df.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    cursor.close()

def load_media(df, connection):
    """Load media data"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (objectid, baseimageurl, primaryimageurl, mediacount)
        VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    cursor.close()

def load_colors(df, connection):
    """Load color data"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    cursor.close()
```

### 4. Complete ETL Pipeline

```python
def run_etl_pipeline(max_records=1000):
    """Execute full ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Extracting data from API...")
    records = fetch_all_artifacts(max_records=max_records)
    print(f"Extracted {len(records)} records")
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(records)
    media_df = transform_artifact_media(records)
    colors_df = transform_artifact_colors(records)
    
    # Load
    print("Loading data to database...")
    connection = get_db_connection()
    
    if connection:
        load_metadata(metadata_df, connection)
        print(f"Loaded {len(metadata_df)} metadata records")
        
        load_media(media_df, connection)
        print(f"Loaded {len(media_df)} media records")
        
        load_colors(colors_df, connection)
        print(f"Loaded {len(colors_df)} color records")
        
        connection.close()
        print("ETL Pipeline completed successfully!")
    else:
        print("ETL Pipeline failed - no database connection")
```

### 5. Analytical SQL Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN mediacount > 0 THEN 'With Images' ELSE 'No Images' END as status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY status
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "classification_distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query_name):
    """Execute analytical query and return DataFrame"""
    connection = get_db_connection()
    
    if connection:
        query = ANALYTICAL_QUERIES[query_name]
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    
    return pd.DataFrame()
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    st.markdown("### Data Engineering & Analytics Application")
    
    # Sidebar
    st.sidebar.header("Options")
    
    # ETL Section
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(max_records=500)
            st.success("ETL completed!")
    
    # Analytics Section
    st.header("📊 Analytics Queries")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Image Availability": "artifacts_with_images",
        "Top Colors": "top_colors",
        "Department Distribution": "artifacts_by_department",
        "Classification Types": "classification_distribution"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        query_key = query_options[selected_query]
        df = execute_query(query_key)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualization
        if not df.empty and len(df.columns) >= 2:
            st.subheader("Visualization")
            fig = px.bar(
                df, 
                x=df.columns[0], 
                y=df.columns[1],
                title=selected_query,
                labels={df.columns[0]: df.columns[0].title(), 
                        df.columns[1]: df.columns[1].title()}
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Configuration

### Environment Variables

Create a `.env` file:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts

# Optional: TiDB Cloud
TIDB_HOST=gateway01.us-west-2.prod.aws.tidbcloud.com
TIDB_PORT=4000
TIDB_USER=your_tidb_user
TIDB_PASSWORD=your_tidb_password
```

### Database Initialization

```python
def initialize_database():
    """Create database tables if they don't exist"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    # Create tables (SQL from schema above)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(255),
            dated VARCHAR(255),
            classification VARCHAR(255),
            medium VARCHAR(500),
            department VARCHAR(255),
            division VARCHAR(255),
            technique VARCHAR(500),
            accessionyear INT
        )
    """)
    
    # Create other tables...
    connection.commit()
    cursor.close()
    connection.close()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_loaded_id(connection):
    """Get the last objectid loaded"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Load only new artifacts"""
    connection = get_db_connection()
    last_id = get_last_loaded_id(connection)
    
    # Fetch only new records
    records = fetch_artifacts_after_id(last_id)
    # Transform and load...
```

### Error Handling

```python
def safe_etl_pipeline():
    """ETL with error handling"""
    try:
        records = fetch_all_artifacts(max_records=1000)
    except requests.exceptions.RequestException as e:
        st.error(f"API Error: {e}")
        return
    
    try:
        connection = get_db_connection()
        if not connection:
            raise Exception("Database connection failed")
        
        # ETL operations...
        
    except mysql.connector.Error as e:
        st.error(f"Database Error: {e}")
    finally:
        if connection and connection.is_connected():
            connection.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retry():
    """Create session with retry logic"""
    session = requests.Session()
    retry = Retry(total=3, backoff_factor=1)
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session

# Use in API calls
session = get_session_with_retry()
response = session.get(url, params=params)
```

### Database Connection Issues

```python
# Test database connection
def test_connection():
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def chunked_etl(total_records=10000, chunk_size=500):
    """Process data in chunks to manage memory"""
    for offset in range(0, total_records, chunk_size):
        records = fetch_artifacts(page=offset//100 + 1, size=min(chunk_size, total_records - offset))
        
        # Process chunk
        metadata_df = transform_artifact_metadata(records)
        connection = get_db_connection()
        load_metadata(metadata_df, connection)
        connection.close()
        
        print(f"Processed {offset + len(records)}/{total_records}")
```

This skill enables AI agents to build production-ready data engineering pipelines with the Harvard Art Museums API, covering extraction, transformation, storage, analytics, and visualization.
