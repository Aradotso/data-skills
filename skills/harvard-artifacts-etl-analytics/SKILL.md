---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create analytics dashboard with museum artifacts data
  - extract and transform Harvard museum collection data
  - set up SQL database for art museum artifacts
  - build Streamlit app for artifact analytics
  - query and visualize Harvard Art Museums collection
  - implement data pipeline for museum artifacts
  - analyze art collection data with SQL queries
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline development, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App demonstrates:
- **API Integration**: Fetching paginated artifact data from Harvard Art Museums API
- **ETL Pipeline**: Extracting, transforming, and loading nested JSON into relational SQL tables
- **SQL Analytics**: Running predefined analytical queries on artifact metadata
- **Visualization**: Creating interactive dashboards with Plotly and Streamlit

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
export DB_NAME="your_database_name"
```

### Dependencies

```text
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
cursor = connection.cursor()
```

## Database Schema

Create the three core tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    objectnumber VARCHAR(100),
    dated VARCHAR(200),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    mediatype VARCHAR(100),
    baseimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(BASE_URL, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        print(f"Error: {response.status_code}")
        return None

def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts with rate limiting"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_records.extend(data['records'])
            time.sleep(1)  # Rate limiting
        else:
            break
    
    return all_records
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(records):
    """Transform artifact records into metadata DataFrame"""
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'objectnumber': record.get('objectnumber'),
            'dated': record.get('dated'),
            'url': record.get('url')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(records):
    """Extract media information from nested structure"""
    media = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for image in images:
            media.append({
                'artifact_id': artifact_id,
                'mediatype': 'image',
                'baseimageurl': image.get('baseimageurl')
            })
    
    return pd.DataFrame(media)

def transform_artifact_colors(records):
    """Extract color data from artifacts"""
    colors = []
    
    for record in records:
        artifact_id = record.get('id')
        color_data = record.get('colors', [])
        
        for color_entry in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color_entry.get('color'),
                'percentage': color_entry.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### Load: Insert into SQL Database

```python
def load_metadata_to_sql(df, connection):
    """Batch insert metadata into SQL"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, objectnumber, dated, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media_to_sql(df, connection):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, mediatype, baseimageurl)
    VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")

def load_colors_to_sql(df, connection):
    """Batch insert color data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, percentage)
    VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")
```

### Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, connection, max_pages=10):
    """Execute complete ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Step 1: Extracting data from API...")
    records = fetch_all_artifacts(api_key, max_pages=max_pages)
    print(f"Extracted {len(records)} records")
    
    # Transform
    print("Step 2: Transforming data...")
    metadata_df = transform_artifact_metadata(records)
    media_df = transform_artifact_media(records)
    colors_df = transform_artifact_colors(records)
    
    # Load
    print("Step 3: Loading data to SQL...")
    load_metadata_to_sql(metadata_df, connection)
    load_media_to_sql(media_df, connection)
    load_colors_to_sql(colors_df, connection)
    
    print("ETL Pipeline completed successfully!")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Artifacts with media
query_media_availability = """
SELECT 
    CASE WHEN media_count > 0 THEN 'With Media' ELSE 'Without Media' END as media_status,
    COUNT(*) as artifact_count
FROM (
    SELECT m.id, COUNT(am.media_id) as media_count
    FROM artifactmetadata m
    LEFT JOIN artifactmedia am ON m.id = am.artifact_id
    GROUP BY m.id
) as subquery
GROUP BY media_status;
"""

# Query 4: Top colors across artifacts
query_top_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
FROM artifactcolors
GROUP BY color
ORDER BY usage_count DESC
LIMIT 10;
"""

# Query 5: Department distribution
query_by_department = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY count DESC;
"""

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end ETL and Analytics Application")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    try:
        connection = mysql.connector.connect(**db_config)
        st.sidebar.success("✅ Database Connected")
    except Exception as e:
        st.sidebar.error(f"❌ Database Error: {e}")
        return
    
    # Tabs for different functionalities
    tab1, tab2, tab3 = st.tabs(["📥 ETL Pipeline", "📊 Analytics", "📈 Visualizations"])
    
    with tab1:
        show_etl_interface(api_key, connection)
    
    with tab2:
        show_analytics_interface(connection)
    
    with tab3:
        show_visualizations(connection)

def show_etl_interface(api_key, connection):
    """ETL Pipeline interface"""
    st.header("ETL Pipeline")
    
    max_pages = st.slider("Number of pages to fetch", 1, 20, 5)
    
    if st.button("🚀 Run ETL Pipeline"):
        with st.spinner("Running ETL Pipeline..."):
            run_etl_pipeline(api_key, connection, max_pages=max_pages)
            st.success("ETL Pipeline completed!")

def show_analytics_interface(connection):
    """Analytics queries interface"""
    st.header("SQL Analytics")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Media Availability": query_media_availability,
        "Top Colors": query_top_colors,
        "Department Distribution": query_by_department
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(connection, queries[selected_query])
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations(connection):
    """Pre-built visualizations"""
    st.header("Interactive Visualizations")
    
    # Visualization 1: Culture distribution
    df_culture = execute_query(connection, query_by_culture)
    fig1 = px.bar(df_culture, x='culture', y='artifact_count',
                  title='Top 10 Cultures by Artifact Count')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Visualization 2: Century timeline
    df_century = execute_query(connection, query_by_century)
    fig2 = px.line(df_century, x='century', y='count',
                   title='Artifacts by Century')
    st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Handling API Rate Limits

```python
from time import sleep
from functools import wraps

