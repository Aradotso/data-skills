---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create a data engineering app with Harvard Art Museums API
  - set up artifact analytics dashboard with Streamlit
  - extract and analyze Harvard museum collections
  - build a data pipeline with SQL and visualization
  - create museum artifact analytics application
  - implement ETL for art museum data
  - develop Harvard artifacts data engineering project
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards using Streamlit.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

**Required dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Project Structure

The application consists of three main database tables:

1. **artifactmetadata** - Core artifact information (ID, title, culture, century, department)
2. **artifactmedia** - Media files and images associated with artifacts
3. **artifactcolors** - Color palette data extracted from artifacts

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org'
```

### Database Setup

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

## Key Components

### 1. ETL Pipeline - Extract

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = f'https://api.harvardartmuseums.org/object'
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Paginated data collection
def collect_all_artifacts(api_key, max_pages=10):
    all_artifacts = []
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
        print(f"Collected page {page}/{max_pages}")
    return all_artifacts
```

### 2. ETL Pipeline - Transform

```python
def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata table
    """
    metadata_list = []
    for artifact in artifacts:
        metadata_list.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown')
        })
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image data from nested JSON
    """
    media_list = []
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'image_width': img.get('width'),
                'image_height': img.get('height'),
                'format': img.get('format', 'unknown')
            })
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color palette data
    """
    color_list = []
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            })
    return pd.DataFrame(color_list)
```

### 3. ETL Pipeline - Load

```python
def create_tables(cursor):
    """
    Create database schema
    """
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            department VARCHAR(200),
            classification VARCHAR(200),
            dated VARCHAR(200),
            technique TEXT
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(500),
            image_width INT,
            image_height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)

def load_data_to_sql(df, table_name, cursor, conn):
    """
    Batch insert DataFrame into SQL table
    """
    if df.empty:
        return
    
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    sql = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(sql, data)
    conn.commit()
    print(f"Loaded {len(data)} rows into {table_name}")
```

### 4. Complete ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, max_pages=5):
    """
    Execute full ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    artifacts = collect_all_artifacts(api_key, max_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    create_tables(cursor)
    load_data_to_sql(metadata_df, 'artifactmetadata', cursor, conn)
    load_data_to_sql(media_df, 'artifactmedia', cursor, conn)
    load_data_to_sql(colors_df, 'artifactcolors', cursor, conn)
    
    cursor.close()
    conn.close()
    print("ETL pipeline completed!")
```

### 5. SQL Analytics Queries

```python
# Sample analytical queries for the dashboard

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
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_files
        FROM artifactmedia m
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY spectrum
        ORDER BY count DESC
    """
}

def execute_query(query, db_config):
    """
    Execute SQL query and return results as DataFrame
    """
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = execute_query(ANALYTICS_QUERIES[query_name], db_config)
            
            # Display results
            st.subheader("Query Results")
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
    
    # ETL Controls
    st.sidebar.header("ETL Pipeline")
    if st.sidebar.button("Run ETL"):
        run_etl_pipeline(
            os.getenv('HARVARD_API_KEY'),
            db_config,
            max_pages=5
        )
        st.success("ETL completed!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """
    Fetch with rate limiting to respect API limits
    """
    data = fetch_artifacts(api_key, page)
    time.sleep(delay)
    return data
```

### Error Handling

```python
def safe_extract(artifact, key, default='Unknown'):
    """
    Safely extract values from nested JSON
    """
    try:
        value = artifact.get(key, default)
        return value if value else default
    except:
        return default
```

### Incremental Data Loading

```python
def get_max_artifact_id(cursor):
    """
    Get the last loaded artifact ID for incremental updates
    """
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, db_config):
    """
    Load only new artifacts since last ETL run
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    last_id = get_max_artifact_id(cursor)
    
    # Fetch only artifacts with ID > last_id
    # Implement logic here based on API capabilities
```

## Troubleshooting

**API Connection Issues:**
- Verify API key is valid and set in environment variables
- Check API rate limits (Harvard API: 2500 requests/day)
- Use pagination to avoid timeouts

**Database Connection Errors:**
- Ensure database credentials are correct
- Check firewall rules for remote database connections
- Verify database exists before running ETL

**Data Quality Issues:**
- Handle missing/null values in transform step
- Use `INSERT IGNORE` to avoid duplicate key errors
- Validate data types before loading

**Performance Optimization:**
- Use batch inserts instead of row-by-row
- Index foreign keys for faster joins
- Cache frequently accessed query results

## Advanced Usage

### Custom Analytics Query

```python
def custom_query_interface():
    """
    Allow users to write custom SQL queries
    """
    st.subheader("Custom SQL Query")
    user_query = st.text_area("Enter SQL Query")
    
    if st.button("Execute Custom Query"):
        try:
            df = execute_query(user_query, db_config)
            st.dataframe(df)
        except Exception as e:
            st.error(f"Query Error: {str(e)}")
```

### Export Data

```python
def export_results(df, format='csv'):
    """
    Export query results to file
    """
    if format == 'csv':
        return df.to_csv(index=False).encode('utf-8')
    elif format == 'json':
        return df.to_json(orient='records')
```

This skill enables AI coding agents to help developers build production-ready ETL pipelines for museum data with proper data engineering practices.
