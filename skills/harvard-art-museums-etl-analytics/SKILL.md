---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit and SQL
triggers:
  - build etl pipeline for harvard art museums
  - create analytics dashboard with streamlit and sql
  - extract harvard museum artifacts data
  - set up harvard art api data engineering project
  - visualize museum collection data with plotly
  - design sql schema for artifact metadata
  - implement batch data ingestion pipeline
  - query and analyze art museums database
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering solution that demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization. It extracts artifact data from the Harvard Art Museums API, transforms nested JSON into relational tables, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through Streamlit dashboards with Plotly visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

### Dependencies

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
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### API Key Setup

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Obtain your API key
3. Store securely in environment variables

## Database Schema

The project uses three main tables with relational design:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(200),
    copyright VARCHAR(1000),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## Core ETL Pipeline

### Extract: API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
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
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

def collect_all_artifacts(max_pages=10):
    """
    Collect artifacts across multiple pages with rate limiting
    """
    import time
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(0.5)  # Rate limiting
        else:
            break
    
    return all_artifacts
```

### Transform: Data Processing

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Metadata extraction
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'copyright': artifact.get('copyright'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Media extraction
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri') if artifact.get('images') else None,
            'imagepermissionlevel': artifact.get('imagepermissionlevel')
        }
        media_list.append(media)
        
        # Colors extraction
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Database Insertion

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
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

def batch_insert_metadata(df, connection):
    """
    Batch insert artifact metadata with duplicate handling
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT IGNORE INTO artifactmetadata 
    (id, title, culture, classification, century, dated, department, 
     division, medium, technique, period, copyright, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    records = df.values.tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()

def batch_insert_media(df, connection):
    """
    Batch insert artifact media data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri, imagepermissionlevel)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.values.tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()

def batch_insert_colors(df, connection):
    """
    Batch insert artifact color data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.values.tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")
    cursor.close()
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "ETL Pipeline":
        etl_pipeline_page()
    elif page == "SQL Analytics":
        sql_analytics_page()
    elif page == "Visualizations":
        visualizations_page()

def data_collection_page():
    st.header("📥 Data Collection from API")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_all_artifacts(max_pages=num_pages)
            st.success(f"Collected {len(artifacts)} artifacts")
            st.session_state['raw_artifacts'] = artifacts
            
            # Preview
            st.subheader("Sample Data")
            st.json(artifacts[0] if artifacts else {})

def etl_pipeline_page():
    st.header("🔄 ETL Pipeline Execution")
    
    if 'raw_artifacts' not in st.session_state:
        st.warning("Please collect data first")
        return
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(
                st.session_state['raw_artifacts']
            )
            
            st.success("Transformation complete")
            
            col1, col2, col3 = st.columns(3)
            col1.metric("Metadata Records", len(metadata_df))
            col2.metric("Media Records", len(media_df))
            col3.metric("Color Records", len(colors_df))
            
            # Load to database
            connection = create_db_connection()
            if connection:
                batch_insert_metadata(metadata_df, connection)
                batch_insert_media(media_df, connection)
                batch_insert_colors(colors_df, connection)
                connection.close()
                st.success("✅ Data loaded to database")
```

### SQL Analytics Queries

```python
def sql_analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    # Predefined analytical queries
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
            GROUP BY culture 
            ORDER BY count DESC 
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
            SELECT department, COUNT(*) as artifact_count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY artifact_count DESC
        """,
        "Most Common Colors": """
            SELECT color, COUNT(*) as frequency, 
                   AVG(percent) as avg_percent 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 10
        """,
        "Image Availability Rate": """
            SELECT 
                COUNT(CASE WHEN primaryimageurl IS NOT NULL THEN 1 END) * 100.0 / COUNT(*) as image_rate
            FROM artifactmedia
        """,
        "Classification Analysis": """
            SELECT classification, COUNT(*) as count,
                   COUNT(DISTINCT culture) as culture_diversity
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = create_db_connection()
        if connection:
            df_result = pd.read_sql(queries[selected_query], connection)
            connection.close()
            
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result, 
                    x=df_result.columns[0], 
                    y=df_result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Advanced Analytics Patterns

### Time-Series Analysis

```python
def analyze_temporal_patterns():
    """
    Analyze artifact acquisition patterns over centuries
    """
    query = """
    SELECT 
        century,
        COUNT(*) as artifacts,
        COUNT(DISTINCT classification) as classifications,
        COUNT(DISTINCT culture) as cultures
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY century
    """
    
    connection = create_db_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

