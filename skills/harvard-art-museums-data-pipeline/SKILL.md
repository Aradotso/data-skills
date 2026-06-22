---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL workflow for museum artifact data
  - create analytics dashboard for Harvard artifacts
  - query and visualize museum collection data
  - integrate Harvard Art Museums API with SQL database
  - build Streamlit app for artifact analytics
  - extract and transform museum metadata
  - analyze art collection data with Python
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for extracting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL patterns, SQL analytics, and interactive visualization using Streamlit.

## What It Does

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into relational database tables
- **SQL Storage**: Stores structured data in MySQL/TiDB with proper relationships
- **Analytics**: Executes 20+ predefined analytical queries
- **Visualization**: Generates interactive dashboards with Plotly

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from https://www.harvardartmuseums.org/collections/api

Set up environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

Configure your MySQL/TiDB connection:

```python
import os
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    description TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
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

def collect_all_artifacts(api_key, max_pages=10):
    """Collect artifacts with pagination"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        
        # Check if more pages exist
        if page >= data.get('info', {}).get('pages', 0):
            break
    
    return all_artifacts
```

### 2. ETL Transform Function

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform raw API data into structured DataFrames"""
    
    # Metadata extraction
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'description': artifact.get('description')
        })
        
        # Extract media
        for media in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': media.get('baseimageurl'),
                'iiifbaseuri': media.get('iiifbaseuri')
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 3. Loading Data to SQL

```python
def load_to_database(df_metadata, df_media, df_colors, connection):
    """Load transformed data into MySQL database"""
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, dated, url, description)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        cursor.execute(query, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_type, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 4. Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(DISTINCT a.id) as total_artifacts
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit dashboard"""
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    if st.sidebar.button("Connect to Database"):
        try:
            connection = mysql.connector.connect(**db_config)
            st.sidebar.success("Connected to database!")
            st.session_state.connection = connection
        except Exception as e:
            st.sidebar.error(f"Connection failed: {e}")
    
    # ETL Pipeline
    st.header("ETL Pipeline")
    num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = collect_all_artifacts(api_key, max_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifact_data(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(df_metadata, df_media, df_colors, 
                           st.session_state.connection)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("Analytics")
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        df_result = execute_query(st.session_state.connection, query)
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig)

if __name__ == "__main__":
    create_dashboard()
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

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """Fetch data with rate limiting"""
    time.sleep(delay)
    return fetch_artifacts(api_key, page)
```

### Batch Processing

```python
def batch_insert(cursor, table, columns, data, batch_size=100):
    """Insert data in batches for better performance"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i+batch_size]
        placeholders = ','.join(['%s'] * len(columns))
        query = f"INSERT INTO {table} ({','.join(columns)}) VALUES ({placeholders})"
        cursor.executemany(query, batch)
```

### Error Handling

```python
def safe_api_call(api_key, page, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

## Troubleshooting

**API Rate Limit Errors**: Add delays between requests using `time.sleep()` or implement exponential backoff.

**Database Connection Issues**: Verify credentials in environment variables and ensure database server is accessible.

**Memory Issues with Large Datasets**: Process data in smaller batches instead of loading everything into memory.

**Empty Query Results**: Check that ETL pipeline has run successfully and data exists in tables.

**Streamlit Session State**: Use `st.session_state` to persist database connections across reruns.
