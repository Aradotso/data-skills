---
name: harvard-artifacts-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build etl pipeline for harvard art museums
  - create data engineering project with streamlit
  - extract and analyze harvard artifacts data
  - set up sql analytics for museum collections
  - visualize harvard art museum data
  - implement etl workflow with api integration
  - analyze artifact metadata with sql queries
  - create interactive dashboard for museum data
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a complete data engineering application that demonstrates ETL pipelines, SQL analytics, and interactive visualization using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases, and provides an interactive Streamlit dashboard with 20+ analytical queries.

**Key capabilities:**
- API data collection with pagination and rate limiting
- ETL pipeline transforming JSON to relational SQL schema
- Multi-table database design (metadata, media, colors)
- Interactive SQL analytics dashboard
- Real-time visualization with Plotly

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

### API Key Setup

Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api

Set up environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

The project supports MySQL and TiDB Cloud. Set up connection parameters:

```python
# Database configuration (use environment variables)
import os

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

## Database Schema

### Table Structure

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    medium VARCHAR(300),
    dimensions VARCHAR(500),
    dated VARCHAR(200),
    period VARCHAR(200),
    url VARCHAR(500),
    creditline TEXT,
    copyright TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import pandas as pd
import time

def fetch_artifacts(api_key, num_records=100):
    """
    Extract artifact data from Harvard Art Museums API
    Handles pagination and rate limiting
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            artifacts.extend(records)
            
            if len(records) == 0:
                break
                
            page += 1
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Process and Structure Data

```python
def transform_artifacts(raw_data):
    """
    Transform nested JSON into structured DataFrames
    Returns three DataFrames: metadata, media, colors
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'copyright': artifact.get('copyright')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': img.get('baseimageurl'),
                'media_type': 'image'
            }
            media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed data into SQL database
    Uses batch inserts for performance
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Load metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             technique, medium, dimensions, dated, period, url, creditline, copyright)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Load media
        media_query = """
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Load colors
        colors_query = """
            INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for API key input
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection
    db_config = {
        'host': st.sidebar.text_input("Database Host", "localhost"),
        'user': st.sidebar.text_input("Database User", "root"),
        'password': st.sidebar.text_input("Database Password", type="password"),
        'database': st.sidebar.text_input("Database Name", "harvard_artifacts")
    }
    
    # Main sections
    tab1, tab2, tab3 = st.tabs(["📥 ETL Pipeline", "📊 SQL Analytics", "📈 Visualizations"])
    
    with tab1:
        run_etl_pipeline(api_key, db_config)
    
    with tab2:
        run_sql_analytics(db_config)
    
    with tab3:
        display_visualizations(db_config)

if __name__ == "__main__":
    main()
```

### ETL Pipeline Tab

```python
def run_etl_pipeline(api_key, db_config):
    st.header("Extract, Transform, Load Pipeline")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            raw_data = fetch_artifacts(api_key, num_records)
            st.success(f"✅ Extracted {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success(f"✅ Transformed into {len(metadata_df)} metadata records")
            st.dataframe(metadata_df.head())
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("✅ Data loaded successfully")
```

## SQL Analytics Queries

### Pre-built Analytical Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN EXISTS (SELECT 1 FROM artifactmedia WHERE artifact_id = m.id) 
                 THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata m
        GROUP BY media_status
    """,
    
    "Top Color Usage": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Multiple Images": """
        SELECT m.title, COUNT(media.image_url) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.id, m.title
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 10
    """
}
```

### Execute Analytics Query

```python
def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return pd.DataFrame()

def run_sql_analytics(db_config):
    st.header("SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.expander("View SQL Query"):
            st.code(query, language="sql")
        
        results = execute_query(query, db_config)
        
        if not results.empty:
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results.columns) == 2:
                fig = px.bar(results, x=results.columns[0], y=results.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)
```

## Visualization Patterns

```python
import plotly.express as px
import plotly.graph_objects as go

def display_visualizations(db_config):
    st.header("Interactive Visualizations")
    
    # Culture distribution pie chart
    query = "SELECT culture, COUNT(*) as count FROM artifactmetadata WHERE culture IS NOT NULL GROUP BY culture LIMIT 10"
    df = execute_query(query, db_config)
    
    if not df.empty:
        fig = px.pie(df, values='count', names='culture', 
                     title='Artifact Distribution by Culture')
        st.plotly_chart(fig, use_container_width=True)
    
    # Color usage heatmap
    color_query = """
        SELECT color_name, AVG(percentage) as avg_pct
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY avg_pct DESC
        LIMIT 20
    """
    color_df = execute_query(color_query, db_config)
    
    if not color_df.empty:
        fig = px.bar(color_df, x='color_name', y='avg_pct',
                     title='Average Color Percentage in Artifacts',
                     color='avg_pct', color_continuous_scale='Viridis')
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting API Requests

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator for rate limiting API calls"""
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
def fetch_artifact_by_id(artifact_id, api_key):
    url = f"https://api.harvardartmuseums.org/object/{artifact_id}"
    response = requests.get(url, params={'apikey': api_key})
    return response.json()
```

### Batch Processing

```python
def batch_insert(data_list, batch_size=1000):
    """Process large datasets in batches"""
    for i in range(0, len(data_list), batch_size):
        batch = data_list[i:i + batch_size]
        yield batch

# Usage
for batch in batch_insert(metadata_df.values.tolist(), batch_size=500):
    cursor.executemany(insert_query, batch)
    connection.commit()
```

## Troubleshooting

### API Connection Issues

```python
# Check API key validity
def validate_api_key(api_key):
    test_url = "https://api.harvardartmuseums.org/object"
    response = requests.get(test_url, params={'apikey': api_key, 'size': 1})
    return response.status_code == 200

# Handle API errors gracefully
def safe_api_call(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✅ Database connected successfully")
            connection.close()
            return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
# Use chunked reading for large result sets
def fetch_large_dataset(query, db_config, chunksize=10000):
    connection = mysql.connector.connect(**db_config)
    for chunk in pd.read_sql(query, connection, chunksize=chunksize):
        yield chunk
    connection.close()

# Usage
for chunk in fetch_large_dataset(query, db_config):
    process_chunk(chunk)
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, passwords)
2. **Implement retry logic** for API calls to handle transient failures
3. **Use batch inserts** for better database performance with large datasets
4. **Handle NULL values** in SQL queries to avoid empty results
5. **Implement proper indexing** on foreign key columns for query performance
6. **Cache expensive operations** in Streamlit using `@st.cache_data`
7. **Validate data** before inserting into database to maintain integrity
