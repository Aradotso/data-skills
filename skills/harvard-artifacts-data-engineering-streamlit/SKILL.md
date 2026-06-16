---
name: harvard-artifacts-data-engineering-streamlit
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit dashboards
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create an ETL pipeline for museum artifact data
  - set up Harvard artifacts analytics dashboard
  - extract and analyze Harvard Art Museums data
  - build a Streamlit app for museum data visualization
  - implement SQL analytics for Harvard artifacts collection
  - create museum artifact data engineering workflow
  - visualize Harvard Art Museums API data
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

- **Data Collection**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color information into relational SQL databases
- **SQL Analytics**: Runs 20+ predefined analytical queries for insights on artifacts by culture, century, department, and media
- **Visualization**: Creates interactive Plotly dashboards in Streamlit for exploring query results
- **Database Management**: Manages MySQL/TiDB Cloud schemas with proper foreign key relationships

## Installation

### Prerequisites

```bash
# Required Python packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

Or use requirements file:

```bash
pip install -r requirements.txt
```

### Environment Setup

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    period VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    rank INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=50, page=1)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def extract_artifact_metadata(records):
    """Extract metadata from API response"""
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title', 'Unknown'),
            'culture': record.get('culture', 'Unknown'),
            'century': record.get('century', 'Unknown'),
            'department': record.get('department', 'Unknown'),
            'classification': record.get('classification', 'Unknown'),
            'period': record.get('period', 'Unknown'),
            'dated': record.get('dated', 'Unknown'),
            'url': record.get('url', ''),
            'rank': record.get('rank', 0)
        })
    
    return pd.DataFrame(metadata)

def extract_artifact_media(records):
    """Extract media/images from API response"""
    media = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'media_url': img.get('baseimageurl', '')
            })
    
    return pd.DataFrame(media)

def extract_artifact_colors(records):
    """Extract color information from API response"""
    colors = []
    
    for record in records:
        artifact_id = record.get('id')
        color_data = record.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('color', ''),
                'color_percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)

# Transform and load
def load_to_database(df, table_name, connection):
    """Load DataFrame to MySQL table"""
    cursor = connection.cursor()
    
    if table_name == 'artifactmetadata':
        query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, department, classification, period, dated, url, rank)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
        """
        data = [tuple(row) for row in df.values]
    
    elif table_name == 'artifactmedia':
        query = """
        INSERT INTO artifactmedia (artifact_id, media_type, media_url)
        VALUES (%s, %s, %s)
        """
        data = [tuple(row) for row in df.values]
    
    elif table_name == 'artifactcolors':
        query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_name, color_percent)
        VALUES (%s, %s, %s, %s)
        """
        data = [tuple(row) for row in df.values]
    
    cursor.executemany(query, data)
    connection.commit()
    cursor.close()
```

### 3. Database Connection

```python
import mysql.connector
from mysql.connector import Error
import os

def create_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        if connection.is_connected():
            print("Successfully connected to database")
            return connection
            
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def execute_query(connection, query):
    """Execute SELECT query and return results"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)
```