def rate_limit(calls_per_second=1):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                sleep(left_to_wait)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limit(calls_per_second=1)
def fetch_artifacts_limited(api_key, page=1):
    return fetch_artifacts(api_key, page)
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, connection, max_pages=10):
    """ETL pipeline with error handling"""
    try:
        records = fetch_all_artifacts(api_key, max_pages)
        
        if not records:
            raise ValueError("No records fetched from API")
        
        metadata_df = transform_artifact_metadata(records)
        media_df = transform_artifact_media(records)
        colors_df = transform_artifact_colors(records)
        
        # Validate data
        if metadata_df.empty:
            raise ValueError("Metadata transformation resulted in empty DataFrame")
        
        load_metadata_to_sql(metadata_df, connection)
        load_media_to_sql(media_df, connection)
        load_colors_to_sql(colors_df, connection)
        
        return True
        
    except requests.RequestException as e:
        print(f"API Error: {e}")
        return False
    except mysql.connector.Error as e:
        print(f"Database Error: {e}")
        connection.rollback()
        return False
    except Exception as e:
        print(f"Unexpected Error: {e}")
        return False
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    """Verify API key and connectivity"""
    try:
        response = requests.get(
            f"{BASE_URL}?apikey={api_key}&size=1",
            timeout=10
        )
        if response.status_code == 200:
            print("✅ API connection successful")
            return True
        elif response.status_code == 401:
            print("❌ Invalid API key")
            return False
        else:
            print(f"❌ API error: {response.status_code}")
            return False
    except requests.Timeout:
        print("❌ Request timed out")
        return False
    except Exception as e:
        print(f"❌ Connection error: {e}")
        return False
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    """Test database connectivity"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        print("✅ Database connection successful")
        return True
    except mysql.connector.Error as e:
        print(f"❌ Database error: {e}")
        return False
```

### Data Validation

```python
def validate_dataframe(df, required_columns):
    """Validate DataFrame before loading"""
    missing_cols = set(required_columns) - set(df.columns)
    if missing_cols:
        raise ValueError(f"Missing columns: {missing_cols}")
    
    if df.empty:
        raise ValueError("DataFrame is empty")
    
    null_counts = df.isnull().sum()
    if null_counts.any():
        print(f"Warning: Null values found:\n{null_counts[null_counts > 0]}")
    
    return True
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

This skill provides everything needed to build, deploy, and extend ETL pipelines and analytics dashboards using the Harvard Art Museums API with modern Python data engineering tools.
