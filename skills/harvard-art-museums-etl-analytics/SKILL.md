---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifacts
  - extract and transform Harvard API data
  - set up data engineering pipeline for art collection
  - query Harvard Art Museums database
  - visualize museum artifact analytics
  - integrate Harvard Art Museums API
  - create museum data warehouse
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build production-ready data engineering pipelines that extract artifact data from the Harvard Art Museums API, transform it into relational structures, load it into SQL databases, and create interactive analytics dashboards using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates real-world data engineering patterns:

- **API Integration**: Paginated data extraction from Harvard Art Museums API with rate limiting
- **ETL Pipeline**: Transform nested JSON into normalized relational tables
- **Database Design**: Three-table schema (metadata, media, colors) with foreign key relationships
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. API Key Setup

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or configure environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

The project supports MySQL or TiDB Cloud. Set up your database connection:

```python
# config.py or in your main application
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

### 3. Database Schema Setup

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    period VARCHAR(255),
    url TEXT,
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    media_type VARCHAR(100),
    image_url TEXT,
    PRIMARY KEY (artifact_id, media_id),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    PRIMARY KEY (artifact_id, color_hex),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        page: Page number (starts at 1)
        size: Number of results per page (max 100)
    
    Returns:
        dict: API response with records and pagination info
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()


def collect_all_artifacts(api_key, max_records=1000):
    """
    Collect multiple pages of artifacts
    
    Args:
        api_key: Harvard API key
        max_records: Maximum number of records to collect
    
    Returns:
        list: All collected artifact records
    """
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=size)
        
        records = data.get('records', [])
        if not records:
            break
            
        all_records.extend(records)
        
        # Check if more pages exist
        if len(all_records) >= data.get('info', {}).get('totalrecords', 0):
            break
            
        page += 1
    
    return all_records[:max_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(records):
    """
    Transform raw API records into metadata DataFrame
    
    Args:
        records: List of artifact records from API
    
    Returns:
        pd.DataFrame: Normalized metadata
    """
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title', 'Unknown')[:500],
            'culture': record.get('culture', 'Unknown')[:255],
            'century': record.get('century', 'Unknown')[:255],
            'classification': record.get('classification', 'Unknown')[:255],
            'department': record.get('department', 'Unknown')[:255],
            'division': record.get('division', 'Unknown')[:255],
            'technique': record.get('technique', 'Unknown')[:500],
            'medium': record.get('medium', 'Unknown')[:500],
            'dated': record.get('dated', 'Unknown')[:255],
            'period': record.get('period', 'Unknown')[:255],
            'url': record.get('url', '')
        })
    
    return pd.DataFrame(metadata)


def transform_artifact_media(records):
    """
    Extract media/image data from artifacts
    
    Args:
        records: List of artifact records from API
    
    Returns:
        pd.DataFrame: Media data with artifact_id foreign key
    """
    media_data = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for img in images:
            media_data.append({
                'artifact_id': artifact_id,
                'media_id': img.get('imageid'),
                'media_type': img.get('format', 'image'),
                'image_url': img.get('baseimageurl', '')
            })
    
    return pd.DataFrame(media_data)


def transform_artifact_colors(records):
    """
    Extract color data from artifacts
    
    Args:
        records: List of artifact records from API
    
    Returns:
        pd.DataFrame: Color distribution data
    """
    color_data = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', '#000000'),
                'color_percent': float(color.get('percent', 0))
            })
    
    return pd.DataFrame(color_data)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df, table_name, db_config):
    """
    Batch insert DataFrame into SQL database
    
    Args:
        df: pandas DataFrame to insert
        table_name: Target table name
        db_config: Database connection configuration
    
    Returns:
        bool: Success status
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Prepare batch insert
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        # Convert DataFrame to list of tuples
        data_tuples = [tuple(row) for row in df.values]
        
        # Batch insert
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        
        print(f"Inserted {cursor.rowcount} rows into {table_name}")
        return True
        
    except Error as e:
        print(f"Error loading data to {table_name}: {e}")
        return False
        
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()


def run_etl_pipeline(api_key, db_config, max_records=500):
    """
    Complete ETL pipeline execution
    
    Args:
        api_key: Harvard API key
        db_config: Database configuration
        max_records: Number of records to process
    """
    # Extract
    print("Extracting data from API...")
    records = collect_all_artifacts(api_key, max_records)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(records)
    df_media = transform_artifact_media(records)
    df_colors = transform_artifact_colors(records)
    
    # Load
    print("Loading to database...")
    load_to_database(df_metadata, 'artifactmetadata', db_config)
    load_to_database(df_media, 'artifactmedia', db_config)
    load_to_database(df_colors, 'artifactcolors', db_config)
    
    print("ETL pipeline completed successfully!")
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Media' ELSE 'No Media' END as media_status,
            COUNT(*) as artifact_count
        FROM (
            SELECT m.id, COUNT(a.media_id) as media_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia a ON m.id = a.artifact_id
            GROUP BY m.id
        ) as media_summary
        GROUP BY media_status
    """,
    
    "top_colors": """
        SELECT color_hex, COUNT(DISTINCT artifact_id) as artifact_count,
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department != 'Unknown'
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}


def execute_query(query, db_config):
    """
    Execute SQL query and return results as DataFrame
    
    Args:
        query: SQL query string
        db_config: Database configuration
    
    Returns:
        pd.DataFrame: Query results
    """
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        if connection.is_connected():
            connection.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """
    Main Streamlit dashboard application
    """
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("⚙️ Configuration")
        
        # API Key input
        api_key = st.text_input("Harvard API Key", 
                                type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        # Database connection
        db_config = {
            'host': st.text_input("DB Host", value=os.getenv('DB_HOST', 'localhost')),
            'user': st.text_input("DB User", value=os.getenv('DB_USER', 'root')),
            'password': st.text_input("DB Password", type="password", 
                                     value=os.getenv('DB_PASSWORD', '')),
            'database': st.text_input("Database", value='harvard_artifacts'),
        }
    
    # Main content tabs
    tab1, tab2, tab3 = st.tabs(["📥 Data Collection", "📊 Analytics", "🔍 Custom Query"])
    
    with tab1:
        st.header("Data Collection & ETL")
        
        max_records = st.slider("Number of records to fetch", 100, 2000, 500, 100)
        
        if st.button("🚀 Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                try:
                    run_etl_pipeline(api_key, db_config, max_records)
                    st.success("ETL pipeline completed successfully!")
                except Exception as e:
                    st.error(f"ETL error: {e}")
    
    with tab2:
        st.header("Predefined Analytics")
        
        query_choice = st.selectbox(
            "Select Analysis",
            list(ANALYTICS_QUERIES.keys())
        )
        
        if st.button("📈 Run Analysis"):
            df_result = execute_query(ANALYTICS_QUERIES[query_choice], db_config)
            
            if not df_result.empty:
                st.dataframe(df_result, use_container_width=True)
                
                # Auto-generate visualization
                if len(df_result.columns) >= 2:
                    fig = px.bar(df_result, 
                                x=df_result.columns[0], 
                                y=df_result.columns[1],
                                title=query_choice.replace('_', ' ').title())
                    st.plotly_chart(fig, use_container_width=True)
    
    with tab3:
        st.header("Custom SQL Query")
        
        custom_query = st.text_area("Enter SQL Query", height=200)
        
        if st.button("▶️ Execute Query"):
            if custom_query:
                df_custom = execute_query(custom_query, db_config)
                st.dataframe(df_custom, use_container_width=True)


if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Launch Streamlit dashboard
streamlit run app.py
```

