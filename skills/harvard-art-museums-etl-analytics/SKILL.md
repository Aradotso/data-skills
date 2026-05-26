---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data to SQL
  - visualize art museum collection data with Streamlit
  - set up data pipeline from Harvard Art Museums API
  - analyze artifact metadata using SQL queries
  - build museum data engineering project
  - create interactive art collection analytics
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides a complete data pipeline:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
2. **Transform**: Convert nested JSON to relational table structures
3. **Load**: Batch insert into MySQL/TiDB Cloud databases
4. **Analyze**: Execute 20+ predefined SQL analytical queries
5. **Visualize**: Display results via interactive Streamlit dashboards with Plotly charts

The application creates three main tables: `artifactmetadata`, `artifactmedia`, and `artifactcolors` with proper foreign key relationships.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

**Required packages:**
- streamlit
- pandas
- requests
- mysql-connector-python (or pymysql)
- plotly

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

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
```

## Core Components

### 1. ETL Pipeline

#### Extract Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
    """
    artifacts = []
    params = {
        'apikey': api_key,
        'size': page_size,
        'page': 1
    }
    
    while len(artifacts) < num_records:
        response = requests.get(BASE_URL, params=params)
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            # Check if more pages available
            if data['info']['next']:
                params['page'] += 1
            else:
                break
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

#### Transform to Relational Format

```python
def transform_artifacts(artifacts):
    """
    Transform nested JSON to relational table structures
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Artifact metadata
        metadata = {
            'object_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Media records
        for image in artifact.get('images', []):
            media = {
                'object_id': artifact.get('id'),
                'image_id': image.get('imageid'),
                'base_url': image.get('baseimageurl'),
                'format': image.get('format'),
                'width': image.get('width'),
                'height': image.get('height')
            }
            media_records.append(media)
        
        # Color records
        for color in artifact.get('colors', []):
            color_rec = {
                'object_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_rec)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

#### Load to Database

```python
def create_tables(conn):
    """
    Create database tables with proper schema
    """
    cursor = conn.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            technique VARCHAR(300),
            dated VARCHAR(200),
            url VARCHAR(500)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    conn.commit()
    cursor.close()

def batch_insert(conn, df, table_name):
    """
    Perform batch insert for better performance
    """
    cursor = conn.cursor()
    
    if len(df) == 0:
        return
    
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    cursor.executemany(query, df.values.tolist())
    conn.commit()
    cursor.close()
```

### 2. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.object_id) as total_artifacts,
            COUNT(DISTINCT media.object_id) as artifacts_with_images,
            ROUND(COUNT(DISTINCT media.object_id) * 100.0 / COUNT(DISTINCT am.object_id), 2) as percentage
        FROM artifactmetadata am
        LEFT JOIN artifactmedia media ON am.object_id = media.object_id
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT 
            am.title,
            am.culture,
            COUNT(media.image_id) as image_count
        FROM artifactmetadata am
        JOIN artifactmedia media ON am.object_id = media.object_id
        GROUP BY am.object_id, am.title, am.culture
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_query(conn, query):
    """
    Execute SQL query and return DataFrame
    """
    cursor = conn.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 3. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for data collection
    st.sidebar.header("Data Collection")
    num_records = st.sidebar.number_input("Number of records to fetch", 
                                          min_value=10, 
                                          max_value=1000, 
                                          value=100)
    
    if st.sidebar.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(API_KEY, num_records)
            
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            
        with st.spinner("Loading to database..."):
            conn = mysql.connector.connect(**db_config)
            create_tables(conn)
            batch_insert(conn, df_metadata, 'artifactmetadata')
            batch_insert(conn, df_media, 'artifactmedia')
            batch_insert(conn, df_colors, 'artifactcolors')
            conn.close()
            
        st.success(f"Successfully loaded {len(artifacts)} artifacts!")
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        results = execute_query(conn, ANALYTICS_QUERIES[query_name])
        conn.close()
        
        st.dataframe(results)
        
        # Auto-visualization
        if len(results.columns) >= 2:
            fig = px.bar(results, 
                        x=results.columns[0], 
                        y=results.columns[1],
                        title=query_name)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, delay=0.5):
    """
    Add delay between API requests to avoid rate limiting
    """
    artifacts = []
    for page in range(1, 11):
        params = {'apikey': api_key, 'page': page, 'size': 100}
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            artifacts.extend(response.json()['records'])
            time.sleep(delay)  # Rate limiting
        else:
            break
    
    return artifacts
```

### Error Handling in ETL

```python
def safe_transform(artifacts):
    """
    Transform with error handling for missing fields
    """
    metadata_records = []
    
    for artifact in artifacts:
        try:
            metadata = {
                'object_id': artifact.get('id', 0),
                'title': artifact.get('title', 'Unknown')[:500],
                'culture': artifact.get('culture', 'Unknown')[:200],
                'century': artifact.get('century', 'Unknown')[:100]
            }
            metadata_records.append(metadata)
        except Exception as e:
            print(f"Error processing artifact {artifact.get('id')}: {e}")
            continue
    
    return pd.DataFrame(metadata_records)
```

## Troubleshooting

### API Key Issues

**Problem**: 401 Unauthorized error
**Solution**: Verify API key is correctly set in environment variables

```python
import os
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY environment variable not set")
```

### Database Connection Errors

**Problem**: Can't connect to database
**Solution**: Check network access and credentials

```python
try:
    conn = mysql.connector.connect(**db_config)
    print("Database connection successful")
except mysql.connector.Error as e:
    print(f"Database error: {e}")
    print("Check host, user, password, and database name")
```

### Memory Issues with Large Datasets

**Problem**: Out of memory when fetching many records
**Solution**: Process in batches

```python
def fetch_in_batches(api_key, total_records, batch_size=100):
    """
    Fetch and process data in batches
    """
    for start in range(0, total_records, batch_size):
        batch = fetch_artifacts(api_key, batch_size)
        df_meta, df_media, df_colors = transform_artifacts(batch)
        
        conn = mysql.connector.connect(**db_config)
        batch_insert(conn, df_meta, 'artifactmetadata')
        batch_insert(conn, df_media, 'artifactmedia')
        batch_insert(conn, df_colors, 'artifactcolors')
        conn.close()
```

### Duplicate Key Errors

**Problem**: Primary key violations on re-runs
**Solution**: Use `INSERT IGNORE` or `ON DUPLICATE KEY UPDATE`

```python
query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
# or
query = f"""
    INSERT INTO {table_name} ({columns}) 
    VALUES ({placeholders})
    ON DUPLICATE KEY UPDATE title=VALUES(title)
"""
```
