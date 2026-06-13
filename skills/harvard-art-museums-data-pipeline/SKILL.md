---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data pipelines using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - "build a data pipeline for Harvard Art Museums"
  - "create ETL pipeline with Harvard API"
  - "set up analytics dashboard with Streamlit"
  - "extract data from Harvard Art Museums API"
  - "build SQL analytics for museum artifacts"
  - "create interactive data visualization dashboard"
  - "implement batch data ingestion pipeline"
  - "analyze museum collection data with Python"
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL pipelines, relational database design, SQL analytics, and interactive dashboards using Streamlit and Plotly.

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

### Required Dependencies

```txt
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
mysql-connector-python>=8.0.0
plotly>=5.11.0
python-dotenv>=0.21.0
```

## Harvard Art Museums API Integration

### API Authentication

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org"

def get_objects(page=1, size=100):
    """Fetch artifacts with pagination support"""
    endpoint = f"{BASE_URL}/object"
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size
    }
    response = requests.get(endpoint, params=params)
    response.raise_for_status()
    return response.json()
```

### Pagination Handling

```python
def collect_all_artifacts(max_records=1000):
    """Collect artifacts with rate limiting and pagination"""
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < max_records:
        data = get_objects(page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
        
        # Rate limiting
        time.sleep(0.5)
    
    return artifacts[:max_records]
```

## ETL Pipeline Implementation

### Extract Phase

```python
import requests
import pandas as pd

def extract_artifact_data(artifact_id=None, limit=100):
    """Extract artifact data from Harvard API"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': limit
    }
    
    if artifact_id:
        url = f"{BASE_URL}/object/{artifact_id}"
    else:
        url = f"{BASE_URL}/object"
    
    response = requests.get(url, params=params)
    data = response.json()
    
    return data.get('records', []) if 'records' in data else [data]
```

### Transform Phase

```python
def transform_artifacts(raw_data):
    """Transform nested JSON into relational tables"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'object_id': artifact.get('objectid'),
            'title': artifact.get('title'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'century': artifact.get('century')
        }
        metadata_records.append(metadata)
        
        # Extract media
        for img in artifact.get('images', []):
            media = {
                'object_id': artifact.get('objectid'),
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            }
            media_records.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_record = {
                'object_id': artifact.get('objectid'),
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

### Load Phase

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(dataframes):
    """Batch insert data into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (object_id, title, dated, period, culture, classification, 
         medium, dimensions, department, division, century)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = dataframes['metadata'].values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia 
        (object_id, image_id, base_url, width, height, format)
        VALUES (%s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE base_url=VALUES(base_url)
        """
        media_values = dataframes['media'].values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Load colors
        color_query = """
        INSERT INTO artifactcolors 
        (object_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        color_values = dataframes['colors'].values.tolist()
        cursor.executemany(color_query, color_values)
        
        conn.commit()
        print(f"Loaded {len(metadata_values)} artifacts successfully")
        
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    object_id INT PRIMARY KEY,
    title VARCHAR(500),
    dated VARCHAR(100),
    period VARCHAR(200),
    culture VARCHAR(200),
    classification VARCHAR(200),
    medium TEXT,
    dimensions VARCHAR(500),
    department VARCHAR(200),
    division VARCHAR(200),
    century VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    image_id INT,
    base_url VARCHAR(500),
    width INT,
    height INT,
    format VARCHAR(50),
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);
```

## SQL Analytics Queries

### Common Analytical Queries

```python
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN image_id IS NOT NULL THEN 'With Images' 
                 ELSE 'No Images' END as status,
            COUNT(DISTINCT object_id) as count
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
    
    "artifacts_with_dimensions": """
        SELECT COUNT(*) as total,
               SUM(CASE WHEN dimensions IS NOT NULL THEN 1 ELSE 0 END) as with_dimensions,
               ROUND(100.0 * SUM(CASE WHEN dimensions IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*), 2) as percentage
        FROM artifactmetadata
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    query = ANALYTICS_QUERIES.get(query_name)
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        analytics_page()
    else:
        visualization_page()

def data_collection_page():
    st.header("📥 Data Collection")
    
    num_records = st.number_input("Number of records to fetch", 10, 1000, 100)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching data from Harvard API..."):
            raw_data = collect_all_artifacts(max_records=num_records)
            transformed_data = transform_artifacts(raw_data)
            load_to_database(transformed_data)
            
            st.success(f"Successfully loaded {num_records} artifacts!")
            st.dataframe(transformed_data['metadata'].head())

def analytics_page():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        df = execute_query(query_name)
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

def visualization_page():
    st.header("📈 Interactive Visualizations")
    
    # Culture distribution
    df_culture = execute_query("artifacts_by_culture")
    fig1 = px.bar(df_culture, x='culture', y='count', 
                  title='Top 10 Cultures by Artifact Count')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    df_colors = execute_query("top_colors")
    fig2 = px.pie(df_colors, values='frequency', names='color',
                  title='Color Distribution in Artifacts')
    st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_records=100):
    """Complete ETL pipeline execution"""
    print("Starting ETL pipeline...")
    
    # Extract
    print(f"Extracting {num_records} records...")
    raw_data = collect_all_artifacts(max_records=num_records)
    
    # Transform
    print("Transforming data...")
    transformed = transform_artifacts(raw_data)
    
    # Load
    print("Loading to database...")
    load_to_database(transformed)
    
    print("ETL pipeline completed successfully!")
    return transformed
```

### Error Handling

```python
def safe_api_call(url, params, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting
- Add `time.sleep()` between requests
- Use batch processing with smaller page sizes
- Implement exponential backoff for retries

### Database Connection Issues
- Verify environment variables are set correctly
- Check firewall rules for database host
- Ensure database user has INSERT/UPDATE privileges

### Memory Issues with Large Datasets
- Process data in batches instead of loading all at once
- Use chunked DataFrame operations
- Implement streaming inserts

### Streamlit Performance
- Cache expensive operations with `@st.cache_data`
- Limit initial data loads
- Use pagination for large result sets

```python
@st.cache_data(ttl=3600)
def cached_query(query_name):
    """Cache query results for 1 hour"""
    return execute_query(query_name)
```