### 4. Analytical SQL Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts with Media": """
        SELECT 
            am.department,
            COUNT(DISTINCT am.id) as total_artifacts,
            COUNT(DISTINCT med.artifact_id) as with_media,
            ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
        FROM artifactmetadata am
        LEFT JOIN artifactmedia med ON am.id = med.artifact_id
        GROUP BY am.department
        ORDER BY media_percentage DESC
    """,
    
    "Top Colors Used": """
        SELECT 
            color_name,
            COUNT(*) as usage_count,
            ROUND(AVG(color_percent), 2) as avg_percent
        FROM artifactcolors
        WHERE color_name != ''
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Culture by Period": """
        SELECT culture, period, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown' AND period != 'Unknown'
        GROUP BY culture, period
        ORDER BY count DESC
        LIMIT 25
    """
}

def run_analytics(connection, query_name):
    """Run analytical query and return results"""
    query = ANALYTICAL_QUERIES.get(query_name)
    if query:
        return execute_query(connection, query)
    return pd.DataFrame()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    connection = create_connection()
    
    if not connection:
        st.error("Failed to connect to database. Check your configuration.")
        return
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input("Records to fetch", min_value=10, max_value=100, value=50)
    
    with col2:
        if st.button("Fetch & Load Data"):
            with st.spinner("Fetching data from API..."):
                data = fetch_artifacts(api_key, size=num_records, page=1)
                records = data['records']
                
                # Extract
                metadata_df = extract_artifact_metadata(records)
                media_df = extract_artifact_media(records)
                colors_df = extract_artifact_colors(records)
                
                # Load
                load_to_database(metadata_df, 'artifactmetadata', connection)
                load_to_database(media_df, 'artifactmedia', connection)
                load_to_database(colors_df, 'artifactcolors', connection)
                
                st.success(f"Loaded {len(records)} artifacts successfully!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_choice = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Running analysis..."):
            results = run_analytics(connection, query_choice)
            
            if not results.empty:
                st.subheader("Results")
                st.dataframe(results, use_container_width=True)
                
                # Visualization
                if len(results.columns) >= 2:
                    fig = px.bar(results, 
                                 x=results.columns[0], 
                                 y=results.columns[1],
                                 title=query_choice)
                    st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing Large Datasets

```python
def batch_fetch_artifacts(api_key, total_records=1000, batch_size=100):
    """Fetch artifacts in batches with pagination"""
    all_records = []
    pages = (total_records // batch_size) + 1
    
    for page in range(1, pages + 1):
        print(f"Fetching page {page}/{pages}")
        data = fetch_artifacts(api_key, size=batch_size, page=page)
        all_records.extend(data['records'])
        
        # Rate limiting
        time.sleep(0.5)
    
    return all_records[:total_records]
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, connection, batch_size=50):
    """ETL pipeline with error handling"""
    try:
        # Extract
        data = fetch_artifacts(api_key, size=batch_size)
        records = data['records']
        
        # Transform
        metadata_df = extract_artifact_metadata(records)
        media_df = extract_artifact_media(records)
        colors_df = extract_artifact_colors(records)
        
        # Validate
        assert not metadata_df.empty, "No metadata extracted"
        
        # Load with transaction
        connection.start_transaction()
        load_to_database(metadata_df, 'artifactmetadata', connection)
        load_to_database(media_df, 'artifactmedia', connection)
        load_to_database(colors_df, 'artifactcolors', connection)
        connection.commit()
        
        return True
        
    except Exception as e:
        connection.rollback()
        print(f"ETL Error: {e}")
        return False
```

### Custom Query Builder

```python
def build_custom_query(table='artifactmetadata', 
                       group_by='culture', 
                       filters=None, 
                       limit=20):
    """Build custom analytical query"""
    query = f"""
        SELECT {group_by}, COUNT(*) as count
        FROM {table}
    """
    
    if filters:
        conditions = " AND ".join([f"{k}='{v}'" for k, v in filters.items()])
        query += f" WHERE {conditions}"
    
    query += f"""
        GROUP BY {group_by}
        ORDER BY count DESC
        LIMIT {limit}
    """
    
    return query

# Usage
custom_query = build_custom_query(
    group_by='department',
    filters={'century': '19th century'},
    limit=10
)
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, size=100, page=1, max_retries=3):
    """Fetch with retry logic for rate limiting"""
    for attempt in range(max_retries):
        try:
            response = requests.get(
                "https://api.harvardartmuseums.org/object",
                params={'apikey': api_key, 'size': size, 'page': page}
            )
            response.raise_for_status()
            return response.json()
            
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def get_robust_connection(max_attempts=3):
    """Get database connection with retry logic"""
    for attempt in range(max_attempts):
        try:
            connection = create_connection()
            if connection and connection.is_connected():
                return connection
        except Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(2)
    
    raise Exception("Could not establish database connection")
```

### Memory-Efficient Processing

```python
def stream_process_artifacts(api_key, total_pages=10, batch_size=100):
    """Process artifacts in streaming fashion to save memory"""
    connection = create_connection()
    
    for page in range(1, total_pages + 1):
        # Fetch batch
        data = fetch_artifacts(api_key, size=batch_size, page=page)
        records = data['records']
        
        # Process and load immediately
        metadata_df = extract_artifact_metadata(records)
        load_to_database(metadata_df, 'artifactmetadata', connection)
        
        # Clear memory
        del records, metadata_df
        
        print(f"Processed page {page}/{total_pages}")
    
    connection.close()
```

## Advanced Features

### Data Quality Checks

```python
def validate_data_quality(connection):
    """Run data quality checks"""
    checks = {
        "Duplicate IDs": """
            SELECT id, COUNT(*) as count
            FROM artifactmetadata
            GROUP BY id
            HAVING count > 1
        """,
        "Missing Cultures": """
            SELECT COUNT(*) as missing_count
            FROM artifactmetadata
            WHERE culture IS NULL OR culture = 'Unknown'
        """,
        "Orphaned Media": """
            SELECT COUNT(*) as orphan_count
            FROM artifactmedia m
            LEFT JOIN artifactmetadata a ON m.artifact_id = a.id
            WHERE a.id IS NULL
        """
    }
    
    results = {}
    for check_name, query in checks.items():
        df = execute_query(connection, query)
        results[check_name] = df
    
    return results
```

This skill provides comprehensive knowledge for building data engineering pipelines with the Harvard Art Museums API, implementing ETL workflows, and creating analytics dashboards with Streamlit.
