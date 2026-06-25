---
name: harvard-artifacts-data-engineering-pipeline
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - build etl pipeline for harvard art museum data
  - create data engineering app with harvard artifacts api
  - setup streamlit analytics dashboard for art museum data
  - extract and transform harvard museum api data
  - analyze harvard art collections with sql queries
  - visualize museum artifact data with plotly
  - implement batch processing for art museum metadata
  - design relational database for museum artifacts
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that demonstrates real-world ETL practices using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through a Streamlit dashboard.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
# Create a .env file with:
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_db_host
# DB_USER=your_db_user
# DB_PASSWORD=your_db_password
# DB_NAME=your_db_name
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()
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

## Database Schema

### Create Tables

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    division VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    creditline TEXT,
    description TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    primaryimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    artifacts = []
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': 1
    }
    
    while len(artifacts) < num_records:
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            artifacts.extend(records)
            
            if len(records) < size:
                break
                
            params['page'] += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching data: {e}")
            break
    
    return artifacts[:num_records]
```

### Transform: Parse Nested JSON

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'description': artifact.get('description')
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('images'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into Database

```python
def load_to_database(df_metadata, df_media, df_colors, conn):
    """
    Batch insert dataframes into MySQL database
    """
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         technique, division, dated, url, creditline, description)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    metadata_values = df_metadata.values.tolist()
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia (artifact_id, baseimageurl, primaryimageurl)
        VALUES (%s, %s, %s)
    """
    
    media_values = df_media.values.tolist()
    if media_values:
        cursor.executemany(media_query, media_values)
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    colors_values = df_colors.values.tolist()
    if colors_values:
        cursor.executemany(colors_query, colors_values)
    
    conn.commit()
    cursor.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🏛️ Harvard Artifacts Collection Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Enter Harvard API Key", type="password")
    num_records = st.number_input("Number of records", 10, 1000, 100)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data..."):
            artifacts = fetch_artifacts(api_key, num_records)
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            
            conn = mysql.connector.connect(**db_config)
            load_to_database(df_metadata, df_media, df_colors, conn)
            conn.close()
            
            st.success(f"Loaded {len(df_metadata)} artifacts successfully!")
            st.dataframe(df_metadata.head())

if __name__ == "__main__":
    main()
```

## SQL Analytics Queries

### Predefined Analytics Queries

```python
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
        ORDER BY century
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' 
                 ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        RIGHT JOIN artifactmetadata ON artifactmedia.artifact_id = artifactmetadata.id
        GROUP BY image_status
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def show_sql_analytics():
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(ANALYTICS_QUERIES[query_name], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(
                df_result,
                x=df_result.columns[0],
                y=df_result.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)
```

## Visualization Patterns

### Interactive Charts with Plotly

```python
def create_culture_timeline(conn):
    """
    Create interactive timeline of artifacts by culture and century
    """
    query = """
        SELECT culture, century, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND century IS NOT NULL
        GROUP BY culture, century
        ORDER BY century, count DESC
    """
    
    df = pd.read_sql(query, conn)
    
    fig = px.scatter(
        df,
        x='century',
        y='culture',
        size='count',
        color='culture',
        title='Artifact Distribution by Culture and Century',
        hover_data=['count']
    )
    
    return fig

def create_color_spectrum_chart(conn):
    """
    Visualize color spectrum distribution
    """
    query = """
        SELECT spectrum, SUM(percent) as total_percent
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY total_percent DESC
    """
    
    df = pd.read_sql(query, conn)
    
    fig = px.pie(
        df,
        names='spectrum',
        values='total_percent',
        title='Color Spectrum Distribution'
    )
    
    return fig
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_records, conn):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        # Extract
        st.info("Extracting data from API...")
        artifacts = fetch_artifacts(api_key, num_records)
        
        if not artifacts:
            st.error("No artifacts fetched. Check API key and connection.")
            return False
        
        # Transform
        st.info("Transforming data...")
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        # Load
        st.info("Loading to database...")
        load_to_database(df_metadata, df_media, df_colors, conn)
        
        st.success(f"✅ Successfully processed {len(df_metadata)} artifacts")
        return True
        
    except requests.exceptions.RequestException as e:
        st.error(f"API Error: {e}")
        return False
    except mysql.connector.Error as e:
        st.error(f"Database Error: {e}")
        return False
    except Exception as e:
        st.error(f"Unexpected Error: {e}")
        return False
```

### Pagination and Rate Limiting

```python
def fetch_with_rate_limit(api_key, total_records, page_size=100, delay=0.5):
    """
    Fetch data with automatic pagination and rate limiting
    """
    all_data = []
    page = 1
    
    while len(all_data) < total_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 429:  # Rate limit hit
            st.warning("Rate limit reached. Waiting 60 seconds...")
            time.sleep(60)
            continue
        
        response.raise_for_status()
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
        
        all_data.extend(records)
        page += 1
        time.sleep(delay)
    
    return all_data[:total_records]
```

## Troubleshooting

### Common Issues

**API Key Invalid:**
```python
# Test API connection
response = requests.get(
    f"{BASE_URL}?apikey={API_KEY}&size=1"
)
if response.status_code == 401:
    st.error("Invalid API key. Get one from harvardartmuseums.org")
```

**Database Connection Errors:**
```python
try:
    conn = mysql.connector.connect(**db_config)
    conn.ping(reconnect=True, attempts=3, delay=5)
except mysql.connector.Error as e:
    st.error(f"Cannot connect to database: {e}")
    st.info("Check DB_HOST, DB_USER, DB_PASSWORD in .env file")
```

**Memory Issues with Large Datasets:**
```python
# Use chunked processing
def load_in_chunks(df, conn, chunk_size=1000):
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        load_to_database(chunk, conn)
        st.progress((i + chunk_size) / len(df))
```

**Empty Results:**
```python
# Add data validation
if df_metadata.empty:
    st.warning("No metadata extracted. Check API response format.")
    st.json(artifacts[0])  # Show sample for debugging
```

## Running the Application

```bash
# Development mode
streamlit run app.py

# Production mode with specific port
streamlit run app.py --server.port 8501 --server.address 0.0.0.0

# With custom config
streamlit run app.py --server.maxUploadSize 200
```

This skill enables AI agents to build complete data engineering pipelines with API integration, ETL processes, SQL analytics, and interactive visualizations using the Harvard Art Museums dataset.
