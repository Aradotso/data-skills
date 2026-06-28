---
name: harvard-artifacts-collection-data-engineering-analytics
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - set up data engineering pipeline with Streamlit
  - query Harvard Art Museums API and store in SQL
  - build interactive visualization for artifact collections
  - implement museum data warehouse with analytics
  - create artifact metadata analysis application
  - design SQL analytics for art museum collections
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering solution for collecting, transforming, storing, and visualizing artifact data from the Harvard Art Museums API. It demonstrates ETL pipelines, relational database design, SQL analytics, and interactive dashboards using Streamlit.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms into relational format, loads into MySQL/TiDB
- **SQL Analytics**: Executes 20+ predefined analytical queries on artifact metadata, media, and colors
- **Interactive Dashboards**: Visualizes query results with Plotly charts in a Streamlit interface
- **Database Design**: Implements normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

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

## Configuration

### API Key Setup

Obtain a free API key from [Harvard Art Museums](https://docs.harvardartmuseums.org/):

```python
# Reference environment variable
import os
API_KEY = os.getenv('HARVARD_API_KEY')
```

### Database Configuration

The application supports MySQL and TiDB Cloud:

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

### artifactmetadata Table

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
);
```

### artifactmedia Table

```sql
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### artifactcolors Table

```sql
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetching Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_records=100):
    """Fetch artifact data with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(all_artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            
            if not data.get('records'):
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts
```

### Transform: Processing Nested JSON

```python
def transform_artifacts(artifacts):
    """Transform raw API data into structured format"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dimensions': artifact.get('dimensions', 'Unknown'),
            'creditline': artifact.get('creditline', 'Unknown'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url', 'Unknown')
        }
        metadata_list.append(metadata)
        
        # Extract media
        for media in artifact.get('images', []):
            media_list.append({
                'artifact_id': artifact.get('id'),
                'media_url': media.get('baseimageurl'),
                'media_type': 'image'
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into SQL

```python
def load_to_database(df_metadata, df_media, df_colors, db_config):
    """Batch insert data into MySQL database"""
    connection = mysql.connector.connect(**db_config)
    cursor = connection.cursor()
    
    # Insert metadata
    insert_metadata = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         dated, medium, dimensions, creditline, accessionyear, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(insert_metadata, df_metadata.values.tolist())
    
    # Insert media
    insert_media = """
        INSERT INTO artifactmedia (artifact_id, media_url, media_type)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(insert_media, df_media.values.tolist())
    
    # Insert colors
    insert_colors = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(insert_colors, df_colors.values.tolist())
    
    connection.commit()
    cursor.close()
    connection.close()
```

## Running the Streamlit Application

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    num_records = st.number_input("Records to Fetch", 
                                  min_value=10, max_value=1000, 
                                  value=100)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_records)
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            load_to_database(df_meta, df_media, df_colors, db_config)
            st.success(f"Loaded {len(artifacts)} artifacts!")
```

### SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
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
    
    "Media Availability": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'Has Images' ELSE 'No Images' END as status,
            COUNT(*) as artifact_count
        FROM (
            SELECT m.id, COUNT(media.id) as media_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia media ON m.id = media.artifact_id
            GROUP BY m.id
        ) as subquery
        GROUP BY status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as occurrences, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department != 'Unknown'
        GROUP BY department
        ORDER BY count DESC
    """
}
```

### Executing Queries and Visualizing

```python
def execute_query(query, db_config):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# Analytics section
st.header("📊 Analytics Dashboard")

query_choice = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))

if st.button("Run Analysis"):
    query = ANALYTICAL_QUERIES[query_choice]
    
    with st.spinner("Running query..."):
        results = execute_query(query, db_config)
        
        # Display table
        st.subheader("Query Results")
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) >= 2:
            st.subheader("Visualization")
            fig = px.bar(results, 
                        x=results.columns[0], 
                        y=results.columns[1],
                        title=query_choice)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """Fetch with rate limiting to avoid API throttling"""
    time.sleep(delay)
    params = {'apikey': api_key, 'page': page}
    response = requests.get(base_url, params=params)
    return response.json()
```

### Handling Missing Data

```python
def clean_artifact_data(artifact):
    """Clean and validate artifact data"""
    return {
        'id': artifact.get('id', 0),
        'title': artifact.get('title') or 'Untitled',
        'culture': artifact.get('culture') or 'Unknown',
        'century': artifact.get('century') or 'Unknown',
        'accessionyear': artifact.get('accessionyear') or None
    }
```

### Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return db_pool.get_connection()
```

## Troubleshooting

### API Key Authentication Errors

```python
# Verify API key is valid
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': api_key, 'size': 1}
)

if response.status_code == 401:
    st.error("Invalid API key. Get one from https://docs.harvardartmuseums.org/")
```

### Database Connection Issues

```python
try:
    connection = mysql.connector.connect(**db_config)
    st.success("Database connected successfully!")
except mysql.connector.Error as err:
    st.error(f"Database error: {err}")
    st.info("Check DB_HOST, DB_USER, DB_PASSWORD environment variables")
```

### Empty Results Handling

```python
if results.empty:
    st.warning("No data found for this query")
else:
    st.dataframe(results)
```

### Memory Management for Large Datasets

```python
def fetch_in_batches(api_key, total_records, batch_size=100):
    """Fetch and process data in batches to manage memory"""
    for offset in range(0, total_records, batch_size):
        batch = fetch_artifacts(api_key, batch_size, offset)
        df_meta, df_media, df_colors = transform_artifacts(batch)
        load_to_database(df_meta, df_media, df_colors, db_config)
        yield len(batch)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

The application provides a complete end-to-end data engineering workflow from API integration to interactive analytics dashboards, demonstrating real-world ETL patterns and SQL analytics capabilities.
