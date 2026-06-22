---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up Harvard artifacts data engineering application
  - create analytics dashboard for museum artifact data
  - fetch and analyze Harvard Art Museums API data
  - build Streamlit app for artifact collection analytics
  - implement SQL analytics for museum data
  - design ETL for art museum collections
  - visualize Harvard artifacts with Plotly
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering application that demonstrates real-world ETL pipelines for museum data. It extracts data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics dashboards using Streamlit.

**Architecture:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
# Create a .env file or export directly
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os

# Load API key from environment
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### Database Configuration

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    provenance TEXT,
    description TEXT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    base_image_url VARCHAR(1000),
    primary_image BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    artifacts = []
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'size': page_size,
        'page': 1
    }
    
    while len(artifacts) < num_records:
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) < page_size:
                break
                
            params['page'] += 1
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Process and Structure Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw artifact data into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'provenance': artifact.get('provenance', ''),
            'description': artifact.get('description', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', ''),
            'url': artifact.get('url', ''),
            'creditline': artifact.get('creditline', ''),
            'accession_number': artifact.get('accessionyear', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        primary_image_id = artifact.get('primaryimageurl')
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl', ''),
                'base_image_url': image.get('iiifbaseuri', ''),
                'primary_image': image.get('baseimageurl') == primary_image_id
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color data from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Insert Data into SQL

```python
def load_metadata(df, connection):
    """
    Batch insert metadata into SQL database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, 
         dated, provenance, description, technique, medium, dimensions, 
         url, creditline, accession_number)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    values = df.values.tolist()
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()

def load_media(df, connection):
    """
    Insert media data into SQL database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, image_url, base_image_url, primary_image)
        VALUES (%s, %s, %s, %s)
    """
    
    values = df.values.tolist()
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()

def load_colors(df, connection):
    """
    Insert color data into SQL database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    values = df.values.tolist()
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, db_config, num_records=100):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data into database...")
    connection = mysql.connector.connect(**db_config)
    
    load_metadata(metadata_df, connection)
    load_media(media_df, connection)
    load_colors(colors_df, connection)
    
    connection.close()
    print("ETL pipeline completed successfully!")
```

## Analytics Queries

### Sample SQL Analytics Queries

```python
# Query 1: Top 10 cultures by artifact count
query_top_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts with images vs without
query_image_availability = """
    SELECT 
        CASE 
            WHEN media_count > 0 THEN 'With Images'
            ELSE 'No Images'
        END as image_status,
        COUNT(*) as count
    FROM (
        SELECT m.id, COUNT(im.media_id) as media_count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia im ON m.id = im.artifact_id
        GROUP BY m.id
    ) as image_counts
    GROUP BY image_status
"""

# Query 3: Most common colors across artifacts
query_top_colors = """
    SELECT color, COUNT(*) as frequency
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 10
"""

# Query 4: Artifacts by century
query_by_century = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY artifact_count DESC
    LIMIT 15
"""

# Query 5: Department distribution
query_by_department = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""
```

### Execute Analytics Queries

```python
def execute_analytics_query(query, connection):
    """
    Execute SQL query and return results as DataFrame
    """
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

### Basic Streamlit App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Sidebar for navigation
st.sidebar.title("Navigation")
page = st.sidebar.radio("Go to", ["ETL Pipeline", "Analytics Dashboard", "Visualizations"])

if page == "ETL Pipeline":
    st.title("ETL Pipeline")
    
    num_records = st.number_input("Number of records to fetch", min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline(
                    os.getenv('HARVARD_API_KEY'),
                    db_config,
                    num_records
                )
                st.success("ETL pipeline completed successfully!")
            except Exception as e:
                st.error(f"Error: {str(e)}")

elif page == "Analytics Dashboard":
    st.title("SQL Analytics Dashboard")
    
    # Query selector
    queries = {
        "Top Cultures": query_top_cultures,
        "Image Availability": query_image_availability,
        "Top Colors": query_top_colors,
        "By Century": query_by_century,
        "By Department": query_by_department
    }
    
    selected_query = st.selectbox("Select Analytics Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = mysql.connector.connect(**db_config)
        df = execute_analytics_query(queries[selected_query], connection)
        connection.close()
        
        st.dataframe(df)

elif page == "Visualizations":
    st.title("Data Visualizations")
    
    connection = mysql.connector.connect(**db_config)
    
    # Culture distribution chart
    df_cultures = execute_analytics_query(query_top_cultures, connection)
    fig = px.bar(df_cultures, x='culture', y='artifact_count', 
                 title='Top 10 Cultures by Artifact Count')
    st.plotly_chart(fig, use_container_width=True)
    
    # Color distribution chart
    df_colors = execute_analytics_query(query_top_colors, connection)
    fig = px.pie(df_colors, names='color', values='frequency',
                 title='Most Common Colors in Artifacts')
    st.plotly_chart(fig, use_container_width=True)
    
    connection.close()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Handling API Rate Limits

```python
import time
from functools import wraps

def rate_limit(max_per_second):
    """
    Decorator to limit API calls
    """
    min_interval = 1.0 / max_per_second
    
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        
        return wrapper
    return decorator

@rate_limit(2)  # 2 requests per second
def fetch_artifact_by_id(artifact_id, api_key):
    url = f"https://api.harvardartmuseums.org/object/{artifact_id}"
    response = requests.get(url, params={'apikey': api_key})
    return response.json()
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, db_config, num_records=100):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        artifacts = fetch_artifacts(api_key, num_records)
        
        if not artifacts:
            raise ValueError("No artifacts fetched from API")
        
        metadata_df = transform_artifact_metadata(artifacts)
        media_df = transform_artifact_media(artifacts)
        colors_df = transform_artifact_colors(artifacts)
        
        connection = mysql.connector.connect(**db_config)
        
        try:
            load_metadata(metadata_df, connection)
            load_media(media_df, connection)
            load_colors(colors_df, connection)
            connection.commit()
        except Exception as db_error:
            connection.rollback()
            raise Exception(f"Database error: {str(db_error)}")
        finally:
            connection.close()
        
        return True
        
    except requests.RequestException as api_error:
        print(f"API Error: {str(api_error)}")
        return False
    except Exception as e:
        print(f"ETL Error: {str(e)}")
        return False
```

## Troubleshooting

### API Connection Issues

```python
# Test API connection
def test_api_connection(api_key):
    test_url = "https://api.harvardartmuseums.org/object"
    response = requests.get(test_url, params={'apikey': api_key, 'size': 1})
    
    if response.status_code == 200:
        print("API connection successful")
        return True
    elif response.status_code == 401:
        print("Invalid API key")
        return False
    else:
        print(f"API error: {response.status_code}")
        return False
```

### Database Connection Issues

```python
# Test database connection
def test_db_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        connection.close()
        print("Database connection successful")
        return True
    except mysql.connector.Error as err:
        print(f"Database connection failed: {err}")
        return False
```

### Data Quality Checks

```python
def validate_data_quality(df, table_name):
    """
    Perform data quality checks before loading
    """
    print(f"\n--- {table_name} Quality Report ---")
    print(f"Total records: {len(df)}")
    print(f"Null values:\n{df.isnull().sum()}")
    print(f"Duplicate records: {df.duplicated().sum()}")
    
    # Check for required fields
    if table_name == "metadata":
        required_fields = ['id', 'title']
        for field in required_fields:
            null_count = df[field].isnull().sum()
            if null_count > 0:
                print(f"Warning: {null_count} null values in required field '{field}'")
```

## Key Takeaways

- Always use environment variables for credentials
- Implement rate limiting for API calls
- Use batch inserts for better database performance
- Add comprehensive error handling and logging
- Validate data quality before loading
- Design normalized database schemas with proper foreign keys
- Cache frequently accessed data to reduce API calls
