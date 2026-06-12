---
name: harvard-artifacts-data-engineering-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL pipeline for museum artifact data
  - create analytics dashboard for Harvard artifacts collection
  - build Streamlit app with Harvard Art Museums data
  - query and visualize museum artifact metadata
  - design SQL schema for art museum collections
  - fetch and store Harvard Art Museums API data
  - analyze artifact data by culture and century
---

# Harvard Artifacts Data Engineering Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipeline development, SQL database design, analytical query execution, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database structures
- Loads data into MySQL/TiDB Cloud with proper schema design
- Executes analytical SQL queries for insights
- Visualizes results using Plotly in a Streamlit dashboard

**Architecture:** API → ETL → SQL → Analytics → Visualization

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

### Dependencies

```txt
streamlit>=1.25.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.0.33
plotly>=5.14.0
python-dotenv>=1.0.0
```

## Configuration

### API Key Setup

Obtain an API key from the Harvard Art Museums API:
https://www.harvardartmuseums.org/collections/api

Store credentials securely:

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

def get_db_connection():
    return mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```python
def create_tables(connection):
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            dated VARCHAR(255),
            classification VARCHAR(255),
            medium VARCHAR(500),
            department VARCHAR(255),
            division VARCHAR(255),
            technique VARCHAR(500),
            period VARCHAR(255),
            url VARCHAR(500),
            creditline TEXT,
            description TEXT,
            INDEX idx_culture (culture),
            INDEX idx_century (century),
            INDEX idx_classification (classification)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl VARCHAR(500),
            primaryimageurl VARCHAR(500),
            imagepermissionlevel INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
            INDEX idx_artifact_id (artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
            INDEX idx_artifact_id (artifact_id),
            INDEX idx_color (color)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(API_BASE_URL, params=params)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
                print(f"Fetched page {page}/{num_pages} - {len(data['records'])} records")
            
            # Rate limiting
            time.sleep(1)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return all_artifacts
```

### Transform: Process JSON Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into structured DataFrames
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'description': artifact.get('description')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        artifact_id = artifact.get('id')
        media = {
            'artifact_id': artifact_id,
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
        }
        media_records.append(media)
        
        # Extract color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                color_records.append(color_record)
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

### Load: Insert into Database

```python
def load_data(connection, dataframes):
    """
    Batch insert data into SQL database
    """
    cursor = connection.cursor()
    
    # Load metadata
    metadata_df = dataframes['metadata']
    if not metadata_df.empty:
        insert_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, dated, classification, medium, 
             department, division, technique, period, url, creditline, description)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(insert_query, metadata_df.values.tolist())
    
    # Load media
    media_df = dataframes['media']
    if not media_df.empty:
        insert_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(insert_query, media_df.values.tolist())
    
    # Load colors
    colors_df = dataframes['colors']
    if not colors_df.empty:
        insert_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(insert_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
ANALYTICAL_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL 
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'With Images' 
                 ELSE 'Without Images' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY count DESC
        LIMIT 15
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

def execute_query(connection, query_name):
    """Execute analytical query and return DataFrame"""
    cursor = connection.cursor()
    cursor.execute(ANALYTICAL_QUERIES[query_name])
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Complete Application Template

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("Data Collection")
        num_pages = st.slider("Number of pages to fetch", 1, 10, 5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_artifacts(api_key, num_pages=num_pages)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                dataframes = transform_artifacts(artifacts)
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                conn = get_db_connection()
                create_tables(conn)
                load_data(conn, dataframes)
                conn.close()
                st.success("Data loaded to database")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    conn = get_db_connection()
    
    # Query selector
    query_options = list(ANALYTICAL_QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        df = execute_query(conn, selected_query)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df, use_container_width=True)
        
        # Visualization
        if len(df.columns) >= 2:
            st.subheader("Visualization")
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query.replace('_', ' ').title())
            st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

### Run the Application

```bash
streamlit run app.py
```

## Common Patterns

### Pattern: Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key, last_id):
    """Fetch only artifacts with ID greater than last_id"""
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'q': f'id:>{last_id}'
    }
    response = requests.get(API_BASE_URL, params=params)
    return response.json().get('records', [])
```

### Pattern: Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, connection):
    """ETL pipeline with comprehensive error handling"""
    try:
        logger.info("Starting ETL pipeline")
        artifacts = fetch_artifacts(api_key)
        
        if not artifacts:
            logger.warning("No artifacts fetched")
            return False
        
        dataframes = transform_artifacts(artifacts)
        load_data(connection, dataframes)
        
        logger.info(f"Successfully processed {len(artifacts)} artifacts")
        return True
        
    except Exception as e:
        logger.error(f"ETL pipeline failed: {e}")
        return False
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from functools import wraps

def rate_limit(calls_per_second=1):
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = 1.0 / calls_per_second - elapsed
            if wait_time > 0:
                time.sleep(wait_time)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limit(calls_per_second=1)
def fetch_with_rate_limit(url, params):
    return requests.get(url, params=params)
```

### Database Connection Issues

```python
def get_db_connection_with_retry(max_retries=3):
    """Get database connection with retry logic"""
    for attempt in range(max_retries):
        try:
            conn = mysql.connector.connect(**db_config)
            return conn
        except mysql.connector.Error as e:
            logger.warning(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
            else:
                raise
```

### Handling NULL Values

```python
def clean_dataframe(df):
    """Clean DataFrame for SQL insertion"""
    # Replace None with empty string for VARCHAR columns
    str_columns = df.select_dtypes(include=['object']).columns
    df[str_columns] = df[str_columns].fillna('')
    
    # Replace None with 0 for numeric columns
    num_columns = df.select_dtypes(include=['float64', 'int64']).columns
    df[num_columns] = df[num_columns].fillna(0)
    
    return df
```

This skill provides comprehensive guidance for building production-ready ETL pipelines with the Harvard Art Museums API, including data engineering best practices, SQL analytics, and interactive visualization.
