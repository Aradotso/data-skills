---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create a data engineering app with Harvard Art Museums API
  - set up artifact collection analytics with SQL
  - build a Streamlit dashboard for art museum data
  - implement ETL from Harvard API to MySQL
  - analyze art artifacts with SQL queries
  - create museum data visualization pipeline
  - extract and transform Harvard museum artifacts
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud
- Executes analytical SQL queries
- Visualizes results through interactive Streamlit dashboards

Architecture: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

## Configuration

### Environment Variables

Create a `.env` file or configure Streamlit secrets:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Database
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Streamlit Secrets (`.streamlit/secrets.toml`)

```toml
HARVARD_API_KEY = "your_api_key_here"
DB_HOST = "your_database_host"
DB_PORT = 3306
DB_USER = "your_username"
DB_PASSWORD = "your_password"
DB_NAME = "harvard_artifacts"
```

Get your Harvard Art Museums API key: https://www.harvardartmuseums.org/collections/api

## Database Schema

The application creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    description TEXT,
    provenance TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(100),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will launch at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, num_records=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    size = 100  # API limit per request
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': size,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_records.extend(records)
            
            if not records or len(all_records) >= num_records:
                break
            
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_records]
```

### 2. Data Transformation

```python
import pandas as pd

def transform_to_metadata(records):
    """Transform API records to metadata DataFrame"""
    metadata = []
    
    for record in records:
        metadata.append({
            'objectid': record.get('objectid'),
            'title': record.get('title', 'Unknown'),
            'culture': record.get('culture', 'Unknown'),
            'period': record.get('period', 'Unknown'),
            'century': record.get('century', 'Unknown'),
            'dated': record.get('dated', 'Unknown'),
            'department': record.get('department', 'Unknown'),
            'classification': record.get('classification', 'Unknown'),
            'medium': record.get('medium', 'Unknown'),
            'dimensions': record.get('dimensions', 'Unknown'),
            'creditline': record.get('creditline', 'Unknown'),
            'description': record.get('description', 'Unknown'),
            'provenance': record.get('provenance', 'Unknown'),
            'url': record.get('url', 'Unknown')
        })
    
    return pd.DataFrame(metadata)

def transform_to_media(records):
    """Transform API records to media DataFrame"""
    media = []
    
    for record in records:
        objectid = record.get('objectid')
        images = record.get('images', [])
        
        for image in images:
            media.append({
                'mediaid': image.get('imageid'),
                'objectid': objectid,
                'baseimageurl': image.get('baseimageurl', 'Unknown'),
                'format': image.get('format', 'Unknown'),
                'iiifbaseuri': image.get('iiifbaseuri', 'Unknown')
            })
    
    return pd.DataFrame(media)

def transform_to_colors(records):
    """Transform API records to colors DataFrame"""
    colors = []
    
    for record in records:
        objectid = record.get('objectid')
        color_data = record.get('colors', [])
        
        for color in color_data:
            colors.append({
                'objectid': objectid,
                'color': color.get('color', 'Unknown'),
                'spectrum': color.get('spectrum', 'Unknown'),
                'hue': color.get('hue', 'Unknown'),
                'percent': color.get('percent', 0)
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_connection(host, port, user, password, database):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=host,
            port=port,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def batch_insert_metadata(connection, df):
    """Batch insert metadata records"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, period, century, dated, department, 
     classification, medium, dimensions, creditline, description, provenance, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    records = df.values.tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    cursor.close()

def batch_insert_media(connection, df):
    """Batch insert media records"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (mediaid, objectid, baseimageurl, format, iiifbaseuri)
    VALUES (%s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE baseimageurl=VALUES(baseimageurl)
    """
    
    records = df.values.tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    cursor.close()

def batch_insert_colors(connection, df):
    """Batch insert color records"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.values.tolist()
    cursor.executemany(insert_query, records)
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
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.objectid IS NOT NULL THEN 'With Images' ELSE 'Without Images' END as media_status,
            COUNT(DISTINCT a.objectid) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY media_status
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification != 'Unknown'
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    if 'connection' not in st.session_state:
        st.session_state.connection = create_connection(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT')),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
    
    # ETL Section
    st.header("📥 ETL Pipeline")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, 
                                   value=100, step=10)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            records = fetch_artifacts(api_key, num_records)
            
        with st.spinner("Transforming data..."):
            df_metadata = transform_to_metadata(records)
            df_media = transform_to_media(records)
            df_colors = transform_to_colors(records)
            
        with st.spinner("Loading to database..."):
            batch_insert_metadata(st.session_state.connection, df_metadata)
            batch_insert_media(st.session_state.connection, df_media)
            batch_insert_colors(st.session_state.connection, df_colors)
            
        st.success(f"✅ Successfully loaded {num_records} records!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        df_result = execute_query(st.session_state.connection, query)
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, 
                        x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(api_key, db_config, num_records=100):
    """Complete ETL pipeline execution"""
    # Extract
    print("Extracting data from API...")
    records = fetch_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_to_metadata(records)
    df_media = transform_to_media(records)
    df_colors = transform_to_colors(records)
    
    # Load
    print("Loading to database...")
    connection = create_connection(**db_config)
    
    batch_insert_metadata(connection, df_metadata)
    batch_insert_media(connection, df_media)
    batch_insert_colors(connection, df_colors)
    
    connection.close()
    print(f"ETL complete! Loaded {num_records} artifacts.")
```

### Query and Visualize Pattern

```python
def query_and_visualize(connection, query, title):
    """Execute query and create visualization"""
    df = execute_query(connection, query)
    
    if len(df.columns) >= 2:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1], title=title)
        return df, fig
    
    return df, None
```

## Troubleshooting

### API Rate Limiting
If you hit API rate limits, add delays between requests:
```python
import time

time.sleep(0.5)  # Wait 500ms between requests
```

### Database Connection Issues
Verify credentials and network access:
```python
try:
    connection = create_connection(host, port, user, password, database)
    if connection.is_connected():
        print("Database connection successful")
except Error as e:
    print(f"Error: {e}")
```

### Missing Data Fields
Handle optional API fields gracefully:
```python
record.get('field_name', 'Unknown')  # Default to 'Unknown'
```

### Memory Issues with Large Datasets
Process in batches:
```python
batch_size = 100
for i in range(0, total_records, batch_size):
    batch = records[i:i+batch_size]
    process_batch(batch)
```

### Streamlit Session State
Reset connection if stale:
```python
if 'connection' in st.session_state:
    if not st.session_state.connection.is_connected():
        st.session_state.connection.reconnect()
```

## Key Use Cases

1. **Museum Data Analytics**: Analyze artifact distributions, classifications, and trends
2. **ETL Learning**: Understand real-world data pipeline implementation
3. **API Integration**: Learn pagination, rate limiting, and data extraction
4. **SQL Analytics**: Practice complex queries on relational data
5. **Data Visualization**: Create interactive dashboards with Plotly
6. **Streamlit Applications**: Build full-stack data apps with Python
