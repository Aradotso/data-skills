---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit visualizations
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - set up data engineering workflow for museum artifact data
  - create analytics dashboard for Harvard Art Museums collection
  - extract and transform Harvard museum API data to SQL
  - build Streamlit app for art museum data visualization
  - implement batch data ingestion from Harvard Art Museums
  - query and visualize artifact metadata with SQL
  - design relational database schema for museum artifacts
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts
- **Batch Processing**: Optimized batch inserts for large datasets

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
cat > .env << EOF
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
EOF
```

### Get Harvard Art Museums API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free for educational/personal use)
3. Add to `.env` file as `HARVARD_API_KEY`

## Configuration

### Environment Variables

```python
import os
from dotenv import load_dotenv

load_dotenv()

# API Configuration
API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = 'https://api.harvardartmuseums.org/object'

# Database Configuration
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

### Database Schema Setup

```python
import mysql.connector

def create_tables(connection):
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            technique VARCHAR(500),
            dated VARCHAR(255),
            url TEXT,
            accessionyear INT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url TEXT,
            media_type VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_name VARCHAR(100),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    try:
        response = requests.get(API_BASE_URL, params=params, timeout=30)
        response.raise_for_status()
        data = response.json()
        
        return {
            'records': data.get('records', []),
            'total_pages': data.get('info', {}).get('pages', 0),
            'total_records': data.get('info', {}).get('totalrecords', 0)
        }
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

def fetch_all_artifacts(api_key, max_pages=10):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        result = fetch_artifacts(api_key, page=page)
        
        if result and result['records']:
            all_artifacts.extend(result['records'])
            time.sleep(1)  # Rate limiting
        else:
            break
    
    return all_artifacts
```

### Transform: Clean and Structure Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """
    Transform artifact data into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:255],
            'period': artifact.get('period', 'Unknown')[:255],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:255],
            'department': artifact.get('department', 'Unknown')[:255],
            'division': artifact.get('division', 'Unknown')[:255],
            'technique': artifact.get('technique', 'Unknown')[:500],
            'dated': artifact.get('dated', 'Unknown')[:255],
            'url': artifact.get('url', ''),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """
    Extract media/image information
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl', ''),
                'media_type': 'image'
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """
    Extract color information from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('color', 'Unknown'),
                'percentage': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Batch Insert into Database

```python
def batch_insert_metadata(connection, df):
    """
    Batch insert artifact metadata
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, division, technique, dated, url, accessionyear)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    return cursor.rowcount

def batch_insert_media(connection, df):
    """
    Batch insert media information
    """
    if df.empty:
        return 0
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    return cursor.rowcount

def batch_insert_colors(connection, df):
    """
    Batch insert color data
    """
    if df.empty:
        return 0
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    return cursor.rowcount
```

