---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data with Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create analytics dashboard for museum artifact data
  - extract and transform Harvard artifacts data
  - set up SQL database for art collection analysis
  - visualize Harvard museum collection with Streamlit
  - run analytical queries on artifact metadata
  - build data engineering pipeline for museum API
  - process Harvard Art Museums API pagination
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: 20+ predefined analytical queries for artifact analysis
- **Interactive Visualization**: Streamlit-based dashboard with Plotly charts
- **Database Design**: Relational schema with proper foreign key relationships

**Architecture**: API → ETL → SQL → Analytics → Visualization

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

### Required Dependencies

```python
# requirements.txt
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
mysql-connector-python>=8.0.0
plotly>=5.13.0
python-dotenv>=0.21.0
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("HARVARD_API_KEY")
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

## Database Schema

### Create Tables

```python
def create_tables(cursor):
    """Create relational tables for artifact data"""
    
    # Artifact Metadata
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            medium VARCHAR(500),
            creditline TEXT,
            division VARCHAR(255),
            contact VARCHAR(255),
            accessionyear INT
        )
    """)
    
    # Artifact Media
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            mediatype VARCHAR(100),
            mediaurl TEXT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            percentage FLOAT,
            hue VARCHAR(50),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """Fetch artifacts with pagination handling"""
    
    artifacts = []
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': page_size,
        'page': 1
    }
    
    while len(artifacts) < num_records:
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            
            if len(artifacts) >= num_records:
                artifacts = artifacts[:num_records]
                break
                
            params['page'] += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"API Error: {e}")
            break
            
    return artifacts
```

### Transform: Process JSON Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data into metadata DataFrame"""
    
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'creditline': artifact.get('creditline', ''),
            'division': artifact.get('division', 'Unknown'),
            'contact': artifact.get('contact', 'Unknown'),
            'accessionyear': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Extract media information"""
    
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'objectid': objectid,
                'mediatype': 'image',
                'mediaurl': img.get('baseimageurl', '')
            })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Extract color data"""
    
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            colors_list.append({
                'objectid': objectid,
                'color': color.get('color', 'Unknown'),
                'percentage': color.get('percent', 0.0),
                'hue': color.get('hue', 'Unknown')
            })
    
    return pd.DataFrame(colors_list)
```

### Load: Insert into Database

```python
def load_metadata(df, cursor):
    """Batch insert metadata"""
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, 
         department, dated, medium, creditline, division, contact, accessionyear)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE objectid=objectid
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)

def load_media(df, cursor):
    """Batch insert media data"""
    
    insert_query = """
        INSERT INTO artifactmedia (objectid, mediatype, mediaurl)
        VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)

def load_colors(df, cursor):
    """Batch insert color data"""
    
    insert_query = """
        INSERT INTO artifactcolors (objectid, color, percentage, hue)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
```

## Streamlit Application

### Complete ETL Workflow

```python
import streamlit as st
import mysql.connector
from dotenv import load_dotenv

load_dotenv()

def run_etl_pipeline():
    """Execute complete ETL pipeline"""
    
    st.header("ETL Pipeline")
    
    api_key = os.getenv("HARVARD_API_KEY")
    num_records = st.number_input("Number of records", 10, 1000, 100)
    
    if st.button("Run ETL"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(api_key, num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_metadata(artifacts)
            df_media = transform_media(artifacts)
            df_colors = transform_colors(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = mysql.connector.connect(**db_config)
            cursor = conn.cursor()
            
            create_tables(cursor)
            load_metadata(df_metadata, cursor)
            load_media(df_media, cursor)
            load_colors(df_colors, cursor)
            
            conn.commit()
            cursor.close()
            conn.close()
            st.success("Data loaded to database")
```

### Analytics Dashboard

```python
import plotly.express as px

def analytics_dashboard():
    """Display analytics queries and visualizations"""
    
    st.header("SQL Analytics Dashboard")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY century
        """,
        "Artifacts with Images": """
            SELECT 
                CASE WHEN mediaurl IS NOT NULL THEN 'With Images' ELSE 'No Images' END as media_status,
                COUNT(DISTINCT am.objectid) as count
            FROM artifactmetadata am
            LEFT JOIN artifactmedia ame ON am.objectid = ame.objectid
            GROUP BY media_status
        """,
        "Top Colors Used": """
            SELECT color, AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY avg_percentage DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        conn = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig)
```

## Common Patterns

### Rate-Limited API Calls

```python
def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Too many requests
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

### Incremental Data Loading

```python
def get_last_loaded_id(cursor):
    """Get the last loaded artifact ID"""
    
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, cursor):
    """Load only new artifacts"""
    
    last_id = get_last_loaded_id(cursor)
    
    params = {
        'apikey': api_key,
        'q': f'objectid:>{last_id}',
        'size': 100
    }
    
    response = requests.get(BASE_URL, params=params)
    new_artifacts = response.json().get('records', [])
    
    return new_artifacts
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
if not os.getenv("HARVARD_API_KEY"):
    st.error("API key not found. Set HARVARD_API_KEY environment variable.")
    st.stop()
```

### Database Connection Errors

```python
try:
    conn = mysql.connector.connect(**db_config)
    st.success("Database connected")
except mysql.connector.Error as err:
    st.error(f"Database error: {err}")
    st.info("Check DB_HOST, DB_USER, DB_PASSWORD, DB_NAME environment variables")
```

### Empty API Response

```python
if not artifacts:
    st.warning("No artifacts returned. Check API key and query parameters.")
    st.info("API endpoint: https://api.harvardartmuseums.org/object")
```

### Memory Issues with Large Datasets

```python
# Process in batches
BATCH_SIZE = 1000

for offset in range(0, total_records, BATCH_SIZE):
    batch = artifacts[offset:offset + BATCH_SIZE]
    df_batch = transform_metadata(batch)
    load_metadata(df_batch, cursor)
    conn.commit()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

The application provides an interactive interface for ETL pipeline execution, SQL query analytics, and data visualization of Harvard Art Museums collections.
