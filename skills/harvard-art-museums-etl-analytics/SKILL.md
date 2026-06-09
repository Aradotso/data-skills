---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and analyze Harvard Art Museums API data
  - set up artifact data warehouse with SQL
  - visualize museum collection data with Streamlit
  - implement data engineering pipeline for art collections
  - query and analyze artifact metadata from Harvard museums
  - build museum data analytics application
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for collecting, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL patterns including API pagination, relational data modeling, batch processing, and interactive analytics dashboards built with Streamlit.

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

Key capabilities:
- Paginated data collection from Harvard Art Museums API
- Transformation of nested JSON into normalized relational tables
- MySQL/TiDB Cloud storage with proper foreign key relationships
- 20+ predefined SQL analytical queries
- Interactive Plotly visualizations via Streamlit

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

Required packages:
- `streamlit` - Web application framework
- `pandas` - Data manipulation
- `requests` - API calls
- `mysql-connector-python` or `pymysql` - Database connectivity
- `plotly` - Interactive visualizations

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store it securely:
```python
import os

API_KEY = os.environ.get('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.environ.get('DB_HOST'),
    'user': os.environ.get('DB_USER'),
    'password': os.environ.get('DB_PASSWORD'),
    'database': os.environ.get('DB_NAME'),
    'port': int(os.environ.get('DB_PORT', 3306))
}

def get_db_connection():
    return mysql.connector.connect(**db_config)
```

## Database Schema

The ETL process creates three normalized tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    description TEXT,
    technique VARCHAR(500)
);

-- Media/images table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    idsid INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Color palette table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifacts with pagination and rate limiting
    """
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} records")
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
        
        # Rate limiting
        time.sleep(0.5)
    
    return artifacts
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    color_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'description': artifact.get('description'),
            'technique': artifact.get('technique')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'idsid': img.get('idsid')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(color_list)
    )
```

### Load: Batch Insert to SQL

```python
def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert dataframes into MySQL database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_tuples = [tuple(x) for x in metadata_df.to_numpy()]
    metadata_sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, dated, url, description, technique)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_sql, metadata_tuples)
    
    # Insert media
    if not media_df.empty:
        media_tuples = [tuple(x) for x in media_df.to_numpy()]
        media_sql = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, format, idsid)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_sql, media_tuples)
    
    # Insert colors
    if not colors_df.empty:
        color_tuples = [tuple(x) for x in colors_df.to_numpy()]
        color_sql = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(color_sql, color_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        sql_analytics_page()
    elif page == "Visualizations":
        visualization_page()

def data_collection_page():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_pages = st.slider("Number of pages to fetch", 1, 10, 5)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(api_key, num_pages)
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            
        st.success(f"✅ Successfully loaded {len(metadata_df)} artifacts!")

if __name__ == "__main__":
    main()
```

## SQL Analytics Queries

### Common Analytical Queries

```python
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL 
                 THEN 'With Media' 
                 ELSE 'No Media' 
            END as media_status,
            COUNT(DISTINCT a.id) as artifact_count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, 
               AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count,
               COUNT(DISTINCT culture) as cultures
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_query(query_name):
    """
    Execute a predefined analytical query
    """
    conn = get_db_connection()
    query = ANALYTICS_QUERIES[query_name]
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df

def sql_analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            results = execute_query(query_name)
            
        st.subheader("Query Results")
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) >= 2:
            fig = px.bar(
                results.head(20),
                x=results.columns[0],
                y=results.columns[1],
                title=query_name
            )
            st.plotly_chart(fig)
```

## Visualization Examples

```python
def create_culture_chart(df):
    """
    Create interactive bar chart for culture distribution
    """
    fig = px.bar(
        df,
        x='culture',
        y='artifact_count',
        title='Top Cultures by Artifact Count',
        labels={'artifact_count': 'Number of Artifacts'},
        color='artifact_count',
        color_continuous_scale='viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_distribution(df):
    """
    Create pie chart for color usage
    """
    fig = px.pie(
        df,
        values='usage_count',
        names='color',
        title='Color Distribution in Collection'
    )
    return fig
```

## Common Patterns

### Complete ETL Workflow

```python
def run_full_etl_pipeline(api_key, num_pages=5):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Step 1: Extracting data from API...")
    raw_data = fetch_artifacts(api_key, num_pages)
    
    # Transform
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Data quality checks
    print(f"Metadata records: {len(metadata_df)}")
    print(f"Media records: {len(media_df)}")
    print(f"Color records: {len(colors_df)}")
    
    # Load
    print("Step 3: Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("✅ ETL pipeline completed successfully!")
    return metadata_df
```

### Error Handling

```python
def safe_api_call(api_key, page, max_retries=3):
    """
    API call with retry logic
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(
                BASE_URL,
                params={'apikey': api_key, 'page': page},
                timeout=10
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
time.sleep(0.5)  # 500ms delay

# Or use rate limiting library
from ratelimit import limits, sleep_and_retry

@sleep_and_retry
@limits(calls=10, period=1)
def rate_limited_fetch(url, params):
    return requests.get(url, params=params)
```

### Database Connection Issues
```python
# Connection pooling for better performance
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_pooled_connection():
    return connection_pool.get_connection()
```

### Memory Management for Large Datasets
```python
def batch_process_artifacts(api_key, total_pages, batch_size=5):
    """
    Process in batches to manage memory
    """
    for start_page in range(1, total_pages + 1, batch_size):
        end_page = min(start_page + batch_size, total_pages + 1)
        print(f"Processing pages {start_page} to {end_page-1}")
        
        raw_data = fetch_artifacts(api_key, end_page - start_page, start_page)
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        load_to_database(metadata_df, media_df, colors_df)
        
        # Clear memory
        del raw_data, metadata_df, media_df, colors_df
```

### Streamlit Caching for Performance
```python
@st.cache_data(ttl=3600)
def cached_query_execution(query_name):
    """
    Cache query results for 1 hour
    """
    return execute_query(query_name)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8501

# With environment variables
HARVARD_API_KEY=xxx DB_HOST=localhost streamlit run app.py
```
