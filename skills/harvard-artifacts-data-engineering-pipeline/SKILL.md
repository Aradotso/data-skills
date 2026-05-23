---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - show me how to extract and load Harvard artifacts data to SQL
  - create a data engineering pipeline for museum artifacts
  - help me analyze Harvard Art Museums data with SQL
  - build a Streamlit dashboard for Harvard artifacts collection
  - set up ETL for Harvard Art Museums API
  - query and visualize Harvard museum artifacts data
  - implement batch data ingestion from Harvard API
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL patterns, SQL analytics, and interactive visualization using Streamlit.

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY=your_api_key_here
export DB_HOST=your_database_host
export DB_USER=your_database_user
export DB_PASSWORD=your_database_password
export DB_NAME=your_database_name
```

## API Key Setup

Get your Harvard Art Museums API key:
1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Store it securely in environment variables

```python
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

## Core Components

### 1. API Data Extraction

**Fetching artifacts with pagination:**

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = f"https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

**Handling rate limits and batch collection:**

```python
import time

def collect_all_artifacts(api_key, max_pages=10):
    """
    Collect multiple pages of artifacts with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Collected page {page}: {len(artifacts)} artifacts")
            
            # Rate limiting - be respectful to the API
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### 2. Data Transformation

**Extract artifact metadata:**

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_year': artifact.get('accessionyear'),
            'object_number': artifact.get('objectnumber')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

# Usage
artifacts = collect_all_artifacts(api_key, max_pages=5)
df_metadata = transform_artifact_metadata(artifacts)
```

**Extract media information:**

```python
def transform_artifact_media(artifacts):
    """
    Extract and flatten media/image data from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format'),
                'copyright': img.get('copyright')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)
```

**Extract color data:**

```python
def transform_artifact_colors(artifacts):
    """
    Extract color palette data from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_name': color.get('color'),
                'hex_value': color.get('hex'),
                'percentage': color.get('percent')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. Database Setup and Loading

**Database schema creation:**

```python
import pymysql
import os

def create_database_schema(connection):
    """
    Create the relational database schema
    """
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            classification VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(255),
            dated VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            medium TEXT,
            dimensions TEXT,
            creditline TEXT,
            accession_year INT,
            object_number VARCHAR(255),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url TEXT,
            width INT,
            height INT,
            format VARCHAR(50),
            copyright TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_name VARCHAR(100),
            hex_value VARCHAR(10),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

# Database connection
def get_db_connection():
    return pymysql.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        charset='utf8mb4'
    )
```

**Batch data loading:**

```python
def batch_insert_metadata(df, connection):
    """
    Efficiently insert artifact metadata in batches
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, classification, period, century, 
         dated, department, division, medium, dimensions, creditline, 
         accession_year, object_number)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    print(f"Inserted {len(data)} metadata records")

def batch_insert_media(df, connection):
    """
    Insert artifact media data
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, image_id, base_url, width, height, format, copyright)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
```

### 4. SQL Analytics Queries

**Example analytical queries:**

```python
# Query 1: Artifacts by century
query_by_century = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Top cultures by artifact count
query_by_culture = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 15
"""

# Query 3: Media availability analysis
query_media_stats = """
    SELECT 
        COUNT(DISTINCT m.artifact_id) as artifacts_with_images,
        COUNT(m.id) as total_images,
        AVG(m.width) as avg_width,
        AVG(m.height) as avg_height
    FROM artifactmedia m
"""

# Query 4: Color distribution
query_color_distribution = """
    SELECT 
        color_name,
        COUNT(*) as usage_count,
        AVG(percentage) as avg_percentage
    FROM artifactcolors
    WHERE color_name IS NOT NULL
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query 5: Department analysis
query_departments = """
    SELECT 
        department,
        COUNT(*) as artifact_count,
        COUNT(DISTINCT classification) as classification_types
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""

def execute_query(connection, query):
    """
    Execute SQL query and return results as DataFrame
    """
    return pd.read_sql_query(query, connection)
```

### 5. Streamlit Dashboard

**Main application structure:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    st.write("ETL Pipeline and Data Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # ETL Pipeline Section
    st.header("1. Data Collection")
    
    num_pages = st.number_input("Number of pages to collect", 
                                 min_value=1, max_value=100, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Collecting artifacts..."):
            artifacts = collect_all_artifacts(api_key, max_pages=num_pages)
            st.success(f"Collected {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            conn = get_db_connection()
            batch_insert_metadata(df_metadata, conn)
            batch_insert_media(df_media, conn)
            conn.close()
            st.success("Data loaded successfully")
    
    # Analytics Section
    st.header("2. Analytics Dashboard")
    
    query_options = {
        "Artifacts by Century": query_by_century,
        "Top Cultures": query_by_culture,
        "Media Statistics": query_media_stats,
        "Color Distribution": query_color_distribution,
        "Department Analysis": query_departments
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df_result = execute_query(conn, query_options[selected_query])
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, 
                        x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Pipeline Execution

```python
def run_full_etl_pipeline(api_key, db_connection, max_pages=10):
    """
    Complete ETL pipeline from API to database
    """
    print("Step 1: Extract")
    artifacts = collect_all_artifacts(api_key, max_pages=max_pages)
    
    print("Step 2: Transform")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    print("Step 3: Load")
    batch_insert_metadata(df_metadata, db_connection)
    batch_insert_media(df_media, db_connection)
    batch_insert_colors(df_colors, db_connection)
    
    print(f"ETL Complete: {len(artifacts)} artifacts processed")
    
    return {
        'artifacts': len(artifacts),
        'metadata_records': len(df_metadata),
        'media_records': len(df_media),
        'color_records': len(df_colors)
    }
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_execution():
    """
    ETL with comprehensive error handling
    """
    try:
        api_key = os.getenv('HARVARD_API_KEY')
        if not api_key:
            raise ValueError("HARVARD_API_KEY not set")
        
        conn = get_db_connection()
        create_database_schema(conn)
        
        stats = run_full_etl_pipeline(api_key, conn, max_pages=5)
        logger.info(f"ETL completed successfully: {stats}")
        
        conn.close()
        
    except Exception as e:
        logger.error(f"ETL failed: {e}")
        raise
```

## Troubleshooting

**API Rate Limiting:**
- Add delays between requests: `time.sleep(0.5)`
- Implement exponential backoff for retries
- Cache responses to reduce API calls

**Database Connection Issues:**
- Verify environment variables are set correctly
- Check firewall rules for database access
- Use connection pooling for production: `pymysql.pooling`

**Large Dataset Handling:**
- Process data in batches (100-1000 records)
- Use `executemany()` for bulk inserts
- Implement pagination properly
- Consider using chunked DataFrame operations

**Missing Data Fields:**
- Always use `.get()` with defaults for nested JSON
- Validate data before insertion
- Use `NULL` constraints appropriately in SQL schema

**Memory Issues:**
- Process artifacts in batches instead of all at once
- Use generators for large datasets
- Clear DataFrames after loading: `del df_metadata`