## Complete ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, max_pages=5):
    """
    Complete ETL pipeline execution
    """
    # Extract
    print("Starting extraction...")
    artifacts = fetch_all_artifacts(api_key, max_pages=max_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Starting transformation...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Starting load to database...")
    connection = mysql.connector.connect(**db_config)
    
    try:
        create_tables(connection)
        
        metadata_rows = batch_insert_metadata(connection, metadata_df)
        media_rows = batch_insert_media(connection, media_df)
        color_rows = batch_insert_colors(connection, colors_df)
        
        print(f"Loaded {metadata_rows} metadata records")
        print(f"Loaded {media_rows} media records")
        print(f"Loaded {color_rows} color records")
        
    finally:
        connection.close()
    
    return {
        'artifacts': len(artifacts),
        'metadata_rows': metadata_rows,
        'media_rows': media_rows,
        'color_rows': color_rows
    }
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Top 10 cultures by artifact count
QUERY_CULTURES = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture != 'Unknown'
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
QUERY_CENTURIES = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != 'Unknown'
    GROUP BY century
    ORDER BY count DESC
"""

# Department distribution
QUERY_DEPARTMENTS = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    GROUP BY department
    ORDER BY count DESC
"""

# Most common colors
QUERY_COLORS = """
    SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_name
    ORDER BY frequency DESC
    LIMIT 15
"""

# Artifacts with most images
QUERY_MOST_IMAGES = """
    SELECT m.title, m.culture, COUNT(med.id) as image_count
    FROM artifactmetadata m
    JOIN artifactmedia med ON m.id = med.artifact_id
    GROUP BY m.id, m.title, m.culture
    ORDER BY image_count DESC
    LIMIT 10
"""

def execute_query(connection, query):
    """
    Execute SQL query and return DataFrame
    """
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    try:
        connection = mysql.connector.connect(**DB_CONFIG)
        st.sidebar.success("✅ Database Connected")
    except Exception as e:
        st.sidebar.error(f"❌ Database Error: {e}")
        return
    
    # Tab layout
    tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔄 ETL Pipeline", "📈 Custom Query"])
    
    with tab1:
        show_analytics(connection)
    
    with tab2:
        show_etl_interface()
    
    with tab3:
        show_custom_query(connection)
    
    connection.close()

def show_analytics(connection):
    """
    Display predefined analytics
    """
    st.header("Artifact Analytics")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Top Cultures")
        df_cultures = execute_query(connection, QUERY_CULTURES)
        fig = px.bar(df_cultures, x='culture', y='artifact_count',
                     title='Top 10 Cultures by Artifact Count')
        st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        st.subheader("Century Distribution")
        df_centuries = execute_query(connection, QUERY_CENTURIES)
        fig = px.pie(df_centuries, values='count', names='century',
                     title='Artifacts by Century')
        st.plotly_chart(fig, use_container_width=True)
    
    st.subheader("Color Analysis")
    df_colors = execute_query(connection, QUERY_COLORS)
    st.dataframe(df_colors, use_container_width=True)

def show_etl_interface():
    """
    ETL pipeline execution interface
    """
    st.header("ETL Pipeline Control")
    
    max_pages = st.slider("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            try:
                results = run_etl_pipeline(
                    os.getenv('HARVARD_API_KEY'),
                    DB_CONFIG,
                    max_pages=max_pages
                )
                
                st.success("✅ ETL Pipeline Completed!")
                st.json(results)
                
            except Exception as e:
                st.error(f"❌ ETL Error: {e}")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """
    Get the last loaded artifact ID for incremental updates
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts(api_key, last_id):
    """
    Fetch only artifacts newer than last_id
    """
    # Implement incremental fetch logic
    pass
```

### Pattern 2: Data Quality Validation

```python
def validate_artifact_data(df):
    """
    Validate data quality before loading
    """
    issues = []
    
    # Check for missing IDs
    if df['id'].isnull().any():
        issues.append("Missing artifact IDs detected")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append(f"{df['id'].duplicated().sum()} duplicate IDs found")
    
    # Check for missing titles
    missing_titles = df['title'].isnull().sum()
    if missing_titles > 0:
        issues.append(f"{missing_titles} artifacts missing titles")
    
    return issues
```

### Pattern 3: Error Handling and Logging

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('etl_pipeline.log'),
        logging.StreamHandler()
    ]
)

def safe_etl_execution(api_key, db_config, max_pages):
    """
    ETL with comprehensive error handling
    """
    try:
        logging.info(f"Starting ETL pipeline for {max_pages} pages")
        results = run_etl_pipeline(api_key, db_config, max_pages)
        logging.info(f"ETL completed successfully: {results}")
        return results
    
    except requests.exceptions.RequestException as e:
        logging.error(f"API request failed: {e}")
        raise
    
    except mysql.connector.Error as e:
        logging.error(f"Database error: {e}")
        raise
    
    except Exception as e:
        logging.error(f"Unexpected error: {e}")
        raise
```

## Troubleshooting

### API Issues

**Rate Limiting**: Harvard API has rate limits
```python
import time
from functools import wraps

def rate_limit(calls_per_second=1):
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            if wait_time > 0:
                time.sleep(wait_time)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=10)
def fetch_with_rate_limit(url, params):
    return requests.get(url, params=params)
```

**Invalid API Key**: Verify environment variable
```python
def validate_api_key(api_key):
    if not api_key:
        raise ValueError("HARVARD_API_KEY not set in environment")
    
    # Test API key
    response = requests.get(
        f"{API_BASE_URL}?apikey={api_key}&size=1"
    )
    
    if response.status_code == 401:
        raise ValueError("Invalid API key")
    
    return True
```

### Database Connection Issues

**Connection Timeout**:
```python
def get_connection_with_retry(db_config, max_retries=3):
    for attempt in range(max_retries):
        try:
            connection = mysql.connector.connect(
                **db_config,
                connect_timeout=30
            )
            return connection
        except mysql.connector.Error as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

**Foreign Key Constraints**: Ensure parent records exist before inserting child records
```python
def safe_batch_insert(connection, metadata_df, media_df, colors_df):
    """
    Insert in correct order to respect foreign keys
    """
    # 1. Insert metadata first (parent table)
    batch_insert_metadata(connection, metadata_df)
    
    # 2. Insert media (child table)
    batch_insert_media(connection, media_df)
    
    # 3. Insert colors (child table)
    batch_insert_colors(connection, colors_df)
```

### Streamlit Performance

**Large DataFrame Display**:
```python
# Use pagination for large results
def paginate_dataframe(df, page_size=50):
    page_num = st.number_input('Page', min_value=1, 
                               max_value=(len(df) // page_size) + 1)
    start_idx = (page_num - 1) * page_size
    end_idx = start_idx + page_size
    return df.iloc[start_idx:end_idx]
```

**Caching Database Queries**:
```python
@st.cache_data(ttl=3600)
def cached_query(_connection, query):
    """
    Cache query results for 1 hour
    Note: Use underscore prefix for unhashable connection object
    """
    return pd.read_sql(query, _connection)
```

This skill provides comprehensive guidance for building production-ready data engineering pipelines with the Harvard Art Museums API, including ETL workflows, SQL analytics, and interactive dashboards.
