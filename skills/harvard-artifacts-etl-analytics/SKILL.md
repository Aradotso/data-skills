---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create ETL workflow for museum artifacts data
  - analyze Harvard museum collection with SQL
  - set up artifact data engineering pipeline
  - visualize museum data with Streamlit
  - query Harvard Art Museums API and store in database
  - build analytics dashboard for art museum data
  - extract transform load museum artifacts
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:
- Fetches artifact data from Harvard Art Museums API with pagination
- Transforms nested JSON into relational database schema
- Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- Executes 20+ analytical SQL queries
- Visualizes results through interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Store credentials securely using environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Harvard API Key

Obtain an API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):
1. Register for free access
2. Generate API key
3. Store in environment variable or Streamlit secrets

### Database Setup

Create the required database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    technique VARCHAR(255),
    medium VARCHAR(255),
    dimensions VARCHAR(255),
    creditline TEXT,
    accession_number VARCHAR(100),
    provenance TEXT,
    copyright TEXT,
    image_count INT,
    total_page_views INT,
    total_unique_page_views INT,
    url VARCHAR(500),
    last_updated DATETIME
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url VARCHAR(500),
    primary_image_url VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Key API Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
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
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Total pages: {data['info']['totalpages']}")
```

### Rate-Limited Bulk Data Collection

```python
import time
from typing import List, Dict

def collect_all_artifacts(api_key: str, max_pages: int = 10) -> List[Dict]:
    """Collect artifacts across multiple pages with rate limiting"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Collected page {page}/{max_pages}: {len(artifacts)} artifacts")
            
            # Rate limiting: respect API limits
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            continue
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract: API to Raw Data

```python
import pandas as pd

def extract_artifact_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract metadata from artifact records"""
    metadata = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance'),
            'copyright': artifact.get('copyright'),
            'image_count': artifact.get('totalpageviews', 0),
            'total_page_views': artifact.get('totalpageviews', 0),
            'total_unique_page_views': artifact.get('totaluniquepageviews', 0),
            'url': artifact.get('url'),
            'last_updated': artifact.get('lastupdate')
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)
```

### Transform: Nested JSON to Relational

```python
def extract_artifact_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract media/image data from artifacts"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        primary_image = artifact.get('primaryimageurl')
        
        # Extract images from nested structure
        images = artifact.get('images', [])
        if images:
            for img in images:
                media_records.append({
                    'artifact_id': artifact_id,
                    'base_image_url': img.get('baseimageurl'),
                    'primary_image_url': primary_image
                })
        elif primary_image:
            # Add primary image if no images array
            media_records.append({
                'artifact_id': artifact_id,
                'base_image_url': None,
                'primary_image_url': primary_image
            })
    
    return pd.DataFrame(media_records)

def extract_artifact_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract color data from artifacts"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color_obj in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color_obj.get('color'),
                'spectrum': color_obj.get('spectrum'),
                'hue': color_obj.get('hue'),
                'percent': color_obj.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

### Load: Batch Insert to Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
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
        print(f"Error connecting to database: {e}")
        return None

def load_metadata(df: pd.DataFrame, connection):
    """Batch insert metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, dated, 
     description, technique, medium, dimensions, creditline, 
     accession_number, provenance, copyright, image_count, 
     total_page_views, total_unique_page_views, url, last_updated)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), last_updated=VALUES(last_updated)
    """
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def load_media(df: pd.DataFrame, connection):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, base_image_url, primary_image_url)
    VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
# Sample analytical queries for the dashboard

