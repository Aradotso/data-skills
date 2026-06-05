---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an art museum data pipeline
  - extract data from Harvard Art Museums API
  - create ETL pipeline with Streamlit dashboard
  - analyze Harvard artifacts with SQL
  - set up art collection analytics app
  - build museum data engineering project
  - create Harvard API data visualization
  - design artifact metadata ETL system
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates real-world ETL workflows, SQL database design, analytics query execution, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums API
- **ETL Pipeline**: Transforms nested JSON into relational database tables
- **SQL Analytics**: Executes 20+ predefined analytical queries
- **Interactive Visualization**: Streamlit dashboard with Plotly charts
- **Database Management**: MySQL/TiDB Cloud integration with proper schema design

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the application
streamlit run app.py
```

**Required Dependencies** (requirements.txt):
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

## Core Architecture

**Data Flow**: API → Extract → Transform → Load → SQL → Analytics → Visualization

### Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    object_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    image_url TEXT,
    base_image_url TEXT,
    thumbnail_url TEXT,
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    object_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percentage FLOAT,
    FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        return data['records'], data['info']
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return [], {}

# Usage
API_KEY = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(API_KEY, page=1, size=50)
print(f"Total records: {info.get('totalrecords', 0)}")
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'object_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'dimensions': artifact.get('dimensions', 'Unknown'),
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('id')
        primary_image = artifact.get('primaryimageurl', '')
        
        if primary_image:
            media = {
                'object_id': object_id,
                'image_url': primary_image,
                'base_image_url': artifact.get('baseimageurl', ''),
                'thumbnail_url': artifact.get('images', [{}])[0].get('thumbnailurl', '') if artifact.get('images') else ''
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color data from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'object_id': object_id,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('name', ''),
                'color_percentage': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata_to_db(df, connection):
    """
    Batch insert metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (object_id, title, culture, century, classification, department, dated, period, technique, dimensions, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    data = df.values.tolist()
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
    finally:
        cursor.close()

def load_media_to_db(df, connection):
    """
    Batch insert media data into database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (object_id, image_url, base_image_url, thumbnail_url)
    VALUES (%s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

### 4. SQL Analytics Queries

```python
# Example analytical queries

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "artifacts_with_images": """
        SELECT 
            COUNT(DISTINCT am.object_id) as with_images,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
            ROUND(COUNT(DISTINCT am.object_id) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmedia am
    """,
    
    "top_colors": """
        SELECT 
            color_name,
            COUNT(*) as usage_count,
            AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department != 'Unknown'
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_analytics_query(connection, query_name):
    """
    Execute a predefined analytics query
    """
    cursor = connection.cursor(dictionary=True)
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        return None
    
    try:
        cursor.execute(query)
        results = cursor.fetchall()
        return pd.DataFrame(results)
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        cursor.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # ETL Pipeline Section
    st.header("1. ETL Pipeline")
    
    num_records = st.number_input("Number of records to fetch", min_value=10, max_value=500, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts, info = fetch_artifacts(api_key, page=1, size=num_records)
            
        with st.spinner("Transforming data..."):
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            
        with st.spinner("Loading to database..."):
            conn = create_database_connection()
            load_metadata_to_db(metadata_df, conn)
            load_media_to_db(media_df, conn)
            conn.close()
            
        st.success(f"ETL Complete! Processed {len(artifacts)} artifacts")
    
    # Analytics Section
    st.header("2. SQL Analytics")
    
    query_selection = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        conn = create_database_connection()
        results_df = execute_analytics_query(conn, query_selection)
        conn.close()
        
        if results_df is not None and not results_df.empty:
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=f"Visualization: {query_selection}"
                )
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_complete_pipeline(api_key, num_pages=5, records_per_page=100):
    """
    Run full ETL pipeline with multiple pages
    """
    connection = create_database_connection()
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        artifacts, info = fetch_artifacts(api_key, page=page, size=records_per_page)
        
        if not artifacts:
            break
        
        # Transform
        metadata_df = transform_artifact_metadata(artifacts)
        media_df = transform_artifact_media(artifacts)
        colors_df = transform_artifact_colors(artifacts)
        
        # Load
        load_metadata_to_db(metadata_df, connection)
        load_media_to_db(media_df, connection)
        
        # Rate limiting
        import time
        time.sleep(1)
    
    connection.close()
    print("Pipeline complete!")
```

### Error Handling & Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key, page=1, size=100, max_retries=3):
    """
    Fetch artifacts with exponential backoff retry
    """
    for attempt in range(max_retries):
        try:
            artifacts, info = fetch_artifacts(api_key, page, size)
            return artifacts, info
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

## Troubleshooting

### API Rate Limiting

```python
# Add delays between requests
import time

for page in range(1, 10):
    artifacts, _ = fetch_artifacts(API_KEY, page=page)
    # Process artifacts...
    time.sleep(1)  # 1 second delay between requests
```

### Database Connection Issues

```python
# Test connection
def test_database_connection():
    try:
        conn = create_database_connection()
        if conn and conn.is_connected():
            print("✓ Database connection successful")
            conn.close()
            return True
        else:
            print("✗ Database connection failed")
            return False
    except Exception as e:
        print(f"✗ Connection error: {e}")
        return False
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default='Unknown'):
    """
    Safely extract values with defaults
    """
    value = dictionary.get(key, default)
    return value if value else default

# Usage in transformation
metadata = {
    'title': safe_get(artifact, 'title'),
    'culture': safe_get(artifact, 'culture'),
    'century': safe_get(artifact, 'century')
}
```

### Memory Management for Large Datasets

```python
def process_in_batches(api_key, total_pages, batch_size=5):
    """
    Process large datasets in batches to manage memory
    """
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        print(f"Processing batch: pages {batch_start}-{batch_end-1}")
        
        conn = create_database_connection()
        
        for page in range(batch_start, batch_end):
            artifacts, _ = fetch_artifacts(api_key, page=page)
            # Transform and load...
        
        conn.close()
        print(f"Batch complete. Memory cleared.")
```

This skill provides comprehensive guidance for building production-ready data engineering pipelines with the Harvard Art Museums API, including ETL best practices, SQL analytics, and interactive visualization dashboards.
