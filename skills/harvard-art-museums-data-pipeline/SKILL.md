---
name: harvard-art-museums-data-pipeline
description: ETL pipeline and analytics application for Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - build a data pipeline for museum artifacts
  - create ETL workflow with Harvard Art Museums API
  - analyze art collections using SQL and visualization
  - extract and transform museum data into relational database
  - build streamlit dashboard for art analytics
  - implement batch data ingestion from API to MySQL
  - query and visualize art museum metadata
  - design analytics for cultural artifact collections
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It implements ETL pipelines that extract artifact data, transform nested JSON into relational schemas, load into SQL databases, and visualize analytics through Streamlit dashboards. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Set up your API key and database credentials:

```bash
# Harvard Art Museums API
export HARVARD_API_KEY="your_api_key_here"

# MySQL/TiDB Connection
export DB_HOST="your_database_host"
export DB_PORT="3306"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
```

### Database Setup

```python
import mysql.connector
import os

# Establish connection
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

# Create tables
cursor = conn.cursor()

# Artifact metadata table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dimensions VARCHAR(500),
    credit_line TEXT,
    provenance TEXT,
    description TEXT,
    url VARCHAR(500),
    lastupdate DATETIME
)
""")

# Artifact media table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

# Artifact colors table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
cursor.close()
conn.close()
```

## API Integration

### Fetching Data from Harvard Art Museums API

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
data = fetch_artifacts(api_key, page=1, size=50)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Total pages: {data['info']['pages']}")
```

### Handling Pagination and Rate Limits

```python
import time

def fetch_all_artifacts(api_key, max_pages=10, delay=0.5):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data['records'])
            
            print(f"Fetched page {page}/{max_pages}")
            
            # Rate limiting
            time.sleep(delay)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract: Parse Artifact Data

```python
import pandas as pd

def extract_artifact_metadata(artifact):
    """
    Extract metadata from artifact JSON
    """
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title'),
        'culture': artifact.get('culture'),
        'century': artifact.get('century'),
        'classification': artifact.get('classification'),
        'department': artifact.get('department'),
        'dated': artifact.get('dated'),
        'period': artifact.get('period'),
        'medium': artifact.get('medium'),
        'technique': artifact.get('technique'),
        'dimensions': artifact.get('dimensions'),
        'credit_line': artifact.get('creditline'),
        'provenance': artifact.get('provenance'),
        'description': artifact.get('description'),
        'url': artifact.get('url'),
        'lastupdate': artifact.get('lastupdate')
    }

def extract_artifact_media(artifact):
    """
    Extract media/image information
    """
    media_list = []
    
    if 'images' in artifact and artifact['images']:
        for img in artifact['images']:
            media_list.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'width': img.get('width'),
                'height': img.get('height')
            })
    
    return media_list

def extract_artifact_colors(artifact):
    """
    Extract color information
    """
    color_list = []
    
    if 'colors' in artifact and artifact['colors']:
        for color in artifact['colors']:
            color_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return color_list
```

### Transform: Create DataFrames

```python
def transform_artifacts(artifacts):
    """
    Transform list of artifacts into normalized dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append(extract_artifact_metadata(artifact))
        
        # Extract media
        media_records.extend(extract_artifact_media(artifact))
        
        # Extract colors
        color_records.extend(extract_artifact_colors(artifact))
    
    # Create DataFrames
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    # Clean data
    df_metadata = df_metadata.fillna('')
    df_media = df_media.fillna('')
    df_colors = df_colors.fillna('')
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into Database

```python
from sqlalchemy import create_engine
import os

def load_to_database(df_metadata, df_media, df_colors):
    """
    Load dataframes into MySQL database using batch inserts
    """
    # Create SQLAlchemy engine
    connection_string = f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    engine = create_engine(connection_string)
    
    # Load data with batch inserts
    df_metadata.to_sql('artifactmetadata', engine, if_exists='append', index=False, method='multi', chunksize=1000)
    
    df_media.to_sql('artifactmedia', engine, if_exists='append', index=False, method='multi', chunksize=1000)
    
    df_colors.to_sql('artifactcolors', engine, if_exists='append', index=False, method='multi', chunksize=1000)
    
    print(f"Loaded {len(df_metadata)} artifacts, {len(df_media)} media records, {len(df_colors)} color records")

