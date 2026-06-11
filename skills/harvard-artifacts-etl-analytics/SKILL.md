---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create analytics dashboard for museum artifacts data
  - set up Harvard artifacts collection data engineering project
  - process Harvard Art Museums API data with Python
  - build Streamlit app for museum data visualization
  - query Harvard artifacts database with SQL
  - extract and transform Harvard museum data
  - create visualizations from Harvard Art Museums API
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App is a complete data pipeline that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON responses into normalized relational database tables
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** artifact collections using predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

```bash
# Python 3.8 or higher
python --version

# Install required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your credentials
```

### Required Environment Variables

Create a `.env` file with:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

To get a Harvard Art Museums API key, register at: https://www.harvardartmuseums.org/collections/api

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
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

def extract_all_artifacts(max_pages=5):
    """Extract multiple pages of artifacts with rate limiting"""
    import time
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        time.sleep(1)  # Rate limiting
    
    return all_artifacts
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata table"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Transform media/image data into separate table"""
    media_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            record = {
                'objectid': object_id,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Transform color data into separate table"""
    color_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'objectid': object_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### 3. Database Schema Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """Create database tables with proper schema"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            department VARCHAR(200),
            classification VARCHAR(200),
            medium TEXT,
            dated VARCHAR(200),
            accessionyear INT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Data Loading (ETL Load Phase)

```python
def load_dataframe_to_sql(df, table_name, connection):
    """Batch insert DataFrame into SQL table"""
    cursor = connection.cursor()
    
    if df.empty:
        return
    
    # Generate INSERT statement
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    insert_query = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Handle duplicates
    insert_query += " ON DUPLICATE KEY UPDATE "
    updates = [f"{col}=VALUES({col})" for col in df.columns if col != 'id']
    insert_query += ','.join(updates)
    
    # Batch insert
    data_tuples = [tuple(x) for x in df.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(df)} records into {table_name}")

def run_etl_pipeline():
    """Complete ETL pipeline execution"""
    # Extract
    print("Extracting data from API...")
    artifacts = extract_all_artifacts(max_pages=5)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    connection = create_database_connection()
    create_tables(connection)
    
    load_dataframe_to_sql(metadata_df, 'artifactmetadata', connection)
    load_dataframe_to_sql(media_df, 'artifactmedia', connection)
    load_dataframe_to_sql(colors_df, 'artifactcolors', connection)
    
    connection.close()
    print("ETL pipeline completed successfully!")
```

### 5. SQL Analytics Queries

```python
# Sample analytical queries

ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Multiple Images": """
        SELECT am.objectid, am.title, COUNT(media.imageid) as image_count
        FROM artifactmetadata am
        JOIN artifactmedia media ON am.objectid = media.objectid
        GROUP BY am.objectid, am.title
        HAVING image_count > 3
        ORDER BY image_count DESC
    """,
    
    "Average Image Dimensions": """
        SELECT 
            AVG(width) as avg_width, 
            AVG(height) as avg_height,
            COUNT(*) as total_images
        FROM artifactmedia
        WHERE width IS NOT NULL AND height IS NOT NULL
    """
}

def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # ETL Section
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline()
            st.success("ETL completed successfully!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        connection = create_database_connection()
        
        if connection:
            query = ANALYTICS_QUERIES[query_name]
            
            # Show query
            with st.expander("View SQL Query"):
                st.code(query, language='sql')
            
            # Execute and display results
            df_result = execute_query(query, connection)
            
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Auto-visualization
            if len(df_result.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
            
            connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_last_processed_id(connection):
    """Get the highest objectid already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Load only new artifacts since last run"""
    connection = create_database_connection()
    last_id = get_last_processed_id(connection)
    
    # Fetch only artifacts with ID > last_id
    artifacts = fetch_artifacts_since(last_id)
    # Transform and load as before
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_call(url, params, retries=3):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.RequestException as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Pattern 3: Data Quality Validation

```python
def validate_artifact_data(df):
    """Validate transformed data before loading"""
    issues = []
    
    # Check for required fields
    if df['objectid'].isnull().any():
        issues.append("Missing objectid values")
    
    # Check for duplicates
    if df['objectid'].duplicated().any():
        issues.append("Duplicate objectid values found")
    
    # Check data types
    if not pd.api.types.is_integer_dtype(df['objectid']):
        issues.append("objectid must be integer")
    
    if issues:
        raise ValueError(f"Data validation failed: {', '.join(issues)}")
    
    return True
```

## Troubleshooting

### API Rate Limiting

If you encounter 429 errors:

```python
import time
from functools import wraps

def rate_limit(calls_per_minute=60):
    """Decorator to enforce rate limiting"""
    min_interval = 60.0 / calls_per_minute
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_minute=30)
def fetch_artifacts_limited(page):
    return fetch_artifacts(page)
```

### Database Connection Issues

```python
def get_robust_connection(max_retries=3):
    """Get database connection with retry logic"""
    for attempt in range(max_retries):
        try:
            connection = create_database_connection()
            if connection and connection.is_connected():
                return connection
        except Error as e:
            logger.error(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(2)
    
    raise Exception("Could not establish database connection")
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(total_pages, batch_size=10):
    """Process artifacts in batches to manage memory"""
    connection = create_database_connection()
    
    for start_page in range(1, total_pages + 1, batch_size):
        end_page = min(start_page + batch_size, total_pages + 1)
        
        # Process batch
        batch_artifacts = []
        for page in range(start_page, end_page):
            batch_artifacts.extend(fetch_artifacts(page=page))
        
        # Transform and load batch
        metadata_df = transform_artifact_metadata(batch_artifacts)
        load_dataframe_to_sql(metadata_df, 'artifactmetadata', connection)
        
        # Clear memory
        del batch_artifacts, metadata_df
        logger.info(f"Processed pages {start_page} to {end_page - 1}")
    
    connection.close()
```

### Streamlit Caching for Performance

```python
@st.cache_data(ttl=3600)
def cached_query_execution(query_text):
    """Cache query results for 1 hour"""
    connection = create_database_connection()
    result = execute_query(query_text, connection)
    connection.close()
    return result

# Use in Streamlit app
df_result = cached_query_execution(ANALYTICS_QUERIES[query_name])
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards with the Harvard Art Museums API, enabling AI agents to assist developers in creating production-ready data engineering applications.
