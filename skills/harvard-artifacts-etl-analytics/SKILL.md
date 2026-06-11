---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data with MySQL and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - create a data engineering app with Harvard artifacts collection
  - set up SQL analytics for Harvard museum data
  - build a Streamlit dashboard for art museum artifacts
  - extract and transform Harvard Art Museums API data
  - create artifact analytics with SQL and visualization
  - integrate Harvard Art Museums API with MySQL database
  - build end-to-end data pipeline for museum artifacts
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates:

- **ETL Pipeline**: Extract artifact data from Harvard API, transform nested JSON into relational format, load into MySQL/TiDB
- **SQL Analytics**: Run 20+ predefined analytical queries on artifact metadata, media, and color data
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time insights
- **Database Design**: Proper relational schema with foreign keys across `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

Required packages:
- `streamlit` - Web app framework
- `pandas` - Data manipulation
- `requests` - API calls
- `mysql-connector-python` or `pymysql` - Database connectivity
- `plotly` - Interactive visualizations

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
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
        print(f"Database connection error: {e}")
        return None
```

### Database Schema

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    base_image_url TEXT,
    thumbnail_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    hex_code VARCHAR(10),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """
    Extract artifact data from Harvard Art Museums API
    Handles pagination and rate limiting
    """
    all_records = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(BASE_URL, params=params)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_records.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} records")
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_records
```

### Transform: Process JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational dataframes
    Returns three dataframes: metadata, media, colors
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': img.get('baseimageurl'),
                    'base_image_url': img.get('baseimageurl'),
                    'thumbnail_url': img.get('thumbnailurl')
                }
                media_records.append(media)
        
        # Extract color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percentage': color.get('percent'),
                    'hex_code': color.get('hex')
                }
                color_records.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Batch Insert into Database

```python
def load_to_database(df_metadata, df_media, df_colors):
    """
    Load transformed data into MySQL database with batch inserts
    """
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, division, department, 
         dated, period, technique, medium, creditline, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, image_url, base_image_url, thumbnail_url)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, percentage, hex_code)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, df_colors.values.tolist())
        
        connection.commit()
        print("Data loaded successfully!")
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
        
    finally:
        cursor.close()
        connection.close()