ANALYTICAL_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN m.primary_image_url IS NOT NULL THEN 'With Image' 
                 ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY image_status
    """,
    
    "popular_artifacts": """
        SELECT title, culture, total_page_views, total_unique_page_views
        FROM artifactmetadata
        WHERE total_page_views > 0
        ORDER BY total_page_views DESC
        LIMIT 20
    """
}
```

### Executing Queries in Streamlit

```python
import streamlit as st
import plotly.express as px

def run_analytics_query(query_name: str, connection):
    """Execute analytical query and return results"""
    query = ANALYTICAL_QUERIES.get(query_name)
    
    if not query:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Exception as e:
        st.error(f"Query execution failed: {e}")
        return None

def visualize_results(df: pd.DataFrame, query_name: str):
    """Create visualization based on query results"""
    if df is None or df.empty:
        st.warning("No data to visualize")
        return
    
    # Display data table
    st.dataframe(df)
    
    # Auto-generate chart based on columns
    if len(df.columns) >= 2:
        x_col = df.columns[0]
        y_col = df.columns[1]
        
        fig = px.bar(df, x=x_col, y=y_col, title=f"{query_name.replace('_', ' ').title()}")
        st.plotly_chart(fig, use_container_width=True)
```

## Streamlit Dashboard Structure

```python
import streamlit as st

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("Data Collection")
        num_pages = st.slider("Pages to fetch", 1, 50, 10)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Collecting artifacts..."):
                artifacts = collect_all_artifacts(api_key, max_pages=num_pages)
                
                # ETL Process
                conn = create_connection()
                if conn:
                    metadata_df = extract_artifact_metadata(artifacts)
                    media_df = extract_artifact_media(artifacts)
                    colors_df = extract_artifact_colors(artifacts)
                    
                    load_metadata(metadata_df, conn)
                    load_media(media_df, conn)
                    load_colors(colors_df, conn)
                    
                    st.success(f"Loaded {len(artifacts)} artifacts successfully!")
                    conn.close()
    
    # Main content area - Analytics
    st.header("📊 Analytics Dashboard")
    
    query_options = list(ANALYTICAL_QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        conn = create_connection()
        if conn:
            results = run_analytics_query(selected_query, conn)
            visualize_results(results, selected_query)
            conn.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_update_timestamp(connection):
    """Get the latest update timestamp from database"""
    cursor = connection.cursor()
    query = "SELECT MAX(last_updated) FROM artifactmetadata"
    cursor.execute(query)
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result else None

def fetch_updated_artifacts(api_key, last_update):
    """Fetch only artifacts updated since last sync"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'lastupdate',
        'sortorder': 'desc'
    }
    
    if last_update:
        params['after'] = last_update
    
    response = requests.get(base_url, params=params)
    return response.json() if response.status_code == 200 else None
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, connection, max_pages=10):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        logger.info(f"Starting data collection for {max_pages} pages")
        artifacts = collect_all_artifacts(api_key, max_pages)
        
        if not artifacts:
            logger.warning("No artifacts collected")
            return False
        
        # Transform
        logger.info(f"Transforming {len(artifacts)} artifacts")
        metadata_df = extract_artifact_metadata(artifacts)
        media_df = extract_artifact_media(artifacts)
        colors_df = extract_artifact_colors(artifacts)
        
        # Load
        logger.info("Loading data to database")
        load_metadata(metadata_df, connection)
        load_media(media_df, connection)
        load_colors(colors_df, connection)
        
        logger.info("ETL pipeline completed successfully")
        return True
        
    except Exception as e:
        logger.error(f"ETL pipeline failed: {e}")
        return False
```

## Troubleshooting

### API Rate Limiting
- Harvard API has rate limits; add `time.sleep()` between requests
- Use `size` parameter to fetch more records per request (max 100)
- Implement exponential backoff for failed requests

### Database Connection Issues
```python
# Test database connectivity
def test_connection():
    try:
        conn = create_connection()
        if conn and conn.is_connected():
            print("Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Handling NULL Values
```python
# Clean data before insertion
def clean_dataframe(df):
    """Replace None/NaN values for SQL compatibility"""
    return df.where(pd.notnull(df), None)
```

### Memory Management for Large Datasets
```python
def chunked_load(df, connection, chunk_size=1000):
    """Load large datasets in chunks"""
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        load_metadata(chunk, connection)
        print(f"Loaded chunk {i//chunk_size + 1}")
```

This skill provides a complete foundation for building data engineering pipelines with museum artifact data, from API extraction through analytics visualization.
