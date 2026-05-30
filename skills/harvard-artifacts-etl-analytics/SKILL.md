---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I extract Harvard Art Museums data
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - process Harvard API artifact data
  - set up museum data engineering pipeline
  - query and visualize Harvard artifacts collection
  - implement batch ETL for art museum data
  - analyze artifact metadata with SQL
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipeline development, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application:
- Fetches artifact data from Harvard Art Museums API with pagination
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata
- Visualizes results through interactive Streamlit dashboards

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

**Required packages**:
- `streamlit` - Web application framework
- `pandas` - Data manipulation
- `requests` - API calls
- `mysql-connector-python` or `pymysql` - Database connectivity
- `plotly` - Interactive visualizations

## Database Schema

The project uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    period VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    imageurl VARCHAR(1000),
    description TEXT,
    technique VARCHAR(255),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import os

def fetch_harvard_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_harvard_artifacts(api_key, size=10, page=1)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### Transform: Data Normalization

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata table format"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'period': artifact.get('period', '')[:255],
            'department': artifact.get('department', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Extract media/image data from artifacts"""
    media_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'objectid': objectid,
                'imageurl': img.get('baseimageurl', '')[:1000],
                'description': img.get('description', ''),
                'technique': img.get('technique', '')[:255]
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color data from artifacts"""
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'objectid': objectid,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:100],
                'percent': color.get('percent', 0.0)
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load: Database Insertion

```python
import mysql.connector
from mysql.connector import Error

def get_database_connection():
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
        raise Exception(f"Database connection failed: {e}")

def load_metadata_to_db(df_metadata, connection):
    """Batch insert metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, period, department, 
         classification, dated, accessionyear, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df_metadata.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    return cursor.rowcount

def load_media_to_db(df_media, connection):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (objectid, imageurl, description, technique)
        VALUES (%s, %s, %s, %s)
    """
    
    records = df_media.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    return cursor.rowcount

def load_colors_to_db(df_colors, connection):
    """Batch insert color data"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (objectid, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
    """
    
    records = df_colors.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    return cursor.rowcount
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=5, page_size=100):
    """Execute complete ETL pipeline"""
    api_key = os.getenv('HARVARD_API_KEY')
    connection = get_database_connection()
    
    total_inserted = 0
    
    try:
        for page in range(1, num_pages + 1):
            print(f"Processing page {page}...")
            
            # Extract
            data = fetch_harvard_artifacts(api_key, size=page_size, page=page)
            artifacts = data.get('records', [])
            
            if not artifacts:
                break
            
            # Transform
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            
            # Load
            metadata_count = load_metadata_to_db(df_metadata, connection)
            media_count = load_media_to_db(df_media, connection)
            colors_count = load_colors_to_db(df_colors, connection)
            
            total_inserted += metadata_count
            print(f"Inserted {metadata_count} artifacts, {media_count} images, {colors_count} colors")
        
        return total_inserted
        
    finally:
        connection.close()
```

## Analytical SQL Queries

```python
# Example analytical queries for the application

ANALYTICAL_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Most Viewed Artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "Artifacts with Most Images": """
        SELECT m.objectid, m.title, COUNT(a.mediaid) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.objectid = a.objectid
        GROUP BY m.objectid, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as artifacts_acquired
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 30
    """
}

def execute_query(connection, query_name):
    """Execute analytical query and return results"""
    cursor = connection.cursor(dictionary=True)
    query = ANALYTICAL_QUERIES.get(query_name)
    
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

## Streamlit Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for ETL controls
    st.sidebar.header("ETL Pipeline")
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            records = run_etl_pipeline(num_pages=5, page_size=100)
            st.sidebar.success(f"Loaded {records} artifacts!")
    
    # Main analytics section
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        connection = get_database_connection()
        
        try:
            df_results = execute_query(connection, query_name)
            
            st.subheader("Results")
            st.dataframe(df_results, use_container_width=True)
            
            # Auto-generate visualization
            if len(df_results.columns) >= 2:
                x_col = df_results.columns[0]
                y_col = df_results.columns[1]
                
                fig = px.bar(
                    df_results.head(15),
                    x=x_col,
                    y=y_col,
                    title=f"{query_name} Visualization"
                )
                st.plotly_chart(fig, use_container_width=True)
        
        finally:
            connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pagination Handler for Large Datasets

```python
def fetch_all_artifacts(api_key, max_records=1000, page_size=100):
    """Fetch multiple pages of artifacts with pagination"""
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < max_records:
        data = fetch_harvard_artifacts(api_key, size=page_size, page=page)
        records = data.get('records', [])
        
        if not records:
            break
        
        all_artifacts.extend(records)
        page += 1
        
        # Respect API rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Error Handling with Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key, max_retries=3, **kwargs):
    """Fetch data with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_harvard_artifacts(api_key, **kwargs)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
            time.sleep(wait_time)
```

## Troubleshooting

**API Rate Limiting**: Harvard API has rate limits. Add delays between requests:
```python
import time
time.sleep(1)  # Wait 1 second between API calls
```

**Database Connection Issues**: Verify environment variables are set correctly:
```python
required_vars = ['DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
missing = [v for v in required_vars if not os.getenv(v)]
if missing:
    raise ValueError(f"Missing environment variables: {missing}")
```

**Large Dataset Memory Issues**: Process data in batches:
```python
def batch_process(artifacts, batch_size=100):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        # Process batch
        yield batch
```

**Duplicate Key Errors**: Use `ON DUPLICATE KEY UPDATE` or `INSERT IGNORE` in SQL queries to handle re-runs.

**Empty Results**: Check API parameters - ensure `hasimage=1` is appropriate for your use case:
```python
params = {
    'apikey': api_key,
    'size': size,
    'page': page,
    # Remove hasimage filter to get all artifacts
}
```
