---
name: harvard-art-museums-data-engineering-pipeline
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for the Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - set up Harvard Art Museums data analytics application
  - implement museum artifact collection data engineering
  - visualize Harvard Art Museums data with Streamlit
  - query and analyze Harvard Art Museums artifacts
  - extract and transform Harvard Art Museums API data
  - build analytics dashboard for museum collections
---

# Harvard Art Museums Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational tables, loading into SQL databases (MySQL/TiDB Cloud), running analytical queries, and visualizing results through an interactive Streamlit dashboard.

## Overview

The pipeline follows: **API → ETL → SQL → Analytics → Visualization**

- Collects artifact metadata, media, and color information
- Transforms nested JSON into normalized relational tables
- Performs batch SQL operations for performance
- Provides 20+ predefined analytical queries
- Generates interactive visualizations with Plotly

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Configuration

The project uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    dated VARCHAR(255),
    accession_number VARCHAR(100)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    thumbnail_url VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The application will open in your browser at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Fetch with pagination
def fetch_all_artifacts(max_pages=5):
    all_records = []
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(page=page)
        all_records.extend(data['records'])
        print(f"Fetched page {page}: {len(data['records'])} records")
    return all_records
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform raw API data into structured DataFrames"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', ''),
            'dated': artifact.get('dated', ''),
            'accession_number': artifact.get('accessionyear', '')
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': artifact.get('primaryimageurl'),
                'thumbnail_url': artifact.get('images', [{}])[0].get('baseimageurl') if artifact.get('images') else None
            }
            media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load DataFrames into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (batch insert for performance)
        metadata_tuples = [tuple(x) for x in df_metadata.to_numpy()]
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, technique, dated, accession_number)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_tuples)
        
        # Insert media
        if not df_media.empty:
            media_tuples = [tuple(x) for x in df_media.to_numpy()]
            media_query = """
                INSERT INTO artifactmedia (artifact_id, image_url, thumbnail_url)
                VALUES (%s, %s, %s)
            """
            cursor.executemany(media_query, media_tuples)
        
        # Insert colors
        if not df_colors.empty:
            colors_tuples = [tuple(x) for x in df_colors.to_numpy()]
            colors_query = """
                INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
                VALUES (%s, %s, %s)
            """
            cursor.executemany(colors_query, colors_tuples)
        
        connection.commit()
        print(f"Inserted {len(df_metadata)} artifacts, {len(df_media)} media, {len(df_colors)} colors")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors Used": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Media": """
        SELECT 
            am.classification,
            COUNT(DISTINCT am.id) as total_artifacts,
            COUNT(DISTINCT ame.artifact_id) as artifacts_with_media
        FROM artifactmetadata am
        LEFT JOIN artifactmedia ame ON am.id = ame.artifact_id
        GROUP BY am.classification
        ORDER BY total_artifacts DESC
    """,
    
    "Color Diversity by Period": """
        SELECT 
            am.period,
            COUNT(DISTINCT ac.color_hex) as unique_colors
        FROM artifactmetadata am
        JOIN artifactcolors ac ON am.id = ac.artifact_id
        WHERE am.period IS NOT NULL AND am.period != ''
        GROUP BY am.period
        ORDER BY unique_colors DESC
    """
}

def execute_analytics_query(query_name):
    """Execute predefined analytics query"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    
    # Sidebar for ETL operations
    with st.sidebar:
        st.header("ETL Pipeline")
        
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=20, value=5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                raw_data = fetch_all_artifacts(max_pages=num_pages)
                st.success(f"Fetched {len(raw_data)} artifacts")
            
            with st.spinner("Transforming data..."):
                df_meta, df_media, df_colors = transform_artifacts(raw_data)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                load_to_database(df_meta, df_media, df_colors)
                st.success("Data loaded successfully!")
    
    # Main analytics section
    st.header("📊 Analytics Dashboard")
    
    query_option = st.selectbox(
        "Select Analytics Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            result_df = execute_analytics_query(query_option)
            
            st.subheader("Query Results")
            st.dataframe(result_df, use_container_width=True)
            
            # Auto-generate visualization
            if len(result_df.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(
                    result_df,
                    x=result_df.columns[0],
                    y=result_df.columns[1],
                    title=query_option
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch with delay to respect API rate limits"""
    data = fetch_artifacts(page=page)
    time.sleep(delay)
    return data
```

### Error Handling for ETL

```python
def safe_etl_pipeline(max_pages=5):
    """ETL pipeline with error handling"""
    try:
        raw_data = fetch_all_artifacts(max_pages=max_pages)
        df_meta, df_media, df_colors = transform_artifacts(raw_data)
        load_to_database(df_meta, df_media, df_colors)
        return True, "ETL completed successfully"
    except requests.RequestException as e:
        return False, f"API error: {str(e)}"
    except Error as e:
        return False, f"Database error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected error: {str(e)}"
```

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the last loaded artifact ID"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    last_id = cursor.fetchone()[0] or 0
    connection.close()
    return last_id

def incremental_load():
    """Load only new artifacts"""
    last_id = get_last_artifact_id()
    raw_data = fetch_all_artifacts(max_pages=5)
    new_data = [a for a in raw_data if a['id'] > last_id]
    if new_data:
        df_meta, df_media, df_colors = transform_artifacts(new_data)
        load_to_database(df_meta, df_media, df_colors)
```

## Troubleshooting

### API Authentication Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Errors

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connection_timeout=5
        )
        if connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Empty Results from API

```python
# Check API response structure
def debug_api_response():
    response = fetch_artifacts(page=1, size=10)
    print(f"Total records: {response.get('info', {}).get('totalrecords')}")
    print(f"Records in response: {len(response.get('records', []))}")
    if response.get('records'):
        print(f"Sample record keys: {response['records'][0].keys()}")
```

### Streamlit Performance Issues

```python
# Use caching for expensive operations
@st.cache_data(ttl=3600)
def cached_query_execution(query_name):
    return execute_analytics_query(query_name)

# In Streamlit app
result_df = cached_query_execution(query_option)
```

## Advanced Usage

### Custom Analytics Query

```python
def run_custom_query(custom_sql):
    """Execute custom SQL query"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    try:
        df = pd.read_sql(custom_sql, connection)
        return df
    except Exception as e:
        st.error(f"Query error: {e}")
        return None
    finally:
        connection.close()

# In Streamlit
custom_query = st.text_area("Enter custom SQL query")
if st.button("Execute Custom Query"):
    result = run_custom_query(custom_query)
    if result is not None:
        st.dataframe(result)
```

### Export Results

```python
def export_to_csv(df, filename):
    """Export DataFrame to CSV"""
    df.to_csv(filename, index=False)
    return filename

# In Streamlit
if not result_df.empty:
    csv = result_df.to_csv(index=False).encode('utf-8')
    st.download_button(
        label="Download CSV",
        data=csv,
        file_name=f"{query_option.replace(' ', '_')}.csv",
        mime="text/csv"
    )
```
