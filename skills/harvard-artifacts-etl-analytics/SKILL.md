---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums data
  - set up Harvard artifacts collection data engineering app
  - create analytics dashboard for museum API data
  - fetch and analyze Harvard Art Museums collection
  - build Streamlit app for artifact data visualization
  - implement SQL analytics for museum collection data
  - design ETL pipeline for Harvard API with Python
  - query and visualize Harvard artifacts database
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualization.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** data into SQL databases (MySQL/TiDB Cloud)
- **Analyzes** collections using 20+ predefined SQL queries
- **Visualizes** insights through interactive Streamlit dashboards with Plotly

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

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store credentials securely:

```python
# config.py
import os

HARVARD_API_KEY = os.getenv('HARVARD_API_KEY')
HARVARD_API_BASE_URL = 'https://api.harvardartmuseums.org/object'

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

### Environment Variables

```bash
export HARVARD_API_KEY='your_api_key_here'
export DB_HOST='your_db_host'
export DB_USER='your_db_user'
export DB_PASSWORD='your_db_password'
export DB_NAME='harvard_artifacts'
```

## Database Schema

### Table Structures

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    thumbnail_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Extract artifact data from Harvard API with pagination
    """
    base_url = 'https://api.harvardartmuseums.org/object'
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{num_pages}")
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return artifacts
```

### Transform: Data Normalization

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
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
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl'),
                'thumbnail_url': image.get('iiifbaseuri')
            }
            media_list.append(media)
        
        # Extract color data
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### Load: Batch Insert to SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """
    Load dataframes into MySQL database with batch inserts
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, dated, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, thumbnail_url)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(colors_query, df_colors.values.tolist())
        
        connection.commit()
        print(f"Inserted {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
ANALYTICAL_QUERIES = {
    'artifacts_by_culture': """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    'artifacts_by_century': """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    'color_distribution': """
        SELECT color, COUNT(*) as frequency, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    'media_availability': """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as with_images,
            (SELECT COUNT(*) FROM artifactmetadata) as total,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmedia m
    """,
    
    'department_breakdown': """
        SELECT department, COUNT(*) as count,
               COUNT(DISTINCT classification) as unique_classifications
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(query, db_config):
    """Execute analytical query and return results"""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    connection.close()
    return pd.DataFrame(results)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Collection Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # ETL Section
    st.header("📊 Data Collection (ETL)")
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages", min_value=1, max_value=100, value=10)
    with col2:
        page_size = st.number_input("Page size", min_value=10, max_value=100, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_pages, page_size)
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            load_to_database(df_meta, df_media, df_colors, DB_CONFIG)
            st.success(f"Loaded {len(df_meta)} artifacts successfully!")
    
    # Analytics Section
    st.header("📈 Analytics Dashboard")
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        results_df = execute_query(ANALYTICAL_QUERIES[query_name], DB_CONFIG)
        
        # Display table
        st.dataframe(results_df)
        
        # Auto-generate visualization
        if len(results_df.columns) >= 2:
            fig = px.bar(results_df, 
                        x=results_df.columns[0], 
                        y=results_df.columns[1],
                        title=query_name.replace('_', ' ').title())
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
# Full pipeline execution
def run_complete_etl(api_key, db_config, num_pages=10):
    # Extract
    print("Step 1: Extracting data from API...")
    raw_data = fetch_artifacts(api_key, num_pages)
    
    # Transform
    print("Step 2: Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_data)
    
    # Load
    print("Step 3: Loading to database...")
    load_to_database(df_metadata, df_media, df_colors, db_config)
    
    print("ETL Complete!")
    return len(df_metadata)
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_run(api_key, db_config):
    try:
        artifacts = fetch_artifacts(api_key)
        logger.info(f"Fetched {len(artifacts)} artifacts")
        
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        logger.info("Transformation complete")
        
        load_to_database(df_meta, df_media, df_colors, db_config)
        logger.info("Data loaded successfully")
        
    except Exception as e:
        logger.error(f"ETL failed: {e}")
        raise
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run with custom port
streamlit run app.py --server.port 8080

# Run in headless mode (server deployment)
streamlit run app.py --server.headless true
```

## Troubleshooting

### API Rate Limiting
- Add delays between requests: `time.sleep(0.5)`
- Implement exponential backoff for retries
- Monitor API quota limits

### Database Connection Issues
```python
# Test database connection
def test_db_connection(db_config):
    try:
        conn = mysql.connector.connect(**db_config)
        if conn.is_connected():
            print("Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Memory Management for Large Datasets
```python
# Process in chunks
def fetch_artifacts_chunked(api_key, total_pages, chunk_size=5):
    for start_page in range(1, total_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size, total_pages + 1)
        chunk_data = fetch_artifacts(api_key, end_page - start_page, start_page)
        yield chunk_data
```

### Empty Result Sets
- Verify API key is valid
- Check `hasimage=1` filter isn't too restrictive
- Ensure database tables exist before loading
- Validate network connectivity to Harvard API