### Color Spectrum Analysis

```python
def analyze_color_distribution():
    """
    Analyze color usage across different artifact types
    """
    query = """
    SELECT 
        m.classification,
        c.spectrum,
        COUNT(DISTINCT c.artifact_id) as artifact_count,
        AVG(c.percent) as avg_coverage
    FROM artifactcolors c
    JOIN artifactmetadata m ON c.artifact_id = m.id
    WHERE m.classification IS NOT NULL
    GROUP BY m.classification, c.spectrum
    ORDER BY artifact_count DESC
    """
    
    connection = create_db_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

## Common Patterns

### Incremental Data Updates

```python
def incremental_update(last_update_timestamp):
    """
    Fetch only new artifacts since last update
    """
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'sort': 'lastupdate',
        'sortorder': 'desc',
        'updatedafter': last_update_timestamp
    }
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    
    return response.json()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def safe_etl_execution():
    """
    ETL execution with comprehensive error handling
    """
    try:
        logging.info("Starting ETL pipeline")
        artifacts = collect_all_artifacts()
        
        logging.info(f"Collected {len(artifacts)} artifacts")
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        connection = create_db_connection()
        if not connection:
            raise Exception("Database connection failed")
        
        batch_insert_metadata(metadata_df, connection)
        batch_insert_media(media_df, connection)
        batch_insert_colors(colors_df, connection)
        
        connection.close()
        logging.info("ETL pipeline completed successfully")
        
    except Exception as e:
        logging.error(f"ETL pipeline failed: {e}")
        raise
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limiting errors:

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """
    Decorator to enforce rate limiting
    """
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = 1.0 / calls_per_second - elapsed
            
            if wait_time > 0:
                time.sleep(wait_time)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limit(calls_per_second=1)
def fetch_artifacts_safe(page=1, size=100):
    return fetch_artifacts(page, size)
```

### Database Connection Issues

```python
def robust_db_connection(max_retries=3):
    """
    Create database connection with retry logic
    """
    for attempt in range(max_retries):
        try:
            connection = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                port=os.getenv('DB_PORT', 3306),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return connection
        except Error as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise Exception(f"Failed to connect after {max_retries} attempts: {e}")
```

### Memory Management for Large Datasets

```python
def chunked_data_processing(artifacts, chunk_size=100):
    """
    Process large datasets in chunks to avoid memory issues
    """
    connection = create_db_connection()
    
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        
        batch_insert_metadata(metadata_df, connection)
        batch_insert_media(media_df, connection)
        batch_insert_colors(colors_df, connection)
        
        logging.info(f"Processed chunk {i//chunk_size + 1}")
    
    connection.close()
```

## Performance Optimization

### Bulk Insert Optimization

```python
def optimized_bulk_insert(df, table_name, connection):
    """
    Use LOAD DATA INFILE for faster bulk inserts
    """
    import tempfile
    
    with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.csv') as f:
        df.to_csv(f.name, index=False, header=False)
        temp_file = f.name
    
    cursor = connection.cursor()
    cursor.execute(f"""
        LOAD DATA LOCAL INFILE '{temp_file}'
        INTO TABLE {table_name}
        FIELDS TERMINATED BY ','
        ENCLOSED BY '"'
        LINES TERMINATED BY '\n'
    """)
    connection.commit()
    cursor.close()
    
    os.unlink(temp_file)
```

This skill provides comprehensive coverage of building ETL pipelines and analytics dashboards using the Harvard Art Museums API, with practical code examples for data extraction, transformation, loading, and visualization.