# Complete ETL pipeline
def run_etl_pipeline(api_key, max_pages=5):
    """
    Execute full ETL pipeline
    """
    print("Starting ETL pipeline...")
    
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(api_key, max_pages=max_pages)
    
    # Transform
    print("Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    
    # Load
    print("Loading data to database...")
    load_to_database(df_metadata, df_media, df_colors)
    
    print("ETL pipeline completed successfully!")
```

## SQL Analytics Queries

### Key Analytical Queries

```python
# Query 1: Artifacts by culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Department distribution
query_department = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Query 4: Media availability
query_media = """
SELECT 
    COUNT(DISTINCT am.artifact_id) as artifacts_with_images,
    COUNT(DISTINCT meta.id) as total_artifacts,
    ROUND(COUNT(DISTINCT am.artifact_id) * 100.0 / COUNT(DISTINCT meta.id), 2) as percentage
FROM artifactmetadata meta
LEFT JOIN artifactmedia am ON meta.id = am.artifact_id
"""

# Query 5: Top colors used
query_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percentage
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15
"""

# Query 6: Artifacts by classification and culture
query_classification = """
SELECT classification, culture, COUNT(*) as count
FROM artifactmetadata
WHERE classification != '' AND culture != ''
GROUP BY classification, culture
ORDER BY count DESC
LIMIT 20
"""

# Execute query
def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    connection_string = f"mysql+mysqlconnector://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    engine = create_engine(connection_string)
    
    df_result = pd.read_sql(query, engine)
    return df_result
```

## Streamlit Dashboard

### Basic Streamlit App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Select Analysis",
    ["ETL Pipeline", "SQL Analytics", "Visualizations"]
)

if page == "ETL Pipeline":
    st.header("Run ETL Pipeline")
    
    api_key = st.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    max_pages = st.slider("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline(api_key, max_pages)
                st.success("ETL pipeline completed successfully!")
            except Exception as e:
                st.error(f"Error: {e}")

elif page == "SQL Analytics":
    st.header("SQL Query Analytics")
    
    queries = {
        "Artifacts by Culture": query_culture,
        "Artifacts by Century": query_century,
        "Department Distribution": query_department,
        "Media Availability": query_media,
        "Top Colors": query_colors,
        "Classification Analysis": query_classification
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Executing query..."):
            df_result = execute_query(queries[selected_query])
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
                st.plotly_chart(fig, use_container_width=True)

elif page == "Visualizations":
    st.header("Interactive Visualizations")
    
    # Culture distribution
    df_culture = execute_query(query_culture)
    fig1 = px.bar(df_culture, x='culture', y='artifact_count', title="Top 10 Cultures")
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    df_colors = execute_query(query_colors)
    fig2 = px.pie(df_colors, names='color', values='usage_count', title="Color Distribution")
    st.plotly_chart(fig2, use_container_width=True)
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_update_timestamp(engine):
    """
    Get the most recent update timestamp from database
    """
    query = "SELECT MAX(lastupdate) as last_update FROM artifactmetadata"
    result = pd.read_sql(query, engine)
    return result['last_update'][0]

def fetch_updated_artifacts(api_key, last_update):
    """
    Fetch only artifacts updated since last ETL run
    """
    params = {
        'apikey': api_key,
        'size': 100,
        'q': f'lastupdate:>{last_update}'
    }
    response = requests.get("https://api.harvardartmuseums.org/object", params=params)
    return response.json()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, max_pages=5):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        logger.info("Starting ETL pipeline")
        artifacts = fetch_all_artifacts(api_key, max_pages)
        
        if not artifacts:
            logger.warning("No artifacts fetched")
            return
        
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        load_to_database(df_metadata, df_media, df_colors)
        
        logger.info("ETL completed successfully")
        
    except requests.exceptions.RequestException as e:
        logger.error(f"API request failed: {e}")
    except Exception as e:
        logger.error(f"ETL pipeline failed: {e}")
        raise
```

## Troubleshooting

### API Rate Limiting
- Add delays between requests using `time.sleep()`
- Implement exponential backoff for failed requests
- Check API key validity and quotas

### Database Connection Issues
- Verify environment variables are set correctly
- Test connection with `mysql.connector.connect()` before running ETL
- Ensure firewall rules allow database access

### Memory Issues with Large Datasets
- Use chunked reading: `pd.read_sql(query, engine, chunksize=1000)`
- Process data in batches rather than all at once
- Clear DataFrames after loading: `del df_metadata`

### Missing Data in Visualizations
- Check for NULL values: `df.isnull().sum()`
- Filter empty strings: `df[df['column'] != '']`
- Validate foreign key relationships before loading
