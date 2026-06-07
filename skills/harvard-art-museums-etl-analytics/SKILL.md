---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - create a data engineering pipeline with Harvard art data
  - build analytics dashboard for museum artifacts
  - extract and transform Harvard museum API data
  - set up SQL database for art museum collections
  - visualize Harvard Art Museums data with Streamlit
  - implement museum artifact data pipeline
  - analyze art collection data with SQL queries
---

# Harvard Art Museums ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end ETL pipelines and analytics applications using the Harvard Art Museums API. The project demonstrates extracting artifact data, transforming it into relational structures, loading into SQL databases, and creating interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact data into relational SQL tables
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **Analytics Queries**: 20+ predefined SQL queries for artifact analysis
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your API key from: https://harvardartmuseums.org/collections/api

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(200),
    contact VARCHAR(200),
    description TEXT,
    provenance TEXT,
    commentary TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    image_count INT,
    video_count INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

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
def fetch_all_artifacts(total_records=1000, batch_size=100):
    """Fetch artifacts across multiple pages"""
    all_artifacts = []
    pages_needed = (total_records // batch_size) + 1
    
    for page in range(1, pages_needed + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=batch_size)
        all_artifacts.extend(data['records'])
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:total_records]
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifact_data(raw_artifacts: List[Dict]) -> tuple:
    """Transform raw API data into relational structure"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'contact': artifact.get('contact'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'commentary': artifact.get('commentary')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri') if artifact.get('images') else None,
            'image_count': len(artifact.get('images', [])),
            'video_count': len(artifact.get('videos', []))
        }
        media_records.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'color_percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, dated, classification, 
         department, technique, medium, dimensions, creditline, 
         division, contact, description, provenance, commentary)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    for _, row in metadata_df.iterrows():
        cursor.execute(metadata_query, tuple(row))
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri, image_count, video_count)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    for _, row in media_df.iterrows():
        cursor.execute(media_query, tuple(row))
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_name, color_percent)
            VALUES (%s, %s, %s, %s)
        """
        
        for _, row in colors_df.iterrows():
            cursor.execute(colors_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

### 3. SQL Analytics Queries

```python
# Common analytical queries for the dashboard

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "top_colors": """
        SELECT color_name, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "media_availability": """
        SELECT 
            SUM(CASE WHEN image_count > 0 THEN 1 ELSE 0 END) as with_images,
            SUM(CASE WHEN video_count > 0 THEN 1 ELSE 0 END) as with_videos,
            COUNT(*) as total_artifacts
        FROM artifactmedia
    """,
    
    "artifacts_by_classification": """
        SELECT classification, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY artifact_count DESC
        LIMIT 10
    """
}

def execute_analytics_query(query_name: str) -> pd.DataFrame:
    """Execute a predefined analytics query"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    st.sidebar.header("Navigation")
    page = st.sidebar.radio("Select Page", [
        "ETL Pipeline",
        "SQL Analytics",
        "Data Visualization"
    ])
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualization_page()

def show_etl_page():
    st.header("ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input("Number of records to fetch", 
                                      min_value=10, 
                                      max_value=10000, 
                                      value=100)
    
    with col2:
        batch_size = st.number_input("Batch size", 
                                     min_value=10, 
                                     max_value=100, 
                                     value=50)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts from API..."):
            artifacts = fetch_all_artifacts(num_records, batch_size)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded to database!")
        
        st.balloons()

def show_analytics_page():
    st.header("SQL Analytics")
    
    query_choice = st.selectbox(
        "Select Analytics Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            result_df = execute_analytics_query(query_choice)
            
            st.subheader("Query Results")
            st.dataframe(result_df)
            
            # Auto-generate visualization
            if len(result_df.columns) == 2:
                fig = px.bar(result_df, 
                            x=result_df.columns[0], 
                            y=result_df.columns[1],
                            title=f"{query_choice.replace('_', ' ').title()}")
                st.plotly_chart(fig, use_container_width=True)

def show_visualization_page():
    st.header("Data Visualizations")
    
    # Culture distribution
    st.subheader("Artifacts by Culture")
    culture_data = execute_analytics_query("artifacts_by_culture")
    fig1 = px.bar(culture_data, x='culture', y='artifact_count',
                  title="Top 15 Cultures by Artifact Count")
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    st.subheader("Most Common Colors")
    color_data = execute_analytics_query("top_colors")
    fig2 = px.scatter(color_data, x='usage_count', y='avg_percent',
                     size='usage_count', color='color_name',
                     title="Color Usage and Distribution")
    st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the last artifact ID loaded"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    last_id = cursor.fetchone()[0]
    conn.close()
    return last_id or 0

def fetch_new_artifacts_only():
    """Fetch only new artifacts since last load"""
    last_id = get_last_artifact_id()
    # Implement pagination starting from last_id
    pass
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(num_records):
    """ETL pipeline with error handling"""
    try:
        logger.info(f"Starting ETL for {num_records} records")
        artifacts = fetch_all_artifacts(num_records)
        
        metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
        logger.info("Transformation complete")
        
        load_to_database(metadata_df, media_df, colors_df)
        logger.info("Load complete")
        
        return True
    except Exception as e:
        logger.error(f"ETL failed: {str(e)}")
        return False
```

## Troubleshooting

### API Rate Limiting

If you encounter 429 errors:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    """Create session with automatic retries"""
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues

```python
def test_database_connection():
    """Test database connectivity"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        conn.close()
        return True
    except Exception as e:
        print(f"Connection failed: {e}")
        return False
```

### Memory Optimization for Large Datasets

```python
def batch_load_artifacts(total_records, batch_size=1000):
    """Load artifacts in batches to manage memory"""
    num_batches = (total_records // batch_size) + 1
    
    for batch_num in range(num_batches):
        start_idx = batch_num * batch_size
        batch_artifacts = fetch_all_artifacts(
            total_records=min(batch_size, total_records - start_idx),
            batch_size=100
        )
        
        metadata_df, media_df, colors_df = transform_artifact_data(batch_artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        
        # Clear memory
        del batch_artifacts, metadata_df, media_df, colors_df
```

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(filters: dict) -> str:
    """Build custom SQL query from filters"""
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    if filters.get('department'):
        base_query += f" AND department = '{filters['department']}'"
    
    return base_query
```

This skill provides comprehensive coverage for building ETL pipelines and analytics applications with the Harvard Art Museums API.
