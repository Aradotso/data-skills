---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, Streamlit, and SQL
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard museum API data
  - set up SQL database for art collection analytics
  - visualize museum artifact data with Streamlit
  - query Harvard Art Museums API and store in database
  - analyze art collection metadata with Python
  - design data warehouse for museum collections
---

# Harvard Art Museums ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Pre-built analytical queries for insights on cultures, centuries, departments, and media
- **Visualization Dashboard**: Interactive Streamlit app with Plotly charts for data exploration

**Architecture Flow**: `API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store credentials securely:

```python
# config.py or .env
HARVARD_API_KEY = os.getenv('HARVARD_API_KEY')
HARVARD_API_BASE_URL = "https://api.harvardartmuseums.org/object"

# Database configuration
DB_HOST = os.getenv('DB_HOST', 'localhost')
DB_USER = os.getenv('DB_USER', 'your_user')
DB_PASSWORD = os.getenv('DB_PASSWORD')
DB_NAME = os.getenv('DB_NAME', 'harvard_artifacts')
```

### Database Schema

Create the following tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    url VARCHAR(500),
    creditline TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percent FLOAT,
    spectrum VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    url = f"https://api.harvardartmuseums.org/object"
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
        print(f"Error: {response.status_code}")
        return None

def collect_all_artifacts(api_key, max_pages=10):
    """Collect artifacts across multiple pages with rate limiting"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            print(f"Collected page {page}: {len(data['records'])} artifacts")
            time.sleep(1)  # Rate limiting
        else:
            break
    
    return all_artifacts
```

### 2. ETL Transform Functions

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact JSON to metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline')
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Extract media/image data from artifacts"""
    media_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_data.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri')
            })
    
    return pd.DataFrame(media_data)

def transform_colors(artifacts):
    """Extract color data from artifacts"""
    color_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'percent': color.get('percent'),
                'spectrum': color.get('spectrum')
            })
    
    return pd.DataFrame(color_data)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return conn
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df_metadata):
    """Batch insert metadata into database"""
    conn = get_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    sql = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, technique, dated, url, creditline)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    values = df_metadata.values.tolist()
    
    try:
        cursor.executemany(sql, values)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error loading metadata: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()

def load_media(df_media):
    """Batch insert media data"""
    conn = get_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    sql = """
    INSERT INTO artifactmedia (artifact_id, media_type, baseimageurl, iiifbaseuri)
    VALUES (%s, %s, %s, %s)
    """
    
    values = df_media.values.tolist()
    
    try:
        cursor.executemany(sql, values)
        conn.commit()
        print(f"Inserted {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error loading media: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts
        FROM artifactmedia am
    """,
    
    "Artifacts by Classification": """
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
    conn = get_db_connection()
    if not conn:
        return None
    
    try:
        query = ANALYTICS_QUERIES[query_name]
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        print(f"Query error: {e}")
        return None
    finally:
        conn.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "Analytics Queries", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "Analytics Queries":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_etl_page():
    st.header("ETL Pipeline Control")
    
    api_key = st.text_input("Harvard API Key", type="password")
    max_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = collect_all_artifacts(api_key, max_pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_metadata(artifacts)
            df_media = transform_media(artifacts)
            df_colors = transform_colors(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_metadata(df_metadata)
            load_media(df_media)
            load_colors(df_colors)
            st.success("Data loaded successfully!")

def show_analytics_page():
    st.header("SQL Analytics Queries")
    
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        df_result = execute_query(query_name)
        
        if df_result is not None:
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) == 2:
                fig = px.bar(
                    df_result, 
                    x=df_result.columns[0], 
                    y=df_result.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations_page():
    st.header("Interactive Visualizations")
    
    # Culture distribution
    df_culture = execute_query("Artifacts by Culture")
    if df_culture is not None:
        fig = px.bar(df_culture, x='culture', y='artifact_count', 
                     title="Top 20 Cultures by Artifact Count")
        st.plotly_chart(fig, use_container_width=True)
    
    # Century timeline
    df_century = execute_query("Artifacts by Century")
    if df_century is not None:
        fig = px.line(df_century, x='century', y='artifact_count',
                      title="Artifacts Across Centuries")
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Workflow

```python
import os

def run_full_etl_pipeline():
    """Complete ETL pipeline execution"""
    # 1. Extract
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = collect_all_artifacts(api_key, max_pages=10)
    
    # 2. Transform
    df_metadata = transform_metadata(artifacts)
    df_media = transform_media(artifacts)
    df_colors = transform_colors(artifacts)
    
    # 3. Load
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL pipeline completed successfully")

# Run the pipeline
if __name__ == "__main__":
    run_full_etl_pipeline()
```

### Incremental Updates

```python
def get_latest_artifact_id():
    """Get the most recent artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key):
    """Fetch only new artifacts since last update"""
    latest_id = get_latest_artifact_id()
    # Implement logic to fetch artifacts with ID > latest_id
    # Harvard API supports filtering by ID range
```

## Troubleshooting

### API Rate Limiting
- Add `time.sleep(1)` between requests
- Use smaller page sizes (size=50 instead of 100)
- Monitor response headers for rate limit info

### Database Connection Issues
```python
# Test connection
conn = get_db_connection()
if conn and conn.is_connected():
    print("Database connected successfully")
else:
    print("Check DB_HOST, DB_USER, DB_PASSWORD, DB_NAME environment variables")
```

### Memory Issues with Large Datasets
```python
# Process in chunks
def process_in_batches(artifacts, batch_size=500):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        df_batch = transform_metadata(batch)
        load_metadata(df_batch)
```

### Streamlit Performance
- Cache database queries: `@st.cache_data`
- Limit result sets in SQL queries
- Use connection pooling for database

```python
@st.cache_data(ttl=3600)
def cached_query(query_name):
    return execute_query(query_name)
```

## Running the App

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="your_user"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```
