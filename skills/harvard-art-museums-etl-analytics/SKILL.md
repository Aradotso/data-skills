---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard from museum artifact data
  - extract and load Harvard museum artifacts into SQL
  - analyze Harvard Art Museums collection with SQL
  - visualize museum artifact data with Streamlit
  - set up data engineering pipeline for art collection
  - query Harvard Art Museums API and store in database
  - create interactive museum data visualization
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables building end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection app provides a complete data pipeline:
- **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Convert nested JSON into normalized relational tables
- **Load**: Batch insert into MySQL/TiDB Cloud databases
- **Analyze**: Execute predefined SQL analytics queries
- **Visualize**: Display results in interactive Plotly charts via Streamlit

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages include:
# - streamlit
# - pandas
# - requests
# - mysql-connector-python (or pymysql)
# - plotly
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store credentials securely:

```python
# Use environment variables
import os
API_KEY = os.getenv('HARVARD_API_KEY')

# Or use Streamlit secrets (recommended for deployment)
# Create .streamlit/secrets.toml:
# [api]
# harvard_key = "your-api-key-here"
# 
# [database]
# host = "your-db-host"
# user = "your-db-user"
# password = "your-db-password"
# database = "harvard_artifacts"

import streamlit as st
API_KEY = st.secrets["api"]["harvard_key"]
DB_CONFIG = {
    "host": st.secrets["database"]["host"],
    "user": st.secrets["database"]["user"],
    "password": st.secrets["database"]["password"],
    "database": st.secrets["database"]["database"]
}
```

### Database Schema

The project uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accession_year INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    imageid INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key API Methods

### Extracting Data from Harvard API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def extract_all_artifacts(api_key, max_records=1000):
    """
    Extract multiple pages of artifacts with pagination
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return artifacts[:max_records]
```

### ETL Pipeline Implementation

```python
def transform_metadata(artifacts):
    """
    Transform artifact JSON into structured metadata
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'accession_year': artifact.get('accessionyear'),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'url': artifact.get('url', '')
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """
    Extract media/image information
    """
    media = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'objectid': objectid,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format')
            })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """
    Extract color analysis data
    """
    colors = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### Loading Data into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection(config):
    """
    Create database connection
    """
    try:
        connection = mysql.connector.connect(**config)
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df, connection):
    """
    Batch insert metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, classification, department, 
     dated, accession_year, technique, medium, dimensions, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data_tuples = [tuple(row) for row in df.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    return cursor.rowcount

def load_media(df, connection):
    """
    Batch insert media data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (objectid, imageid, baseimageurl, format)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    return cursor.rowcount

def load_colors(df, connection):
    """
    Batch insert color data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    return cursor.rowcount
```

## Analytics SQL Queries

```python
# Sample analytical queries for museum data

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as occurrences, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 10
    """,
    
    "image_availability": """
        SELECT 
            CASE WHEN m.objectid IS NULL THEN 'No Images' ELSE 'Has Images' END as status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY status
    """,
    
    "accession_by_year": """
        SELECT accession_year, COUNT(*) as count
        FROM artifactmetadata
        WHERE accession_year IS NOT NULL AND accession_year > 1800
        GROUP BY accession_year
        ORDER BY accession_year DESC
        LIMIT 20
    """
}

def execute_analytics_query(connection, query_name):
    """
    Execute an analytics query and return DataFrame
    """
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        return None
    
    return pd.read_sql(query, connection)
```

## Streamlit Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("End-to-end ETL pipeline and analytics dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "ETL Pipeline", "Analytics Dashboard", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "ETL Pipeline":
        show_etl_pipeline()
    elif page == "Analytics Dashboard":
        show_analytics_dashboard()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection from API")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=st.secrets.get("api", {}).get("harvard_key", ""))
    
    num_records = st.slider("Number of records to fetch", 100, 2000, 500)
    
    if st.button("Fetch Artifacts"):
        with st.spinner("Fetching data from Harvard API..."):
            artifacts = extract_all_artifacts(api_key, num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
            st.json(artifacts[0])  # Show sample

def show_etl_pipeline():
    st.header("⚙️ ETL Pipeline Execution")
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            # Extract
            artifacts = extract_all_artifacts(st.secrets["api"]["harvard_key"], 500)
            
            # Transform
            metadata_df = transform_metadata(artifacts)
            media_df = transform_media(artifacts)
            colors_df = transform_colors(artifacts)
            
            st.write(f"Transformed {len(metadata_df)} metadata records")
            st.write(f"Transformed {len(media_df)} media records")
            st.write(f"Transformed {len(colors_df)} color records")
            
            # Load
            conn = get_db_connection(DB_CONFIG)
            load_metadata(metadata_df, conn)
            load_media(media_df, conn)
            load_colors(colors_df, conn)
            conn.close()
            
            st.success("ETL Pipeline completed successfully!")

def show_analytics_dashboard():
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection(DB_CONFIG)
        df = execute_analytics_query(conn, query_name)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=f"Analysis: {query_name}")
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Local development
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8080

# Production deployment
streamlit run app.py --server.address 0.0.0.0
```

## Common Patterns

### Rate Limiting API Requests

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """
    Decorator to rate limit API calls
    """
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
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

@rate_limit(calls_per_second=2)
def fetch_artifacts_limited(api_key, page, size):
    return fetch_artifacts(api_key, page, size)
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, db_config, num_records=500):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        # Extract
        st.info("Starting extraction...")
        artifacts = extract_all_artifacts(api_key, num_records)
        
        if not artifacts:
            st.error("No artifacts extracted")
            return False
        
        # Transform
        st.info("Transforming data...")
        metadata_df = transform_metadata(artifacts)
        media_df = transform_media(artifacts)
        colors_df = transform_colors(artifacts)
        
        # Load
        st.info("Loading to database...")
        conn = get_db_connection(db_config)
        
        if conn is None:
            st.error("Database connection failed")
            return False
        
        load_metadata(metadata_df, conn)
        load_media(media_df, conn)
        load_colors(colors_df, conn)
        conn.close()
        
        st.success(f"ETL completed: {len(artifacts)} artifacts processed")
        return True
        
    except requests.RequestException as e:
        st.error(f"API request failed: {e}")
        return False
    except Exception as e:
        st.error(f"ETL pipeline error: {e}")
        return False
```

## Troubleshooting

### API Key Issues

```python
# Validate API key before use
def validate_api_key(api_key):
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={"apikey": api_key, "size": 1}
        )
        return response.status_code == 200
    except:
        return False
```

### Database Connection Problems

```python
# Test database connectivity
def test_db_connection(config):
    try:
        conn = mysql.connector.connect(**config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        return result is not None
    except Error as e:
        st.error(f"Database error: {e}")
        return False
```

### Memory Issues with Large Datasets

```python
# Process data in chunks
def etl_in_batches(api_key, db_config, total_records=5000, batch_size=500):
    conn = get_db_connection(db_config)
    
    for offset in range(0, total_records, batch_size):
        artifacts = extract_all_artifacts(api_key, batch_size)
        metadata_df = transform_metadata(artifacts)
        load_metadata(metadata_df, conn)
        
        # Clear memory
        del artifacts, metadata_df
    
    conn.close()
```

This skill provides complete coverage for building ETL pipelines and analytics dashboards using the Harvard Art Museums API, from data extraction through visualization with Streamlit.