## Common Patterns

### Incremental Data Loading

```python
def get_max_artifact_id(db_config):
    """Get the maximum artifact ID already in database"""
    query = "SELECT COALESCE(MAX(id), 0) as max_id FROM artifactmetadata"
    df = execute_query(query, db_config)
    return df['max_id'].iloc[0]


def incremental_etl(api_key, db_config):
    """Load only new artifacts not in database"""
    max_id = get_max_artifact_id(db_config)
    
    # Fetch artifacts with ID > max_id
    records = fetch_artifacts_after_id(api_key, max_id)
    
    # Continue with transform and load...
```

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s...")
            time.sleep(wait_time)
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limits:

```python
import time

def fetch_with_rate_limit(api_key, page, delay=1):
    """Add delay between requests"""
    time.sleep(delay)
    return fetch_artifacts(api_key, page)
```

### Database Connection Issues

```python
def test_db_connection(db_config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✓ Database connection successful")
            return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
    finally:
        if connection.is_connected():
            connection.close()
```

### Missing Dependencies

```bash
# Install specific versions if compatibility issues arise
pip install streamlit==1.28.0
pip install pandas==2.0.3
pip install mysql-connector-python==8.1.0
pip install plotly==5.17.0
```

### Data Type Mismatches

```python
def safe_transform(value, max_length=None):
    """Safely transform values with type checking"""
    if value is None:
        return 'Unknown'
    
    str_value = str(value)
    
    if max_length:
        return str_value[:max_length]
    
    return str_value
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API with modern data engineering tools.
