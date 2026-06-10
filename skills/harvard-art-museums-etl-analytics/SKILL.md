---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build a data pipeline for harvard art museums
  - create an etl workflow for museum artifacts
  - set up harvard museums api analytics dashboard
  - extract and analyze art collection data
  - build streamlit app for museum data visualization
  - implement sql analytics for artifact collections
  - create harvard art museums data engineering pipeline
  - visualize museum artifact data with plotly
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build production-ready ETL pipelines that extract artifact data from the Harvard Art Museums API, transform it into structured relational data, load it into SQL databases, and create interactive analytics dashboards using Streamlit and Plotly.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates:
- **API Integration**: Paginated data collection from Harvard Art Museums API
- **ETL Pipeline**: Extract nested JSON, transform to relational format, batch load to SQL
- **Database Design**: Normalized schema with artifact metadata, media, and color tables
- **SQL Analytics**: 20+ pre-built analytical queries for insights
- **Interactive Dashboards**: Streamlit UI with Plotly visualizations

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
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Architecture Overview

**Data Flow**: API → Extract → Transform → Load → SQL → Analytics → Visualization

**Database Schema**:
- `artifactmetadata`: Core artifact information (id, title, culture, century, department)
- `artifactmedia`: Media URLs and image details (artifact_id FK)
- `artifactcolors`: Color palette data (artifact_id FK)

## Key Components

### 1. API Integration

```python
import requests
import os

def fetch_harvard_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_harvard_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
```

### 2. ETL Pipeline - Extract

```python
import pandas as pd

def extract_artifacts(api_key, num_pages=5, page_size=100):
    """
    Extract artifact data across multiple pages
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        response = fetch_harvard_artifacts(api_key, page=page, size=page_size)
        artifacts = response.get('records', [])
        all_artifacts.extend(artifacts)
    
    return all_artifacts

# Extract data
artifacts = extract_artifacts(os.getenv("HARVARD_API_KEY"), num_pages=3)
print(f"Extracted {len(artifacts)} artifacts")
```

### 3. ETL Pipeline - Transform

```python
def transform_artifact_metadata(artifacts):
    """
    Transform raw API response to structured metadata DataFrame
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', ''),
            'accession_year': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """
    Extract media information from nested structures
    """
    media_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_data.append({
                'artifact_id': artifact_id,
                'media_id': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl', ''),
                'format': img.get('format', ''),
                'width': img.get('width'),
                'height': img.get('height')
            })
    
    return pd.DataFrame(media_data)

def transform_artifact_colors(artifacts):
    """
    Extract color palette information
    """
    colors_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            colors_data.append({
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percent': color.get('percent', 0.0),
                'hue': color.get('hue', '')
            })
    
    return pd.DataFrame(colors_data)

# Transform all data
df_metadata = transform_artifact_metadata(artifacts)
df_media = transform_artifact_media(artifacts)
df_colors = transform_artifact_colors(artifacts)
```

### 4. ETL Pipeline - Load to SQL

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL/TiDB connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv("DB_HOST"),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
            database=os.getenv("DB_NAME")
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """
    Create database schema
    """
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            url TEXT,
            accession_year INT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_id INT,
            baseimageurl TEXT,
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            hue VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_dataframe_to_sql(connection, df, table_name):
    """
    Batch insert DataFrame to SQL table
    """
    cursor = connection.cursor()
    
    # Prepare INSERT statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(df)} records into {table_name}")

# Load data
conn = create_database_connection()
create_tables(conn)
load_dataframe_to_sql(conn, df_metadata, 'artifactmetadata')
load_dataframe_to_sql(conn, df_media, 'artifactmedia')
load_dataframe_to_sql(conn, df_colors, 'artifactcolors')
conn.close()
```

### 5. SQL Analytics Queries

```python
def execute_analytics_query(connection, query):
    """
    Execute analytical query and return results as DataFrame
    """
    return pd.read_sql(query, connection)

# Sample analytical queries
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
            CASE WHEN m.artifact_id IS NOT NULL THEN 'With Images' ELSE 'No Images' END as image_status,
            COUNT(DISTINCT a.artifact_id) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.artifact_id = m.artifact_id
        GROUP BY image_status
    """,
    
    "top_colors_by_usage": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
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

# Execute queries
conn = create_database_connection()
for query_name, query_sql in ANALYTICS_QUERIES.items():
    result = execute_analytics_query(conn, query_sql)
    print(f"\n{query_name}:\n{result}")
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Configuration")
api_key = st.sidebar.text_input("API Key", type="password", value=os.getenv("HARVARD_API_KEY", ""))
num_pages = st.sidebar.slider("Number of pages to fetch", 1, 10, 3)

# ETL Pipeline section
if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Extracting data from API..."):
        artifacts = extract_artifacts(api_key, num_pages=num_pages)
        st.success(f"Extracted {len(artifacts)} artifacts")
    
    with st.spinner("Transforming data..."):
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
        st.success("Data transformed successfully")
    
    with st.spinner("Loading to database..."):
        conn = create_database_connection()
        create_tables(conn)
        load_dataframe_to_sql(conn, df_metadata, 'artifactmetadata')
        load_dataframe_to_sql(conn, df_media, 'artifactmedia')
        load_dataframe_to_sql(conn, df_colors, 'artifactcolors')
        conn.close()
        st.success("Data loaded to database")

# Analytics section
st.header("📊 Analytics Dashboard")

query_selection = st.selectbox(
    "Select Analysis",
    list(ANALYTICS_QUERIES.keys())
)

if st.button("Run Query"):
    conn = create_database_connection()
    result_df = execute_analytics_query(conn, ANALYTICS_QUERIES[query_selection])
    conn.close()
    
    # Display results
    st.subheader("Query Results")
    st.dataframe(result_df)
    
    # Auto-generate visualization
    if len(result_df.columns) >= 2:
        fig = px.bar(
            result_df,
            x=result_df.columns[0],
            y=result_df.columns[1],
            title=query_selection.replace('_', ' ').title()
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, num_pages, delay=1):
    """
    Fetch data with rate limiting to avoid API throttling
    """
    artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_harvard_artifacts(api_key, page=page)
        artifacts.extend(data.get('records', []))
        time.sleep(delay)  # Rate limit
    return artifacts
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default='Unknown'):
    """
    Safely extract values with fallback
    """
    value = dictionary.get(key, default)
    return value if value is not None else default
```

### Incremental ETL Updates

```python
def get_max_artifact_id(connection):
    """
    Get latest artifact_id for incremental loads
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, connection):
    """
    Load only new artifacts
    """
    max_id = get_max_artifact_id(connection)
    # Fetch artifacts with id > max_id
    # Transform and load only new records
```

## Troubleshooting

**API Key Issues**:
- Ensure `HARVARD_API_KEY` environment variable is set
- Get API key from: https://docs.harvardartmuseums.org/

**Database Connection Errors**:
- Verify `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` are correct
- Check network connectivity and firewall rules
- For TiDB Cloud, ensure SSL/TLS settings if required

**Memory Issues with Large Datasets**:
- Reduce `page_size` parameter
- Process data in smaller batches
- Use chunked DataFrame operations

**Empty Query Results**:
- Run ETL pipeline first to populate database
- Check table existence with `SHOW TABLES`
- Verify foreign key constraints aren't blocking inserts

**Streamlit Performance**:
- Use `@st.cache_data` for expensive operations
- Limit result set size with SQL `LIMIT` clauses
- Consider pagination for large result sets