```

## Running the ETL Pipeline

```python
def run_etl_pipeline(api_key, num_pages=10):
    """
    Execute complete ETL pipeline
    """
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Step 1: Extracting data from API...")
    raw_data = fetch_artifacts(api_key, num_pages)
    print(f"Extracted {len(raw_data)} artifacts")
    
    # Transform
    print("Step 2: Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_data)
    print(f"Transformed: {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} colors")
    
    # Load
    print("Step 3: Loading data to database...")
    success = load_to_database(df_metadata, df_media, df_colors)
    
    if success:
        print("ETL Pipeline completed successfully!")
    else:
        print("ETL Pipeline failed!")
    
    return success

# Execute
api_key = os.getenv('HARVARD_API_KEY')
run_etl_pipeline(api_key, num_pages=5)
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "ETL Pipeline":
        show_etl_page()
    elif menu == "SQL Analytics":
        show_analytics_page()
    elif menu == "Visualizations":
        show_visualizations_page()

if __name__ == "__main__":
    main()
```

### ETL Page

```python
def show_etl_page():
    st.header("ETL Pipeline Control")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input(
            "Number of pages to fetch",
            min_value=1,
            max_value=100,
            value=10
        )
    
    with col2:
        api_key = st.text_input(
            "API Key",
            type="password",
            value=os.getenv('HARVARD_API_KEY', '')
        )
    
    if st.button("Run ETL Pipeline", type="primary"):
        with st.spinner("Running ETL pipeline..."):
            success = run_etl_pipeline(api_key, num_pages)
            
            if success:
                st.success("✅ ETL Pipeline completed successfully!")
            else:
                st.error("❌ ETL Pipeline failed!")
```

## SQL Analytics Queries

### Predefined Analytics Functions

```python
def get_artifacts_by_culture():
    """Count artifacts by culture"""
    query = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 15
    """
    return execute_query(query)

def get_artifacts_by_century():
    """Distribution of artifacts by century"""
    query = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
    """
    return execute_query(query)

def get_media_availability():
    """Artifacts with and without media"""
    query = """
    SELECT 
        CASE WHEN m.artifact_id IS NOT NULL THEN 'With Media' ELSE 'Without Media' END as media_status,
        COUNT(*) as count
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    GROUP BY media_status
    """
    return execute_query(query)

def get_top_colors():
    """Most common colors across artifacts"""
    query = """
    SELECT color, COUNT(*) as count, AVG(percentage) as avg_percentage
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY count DESC
    LIMIT 10
    """
    return execute_query(query)

def get_artifacts_by_department():
    """Artifact distribution by department"""
    query = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
    """
    return execute_query(query)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = get_db_connection()
    if connection:
        try:
            df = pd.read_sql(query, connection)
            return df
        except Error as e:
            st.error(f"Query error: {e}")
            return None
        finally:
            connection.close()
    return None
```

### Analytics Dashboard Page

```python
def show_analytics_page():
    st.header("SQL Analytics Dashboard")
    
    # Query selector
    queries = {
        "Artifacts by Culture": get_artifacts_by_culture,
        "Artifacts by Century": get_artifacts_by_century,
        "Media Availability": get_media_availability,
        "Top Colors": get_top_colors,
        "Artifacts by Department": get_artifacts_by_department
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = queries[selected_query]()
            
            if df is not None and not df.empty:
                st.subheader("Query Results")
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=selected_query
                    )
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No data returned")
```

## Visualization Examples

### Interactive Bar Chart

```python
def create_bar_chart(df, x_col, y_col, title):
    """Create interactive Plotly bar chart"""
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        color=y_col,
        color_continuous_scale='viridis'
    )
    fig.update_layout(
        xaxis_title=x_col.title(),
        yaxis_title=y_col.title(),
        showlegend=False
    )
    return fig
```

### Pie Chart for Distributions

```python
def create_pie_chart(df, names_col, values_col, title):
    """Create interactive pie chart"""
    fig = px.pie(
        df,
        names=names_col,
        values=values_col,
        title=title,
        hole=0.3
    )
    return fig
```

### Time Series Analysis

```python
def analyze_artifacts_over_time():
    """Analyze artifact acquisition over time"""
    query = """
    SELECT accession_number as year, COUNT(*) as count
    FROM artifactmetadata
    WHERE accession_number IS NOT NULL
    GROUP BY accession_number
    ORDER BY accession_number
    """
    df = execute_query(query)
    
    if df is not None:
        fig = px.line(
            df,
            x='year',
            y='count',
            title='Artifact Acquisitions Over Time'
        )
        return fig
    return None
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_execution(func):
    """Decorator for safe ETL execution with error handling"""
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except Exception as e:
            st.error(f"ETL Error: {str(e)}")
            return None
    return wrapper

@safe_etl_execution
def extract_with_retry(api_key, max_retries=3):
    """Extract data with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the last artifact ID loaded"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    df = execute_query(query)
    return df['max_id'].iloc[0] if df is not None else 0

def incremental_load(api_key, last_id):
    """Load only new artifacts since last run"""
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': last_id
    }
    response = requests.get(BASE_URL, params=params)
    return response.json().get('records', [])
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_single_page(api_key, page):
    # Your API call here
    pass
```

### Database Connection Pooling

For better performance:

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_pooled_connection():
    """Get connection from pool"""
    return db_pool.get_connection()
```

### Handling Missing Data

```python
def clean_artifact_data(artifact):
    """Clean and validate artifact data"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Untitled'),
        'culture': artifact.get('culture') or 'Unknown',
        'century': artifact.get('century') or 'Unknown',
        # Convert empty strings to None for proper NULL handling
        'classification': artifact.get('classification') or None,
        'department': artifact.get('department') or None
    }
```

### Memory Optimization for Large Datasets

```python
def chunked_etl(api_key, total_pages, chunk_size=10):
    """Process ETL in chunks to manage memory"""
    for start_page in range(1, total_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size, total_pages + 1)
        
        # Extract chunk
        raw_data = fetch_artifacts(api_key, num_pages=chunk_size, start_page=start_page)
        
        # Transform chunk
        df_metadata, df_media, df_colors = transform_artifacts(raw_data)
        
        # Load chunk
        load_to_database(df_metadata, df_media, df_colors)
        
        # Clear memory
        del raw_data, df_metadata, df_media, df_colors
        
        print(f"Processed pages {start_page} to {end_page - 1}")
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8080

# With auto-reload during development
streamlit run app.py --server.runOnSave true
```

Access the dashboard at `http://localhost:8501`
