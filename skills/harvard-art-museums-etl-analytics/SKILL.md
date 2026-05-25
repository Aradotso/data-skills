---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL storage, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - integrate Harvard Art Museums API
  - create a data engineering pipeline with Streamlit
  - extract and transform art museum collection data
  - visualize museum artifacts analytics
  - build SQL analytics dashboard for art collections
  - implement museum data pipeline with Python
  - create interactive art data visualization
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering solution that demonstrates real-world ETL pipelines. It extracts artifact data from the Harvard Art Museums API, transforms it into structured relational tables, loads it into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards using Streamlit.

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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

### Environment Variables

Create a `.env` file or set environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
import os

# Database connection configuration
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Create database connection
conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()

# Create tables
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    rank INT
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
)
""")

conn.commit()
cursor.close()
conn.close()
```

## API Integration

### Fetching Data from Harvard Art Museums API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: API key for Harvard Art Museums
        page: Page number for pagination
        size: Number of records per page
    
    Returns:
        JSON response containing artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
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
print(f"Fetched: {len(data['records'])} artifacts")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts with rate limiting"""
    import time
    
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data['records'])
            
            print(f"Fetched page {page}: {len(data['records'])} records")
            
            # Rate limiting - avoid overwhelming the API
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data into structured metadata"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'accessionyear': artifact.get('accessionyear'),
            'rank': artifact.get('rank')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """Transform media/image data"""
    media_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'baseimageurl': artifact.get('baseimageurl', '')[:500],
            'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Transform color data (nested JSON)"""
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'objectid': objectid,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load to Database

```python
def load_to_database(df, table_name, db_config):
    """
    Load DataFrame to MySQL/TiDB database using batch inserts
    
    Args:
        df: pandas DataFrame
        table_name: Target table name
        db_config: Database configuration dict
    """
    import mysql.connector
    
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Prepare insert query
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    
    conn.commit()
    print(f"Inserted {cursor.rowcount} records into {table_name}")
    
    cursor.close()
    conn.close()

# Complete ETL pipeline
def run_etl_pipeline(api_key, db_config, max_pages=5):
    """Execute complete ETL pipeline"""
    
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(api_key, max_pages=max_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    load_to_database(metadata_df, 'artifactmetadata', db_config)
    load_to_database(media_df, 'artifactmedia', db_config)
    load_to_database(colors_df, 'artifactcolors', db_config)
    
    print("ETL pipeline completed successfully!")
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_top_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Department distribution
query_departments = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Query 4: Most popular colors
query_top_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15
"""

# Query 5: Artifacts with high page views
query_popular_artifacts = """
SELECT m.title, m.culture, a.totalpageviews
FROM artifactmetadata m
JOIN artifactmedia a ON m.objectid = a.objectid
ORDER BY a.totalpageviews DESC
LIMIT 20
"""

# Query 6: Color spectrum analysis
query_spectrum = """
SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
FROM artifactcolors
WHERE spectrum IS NOT NULL
GROUP BY spectrum
ORDER BY count DESC
"""

# Query 7: Accession year trends
query_accession_trends = """
SELECT accessionyear, COUNT(*) as artifacts_acquired
FROM artifactmetadata
WHERE accessionyear IS NOT NULL
GROUP BY accessionyear
ORDER BY accessionyear DESC
LIMIT 30
"""

def execute_query(query, db_config):
    """Execute SQL query and return DataFrame"""
    import mysql.connector
    import pandas as pd
    
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

## Streamlit Application

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
import os

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        
        # API Key input
        api_key = st.text_input("Harvard API Key", 
                                type="password",
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        # Database configuration
        st.subheader("Database Settings")
        db_host = st.text_input("Host", value=os.getenv('DB_HOST', ''))
        db_user = st.text_input("User", value=os.getenv('DB_USER', ''))
        db_password = st.text_input("Password", type="password",
                                   value=os.getenv('DB_PASSWORD', ''))
        db_name = st.text_input("Database", value=os.getenv('DB_NAME', ''))
    
    db_config = {
        'host': db_host,
        'user': db_user,
        'password': db_password,
        'database': db_name
    }
    
    # Main tabs
    tab1, tab2, tab3 = st.tabs(["📥 ETL Pipeline", "📊 Analytics", "📈 Visualizations"])
    
    with tab1:
        st.header("ETL Pipeline")
        
        col1, col2 = st.columns(2)
        with col1:
            max_pages = st.number_input("Pages to fetch", min_value=1, max_value=50, value=5)
        
        if st.button("▶️ Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                try:
                    run_etl_pipeline(api_key, db_config, max_pages)
                    st.success("ETL pipeline completed successfully!")
                except Exception as e:
                    st.error(f"Error: {e}")
    
    with tab2:
        st.header("SQL Analytics Dashboard")
        
        queries = {
            "Top Cultures": query_top_cultures,
            "Artifacts by Century": query_by_century,
            "Department Distribution": query_departments,
            "Popular Colors": query_top_colors,
            "Most Viewed Artifacts": query_popular_artifacts
        }
        
        selected_query = st.selectbox("Select Analysis", list(queries.keys()))
        
        if st.button("Execute Query"):
            try:
                df = execute_query(queries[selected_query], db_config)
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) >= 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=selected_query)
                    st.plotly_chart(fig, use_container_width=True)
            except Exception as e:
                st.error(f"Query failed: {e}")
    
    with tab3:
        st.header("Interactive Visualizations")
        
        viz_type = st.selectbox("Visualization Type", 
                                ["Culture Distribution", "Color Analysis", 
                                 "Timeline", "Department Breakdown"])
        
        # Custom visualization logic here
        st.info("Select a visualization type to view insights")

if __name__ == "__main__":
    main()
```

### Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_objectid(db_config):
    """Get the last loaded object ID to support incremental loads"""
    import mysql.connector
    
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    
    cursor.close()
    conn.close()
    
    return result[0] if result[0] else 0

def incremental_etl(api_key, db_config):
    """Load only new artifacts since last run"""
    last_id = get_last_objectid(db_config)
    
    # Fetch artifacts with objectid > last_id
    # Implementation depends on API filtering capabilities
    pass
```

### Error Handling and Logging

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

def safe_etl_pipeline(api_key, db_config):
    """ETL with comprehensive error handling"""
    try:
        logging.info("Starting ETL pipeline")
        artifacts = fetch_all_artifacts(api_key)
        logging.info(f"Extracted {len(artifacts)} artifacts")
        
        metadata_df = transform_artifact_metadata(artifacts)
        logging.info(f"Transformed {len(metadata_df)} metadata records")
        
        load_to_database(metadata_df, 'artifactmetadata', db_config)
        logging.info("Data loaded successfully")
        
    except Exception as e:
        logging.error(f"ETL pipeline failed: {e}", exc_info=True)
        raise
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
            wait_time = min_interval - elapsed
            
            if wait_time > 0:
                time.sleep(wait_time)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_with_limit(api_key, page=1, size=100):
    return fetch_artifacts(api_key, page, size)
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    """Test database connectivity"""
    import mysql.connector
    from mysql.connector import Error
    
    try:
        conn = mysql.connector.connect(**db_config)
        if conn.is_connected():
            print("✓ Database connection successful")
            cursor = conn.cursor()
            cursor.execute("SELECT DATABASE()")
            db = cursor.fetchone()
            print(f"✓ Connected to database: {db[0]}")
            cursor.close()
            conn.close()
            return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Data Validation

```python
def validate_artifacts_data(df):
    """Validate DataFrame before loading"""
    issues = []
    
    # Check for required columns
    required_cols = ['objectid', 'title']
    for col in required_cols:
        if col not in df.columns:
            issues.append(f"Missing required column: {col}")
    
    # Check for duplicates
    if df['objectid'].duplicated().any():
        issues.append("Duplicate object IDs found")
    
    # Check for null values in key columns
    if df['objectid'].isnull().any():
        issues.append("Null values in objectid column")
    
    if issues:
        raise ValueError(f"Data validation failed: {', '.join(issues)}")
    
    return True
```

This skill provides comprehensive guidance for building ETL pipelines with the Harvard Art Museums API, including data extraction, transformation, SQL analytics, and interactive visualization using Streamlit.
