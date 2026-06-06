---
name: harvard-art-museums-etl-pipeline
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API using Python ETL, SQL analytics, and Streamlit dashboards
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard museum API
  - set up SQL analytics for art museum artifacts
  - visualize Harvard art collection data with Streamlit
  - extract and transform museum artifact metadata
  - create interactive dashboards for museum data analytics
  - build a data pipeline from Harvard Art Museums API
  - analyze art museum collections with SQL queries
---

# Harvard Art Museums ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics dashboards via Streamlit.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts nested JSON, transforms to relational format, loads into SQL databases
- **SQL Database**: Stores artifacts in normalized tables (metadata, media, colors)
- **Analytics**: 20+ predefined SQL queries for artifact insights
- **Visualization**: Interactive Plotly dashboards in Streamlit

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL or TiDB Cloud database
```

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="your_database_name"
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Store in environment variable or `.env` file

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Database Schema

The ETL pipeline creates three relational tables:

### artifactmetadata
```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    technique VARCHAR(500)
);
```

### artifactmedia
```sql
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    primaryimageurl VARCHAR(1000),
    totalpageviews INT,
    totaluniquepageviews INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

### artifactcolors
```sql
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Code Examples

### Extract Data from API

```python
import requests
import os

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
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=50, page=1)
artifacts = data.get('records', [])
```

### Transform Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata table format"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:255],
            'period': artifact.get('period', 'Unknown')[:255],
            'century': artifact.get('century', 'Unknown')[:255],
            'classification': artifact.get('classification', 'Unknown')[:255],
            'department': artifact.get('department', 'Unknown')[:255],
            'dated': artifact.get('dated', 'Unknown')[:255],
            'technique': artifact.get('technique', 'Unknown')[:500]
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Transform media/image data"""
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        media = {
            'objectid': objectid,
            'primaryimageurl': artifact.get('primaryimageurl', '')[:1000],
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Transform color data (nested structure)"""
    color_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'objectid': objectid,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            color_list.append(color_record)
    
    return pd.DataFrame(color_list)
```

### Load Data to SQL

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL/TiDB connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df_metadata, connection):
    """Batch insert metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, period, century, classification, department, dated, technique)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    return cursor.rowcount

def load_media(df_media, connection):
    """Load media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (objectid, primaryimageurl, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    return cursor.rowcount

def load_colors(df_colors, connection):
    """Load color data"""
    if df_colors.empty:
        return 0
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_colors.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    return cursor.rowcount
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=5, artifacts_per_page=100):
    """Complete ETL pipeline execution"""
    api_key = os.getenv('HARVARD_API_KEY')
    connection = create_database_connection()
    
    if not connection:
        print("Failed to connect to database")
        return
    
    total_loaded = 0
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = fetch_artifacts(api_key, size=artifacts_per_page, page=page)
        artifacts = data.get('records', [])
        
        if not artifacts:
            break
        
        # Transform
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
        
        # Load
        metadata_count = load_metadata(df_metadata, connection)
        media_count = load_media(df_media, connection)
        colors_count = load_colors(df_colors, connection)
        
        total_loaded += metadata_count
        print(f"Loaded {metadata_count} metadata, {media_count} media, {colors_count} colors")
    
    connection.close()
    print(f"ETL Complete. Total artifacts: {total_loaded}")

# Execute
run_etl_pipeline(num_pages=10, artifacts_per_page=50)
```

## SQL Analytics Examples

### Sample Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != 'Unknown'
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Most common colors
query_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15;
"""

# Query 4: Department distribution
query_department = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Query 5: Artifacts with highest pageviews
query_pageviews = """
SELECT am.title, am.culture, am.century, 
       amed.totalpageviews, amed.primaryimageurl
FROM artifactmetadata am
JOIN artifactmedia amed ON am.objectid = amed.objectid
WHERE amed.totalpageviews > 0
ORDER BY amed.totalpageviews DESC
LIMIT 20;
"""
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

def create_visualization(df, x_col, y_col, title):
    """Create Plotly bar chart"""
    fig = px.bar(
        df, 
        x=x_col, 
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title(), 
                y_col: y_col.replace('_', ' ').title()}
    )
    return fig

# Streamlit App
st.title("Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
query_options = {
    "Top Cultures": query_culture,
    "Artifacts by Century": query_century,
    "Color Analysis": query_colors,
    "Department Distribution": query_department,
    "Top Viewed Artifacts": query_pageviews
}

selected_query = st.sidebar.selectbox("Select Analysis", list(query_options.keys()))

# Execute and display
connection = create_database_connection()
query = query_options[selected_query]

df_results = execute_query(query, connection)

st.subheader(f"Results: {selected_query}")
st.dataframe(df_results)

# Auto-visualization
if len(df_results.columns) >= 2:
    x_col = df_results.columns[0]
    y_col = df_results.columns[1]
    
    fig = create_visualization(df_results, x_col, y_col, selected_query)
    st.plotly_chart(fig, use_container_width=True)

connection.close()
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=gateway01.us-west-2.prod.aws.tidbcloud.com
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts

# ETL Settings
ARTIFACTS_PER_PAGE=100
MAX_PAGES=50
```

### Load Environment Variables

```python
from dotenv import load_dotenv
import os

load_dotenv()

# Access variables
api_key = os.getenv('HARVARD_API_KEY')
db_host = os.getenv('DB_HOST')
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, page, delay=1):
    """Fetch with delay to respect API limits"""
    data = fetch_artifacts(api_key, page=page)
    time.sleep(delay)  # 1 second delay between requests
    return data
```

### Error Handling in ETL

```python
def safe_etl_pipeline(num_pages=10):
    """ETL with error handling"""
    api_key = os.getenv('HARVARD_API_KEY')
    connection = create_database_connection()
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            artifacts = data.get('records', [])
            
            df_metadata = transform_artifact_metadata(artifacts)
            load_metadata(df_metadata, connection)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            continue  # Skip to next page
    
    connection.close()
```

### Incremental Data Loading

```python
def get_max_objectid(connection):
    """Get latest objectid from database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(connection):
    """Only load new artifacts"""
    max_id = get_max_objectid(connection)
    # Fetch only artifacts with objectid > max_id
    # Implementation depends on API filtering capabilities
```

## Troubleshooting

### API Key Issues

```python
# Verify API key works
response = requests.get(
    f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
)
print(f"Status: {response.status_code}")
print(f"Response: {response.json()}")
```

### Database Connection Errors

```python
# Test connection
try:
    conn = create_database_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    print("Database connection successful")
    conn.close()
except Exception as e:
    print(f"Connection failed: {e}")
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default='Unknown'):
    """Safely extract values from nested JSON"""
    value = dictionary.get(key, default)
    return value if value else default

# Usage
culture = safe_get(artifact, 'culture', 'Unknown')
```

### Memory Issues with Large Datasets

```python
# Process in smaller batches
def batch_etl(total_pages=100, batch_size=10):
    """Process in batches to manage memory"""
    for batch_start in range(1, total_pages, batch_size):
        batch_end = min(batch_start + batch_size, total_pages)
        run_etl_pipeline(num_pages=batch_size, start_page=batch_start)
        print(f"Completed batch {batch_start}-{batch_end}")
```

This skill enables AI agents to help developers build complete data engineering pipelines with museum artifact data, from API extraction to interactive analytics dashboards.
