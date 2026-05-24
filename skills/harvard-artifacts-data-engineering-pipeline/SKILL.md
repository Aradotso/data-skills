---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create ETL pipeline for museum artifact data
  - set up analytics dashboard for Harvard artifacts collection
  - query and visualize Harvard museum data with SQL
  - extract artifact metadata from Harvard API
  - build Streamlit app for museum data analytics
  - create database schema for Harvard art collection
  - analyze museum artifact data with Python and SQL
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines for the Harvard Art Museums API. The project demonstrates ETL workflows, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational database tables
- **Database Design**: Structured SQL schema for artifacts metadata, media, and color data
- **Analytics Queries**: 20+ predefined SQL queries for insights on collections
- **Visualization**: Interactive Plotly charts in a Streamlit dashboard

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

### 1. API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### 2. Database Setup

Create the database and tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_type VARCHAR(100),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color_hex VARCHAR(20),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifact data from Harvard Art Museums API
    """
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
def fetch_all_artifacts(max_pages=5):
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        print(f"Fetched page {page}/{info['pages']}")
        
    return all_artifacts
```

### ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """
    Transform nested JSON to relational format
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown')
        }
        metadata_list.append(metadata)
        
        # Extract media
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'media_url': image.get('baseimageurl', '')
                })
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                colors_list.append({
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex', '#000000'),
                    'color_percentage': color.get('percent', 0.0)
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed data into MySQL database
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT IGNORE INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, dated, accessionyear, technique, medium)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, media_type, media_url)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color_hex, color_percentage)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### Analytical SQL Queries

```python
# Sample analytical queries for the dashboard

ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
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
            am.classification,
            COUNT(DISTINCT am.id) as total_artifacts,
            COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
            ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
        FROM artifactmetadata am
        LEFT JOIN artifactmedia media ON am.id = media.artifact_id
        GROUP BY am.classification
        ORDER BY total_artifacts DESC
        LIMIT 10
    """,
    
    "Top Colors Used": """
        SELECT 
            color_hex,
            COUNT(*) as usage_count,
            ROUND(AVG(color_percentage), 2) as avg_percentage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Accession Year Trends": """
        SELECT 
            accessionyear,
            COUNT(*) as artifacts_acquired
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 20
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICAL_QUERIES[query_name]
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

### Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    # Execute query button
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(query_name)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name,
                    labels={df.columns[0]: df.columns[0].title(),
                           df.columns[1]: df.columns[1].title()}
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Pipeline Section
    st.sidebar.header("Data Collection")
    num_pages = st.sidebar.number_input("Pages to fetch", min_value=1, max_value=20, value=5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts from API..."):
            raw_data = fetch_all_artifacts(max_pages=num_pages)
            st.success(f"Fetched {len(raw_data)} artifacts")
            
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.info("Data transformed successfully")
            
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded to database!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID already in database"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    connection.close()
    return max_id

def fetch_new_artifacts_only():
    """Fetch only artifacts not yet in database"""
    max_id = get_max_artifact_id()
    # Use API filters to fetch artifacts with id > max_id
    pass
```

### Error Handling and Retry Logic

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_artifacts_rate_limited(page, delay=1):
    """Add delay between API calls"""
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Database Connection Issues
- Ensure MySQL/TiDB Cloud credentials are correct in `.env`
- Check firewall rules allow connection to database host
- Verify database user has INSERT/SELECT permissions

### Missing Data Fields
```python
# Use .get() with defaults for optional fields
culture = artifact.get('culture', 'Unknown')
accessionyear = artifact.get('accessionyear', None)
```

### Memory Issues with Large Datasets
```python
# Process in batches
def batch_load(artifacts, batch_size=100):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        load_to_database(metadata_df, media_df, colors_df)
```

This skill provides the foundation for building production-ready data engineering pipelines with museum API data, SQL analytics, and interactive dashboards.
