---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data pipelines with Harvard Art Museums API, ETL processes, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with art museum data
  - analyze Harvard artifacts using SQL and Python
  - build a Streamlit analytics dashboard for museum data
  - extract and transform Harvard Art Museums API data
  - set up a data pipeline from API to SQL database
  - visualize museum artifact data with Plotly
  - create an end-to-end analytics app with museum collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end demonstration of real-world data engineering practices. It extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics through a Streamlit interface.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

The project showcases:
- API integration with pagination and rate limiting
- ETL pipeline development with nested JSON transformation
- Relational database design with foreign key relationships
- SQL analytics with 20+ predefined queries
- Interactive visualization using Plotly and Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

Create a `.env` file or configure environment variables:
```bash
export HARVARD_API_KEY='your_api_key_here'
```

### 2. Database Configuration

The app supports MySQL and TiDB Cloud. Set up your database connection:

```python
# Database configuration in your app
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

Environment variables:
```bash
export DB_HOST='your_database_host'
export DB_USER='your_database_user'
export DB_PASSWORD='your_database_password'
export DB_NAME='harvard_artifacts'
export DB_PORT='3306'
```

## Running the Application

```bash
streamlit run app.py
```

The app will launch at `http://localhost:8501`

## Database Schema

The ETL pipeline creates three main tables:

### artifactmetadata
```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    provenance TEXT,
    commentary TEXT
);
```

### artifactmedia
```sql
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    format VARCHAR(50),
    height INT,
    width INT,
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get('records', [])
total_pages = data.get('info', {}).get('pages', 1)
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform artifact data into metadata DataFrame
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'dated': artifact.get('dated', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance'),
            'commentary': artifact.get('commentary')
        })
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract and transform media/image data
    """
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for image in images:
            media_list.append({
                'objectid': objectid,
                'baseimageurl': image.get('baseimageurl', '')[:1000],
                'iiifbaseuri': image.get('iiifbaseuri', '')[:1000],
                'format': image.get('format', '')[:50],
                'height': image.get('height'),
                'width': image.get('width')
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract and transform color data
    """
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            colors_list.append({
                'objectid': objectid,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors_list)
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(config):
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(**config)
        return connection
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
        return None

def batch_insert_metadata(connection, df_metadata):
    """
    Batch insert artifact metadata
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, dated, classification, department, 
     technique, medium, dimensions, creditline, accessionyear, provenance, commentary)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df_metadata.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(connection, df_media):
    """
    Batch insert artifact media/images
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (objectid, baseimageurl, iiifbaseuri, format, height, width)
    VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_media.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_colors(connection, df_colors):
    """
    Batch insert artifact colors
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_colors.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

load_dotenv()

# Configuration
API_KEY = os.getenv('HARVARD_API_KEY')
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

def run_etl_pipeline(num_pages=5):
    """
    Execute complete ETL pipeline
    """
    connection = create_database_connection(DB_CONFIG)
    if not connection:
        return
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = fetch_artifacts(API_KEY, page=page, size=100)
        artifacts = data.get('records', [])
        
        # Transform
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
        
        # Load
        batch_insert_metadata(connection, df_metadata)
        batch_insert_media(connection, df_media)
        batch_insert_colors(connection, df_colors)
    
    connection.close()
    print("ETL pipeline completed")

# Run the pipeline
run_etl_pipeline(num_pages=5)
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Artifacts by century distribution
query_centuries = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Media availability analysis
query_media = """
SELECT 
    COUNT(DISTINCT am.objectid) as total_artifacts,
    COUNT(DISTINCT media.objectid) as artifacts_with_media,
    ROUND(COUNT(DISTINCT media.objectid) * 100.0 / COUNT(DISTINCT am.objectid), 2) as media_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.objectid = media.objectid;
"""

# Top colors in collection
query_colors = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY frequency DESC
LIMIT 10;
"""

# Department-wise classification
query_departments = """
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC;
"""
```

## Streamlit Visualization

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def execute_query(connection, query):
    """
    Execute SQL query and return DataFrame
    """
    return pd.read_sql(query, connection)

def create_bar_chart(df, x_col, y_col, title):
    """
    Create interactive Plotly bar chart
    """
    fig = px.bar(df, x=x_col, y=y_col, title=title)
    fig.update_layout(xaxis_tickangle=-45)
    return fig

# Streamlit app structure
st.title("Harvard Art Museums Analytics Dashboard")

# Database connection
connection = create_database_connection(DB_CONFIG)

# Query selector
queries = {
    "Top Cultures": query_cultures,
    "Century Distribution": query_centuries,
    "Media Availability": query_media,
    "Top Colors": query_colors,
    "Department Classification": query_departments
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    query = queries[selected_query]
    df_result = execute_query(connection, query)
    
    # Display table
    st.dataframe(df_result)
    
    # Create visualization
    if len(df_result.columns) >= 2:
        fig = create_bar_chart(
            df_result.head(15),
            df_result.columns[0],
            df_result.columns[1],
            selected_query
        )
        st.plotly_chart(fig)

connection.close()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, page, size=100, delay=0.5):
    """
    Fetch data with rate limiting to avoid API throttling
    """
    data = fetch_artifacts(api_key, page, size)
    time.sleep(delay)  # Delay between requests
    return data
```

### Error Handling and Retry Logic

```python
from requests.exceptions import RequestException

def fetch_with_retry(api_key, page, max_retries=3):
    """
    Fetch data with retry logic
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            print(f"Attempt {attempt + 1} failed, retrying...")
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Incremental Data Loading

```python
def get_max_objectid(connection):
    """
    Get the maximum objectid already in database
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load(connection, api_key):
    """
    Load only new artifacts not in database
    """
    max_id = get_max_objectid(connection)
    
    # Fetch artifacts with objectid > max_id
    # Implementation depends on API filtering capabilities
    pass
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` environment variable is set
- Verify API key is valid at Harvard Art Museums API portal
- Check API rate limits (typically 2500 requests/day for free tier)

### Database Connection Errors
- Verify all `DB_*` environment variables are correctly set
- Check database server is running and accessible
- Ensure database user has proper permissions (INSERT, SELECT, CREATE)
- For TiDB Cloud, verify SSL/TLS settings if required

### Data Type Errors
- Some fields may exceed VARCHAR limits; adjust schema as needed
- Handle NULL values appropriately in transformations
- Use `.fillna('')` or `.fillna(0)` for missing values

### Memory Issues with Large Datasets
- Process data in batches (default 100 records per page)
- Use `chunksize` parameter with pandas for large DataFrames
- Close database connections when not in use
- Clear intermediate DataFrames after loading

### Streamlit Performance
- Cache database connections with `@st.cache_resource`
- Cache query results with `@st.cache_data`
- Limit result sets for visualization (use LIMIT in queries)

```python
@st.cache_resource
def get_connection():
    return create_database_connection(DB_CONFIG)

@st.cache_data(ttl=3600)
def run_cached_query(query):
    conn = get_connection()
    return pd.read_sql(query, conn)
```
