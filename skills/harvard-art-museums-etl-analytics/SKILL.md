---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data analytics dashboard with museum artifacts
  - extract and transform Harvard museum API data
  - set up SQL database for art collection analytics
  - visualize museum artifact data with Streamlit
  - implement data engineering pipeline for art museums
  - query and analyze Harvard Art Museums collection
  - build interactive analytics app for museum data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What It Does

The Harvard Art Museums ETL Analytics project provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit frontend with Plotly visualizations

Architecture flow: **API → ETL → SQL → Analytics → Visualization**

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

### API Key Setup

Obtain a Harvard Art Museums API key from https://www.harvardartmuseums.org/collections/api

```python
# Store in environment variable
import os
os.environ['HARVARD_API_KEY'] = 'your_api_key_here'

# Or use .env file
# .env
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

```python
import mysql.connector

# Database connection
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}

connection = mysql.connector.connect(**db_config)
```

## Core Components

### 1. API Data Extraction

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
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=50)

print(f"Total records: {info['totalrecords']}")
print(f"Total pages: {info['pages']}")
```

### 2. ETL Pipeline

```python
import pandas as pd
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """
    Transform raw API data into relational tables
    Returns: (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'objectid': artifact.get('objectid'),
                    'baseimageurl': img.get('baseimageurl'),
                    'iiifbaseuri': img.get('iiifbaseuri'),
                    'format': img.get('format'),
                    'width': img.get('width'),
                    'height': img.get('height')
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_record = {
                    'objectid': artifact.get('objectid'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                colors_list.append(color_record)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Usage
metadata_df, media_df, colors_df = transform_artifacts(artifacts)
```

### 3. Database Schema Creation

```python
def create_tables(connection):
    """
    Create SQL tables for artifact data
    """
    cursor = connection.cursor()
    
    # Artifacts metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            dated VARCHAR(200),
            period VARCHAR(200),
            classification VARCHAR(200),
            department VARCHAR(200),
            technique VARCHAR(500),
            medium TEXT,
            dimensions VARCHAR(500),
            creditline TEXT,
            accessionyear INT
        )
    """)
    
    # Artifacts media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifacts colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Loading Data to SQL

```python
def load_to_sql(connection, metadata_df, media_df, colors_df):
    """
    Batch insert DataFrames into SQL tables
    """
    from mysql.connector import Error
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (objectid, title, culture, century, dated, period, 
                 classification, department, technique, medium, 
                 dimensions, creditline, accessionyear)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (objectid, baseimageurl, iiifbaseuri, format, width, height)
                VALUES (%s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (objectid, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

### 5. Analytical SQL Queries

```python
# Example analytical queries
ANALYTICAL_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
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
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Artifacts with Multiple Images": """
        SELECT m.objectid, m.title, COUNT(am.id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia am ON m.objectid = am.objectid
        GROUP BY m.objectid, m.title
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """
}

def execute_query(connection, query_name):
    """Execute analytical query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(ANALYTICAL_QUERIES[query_name])
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # API Key input
    api_key = st.sidebar.text_input("Harvard API Key", 
                                     value=os.getenv('HARVARD_API_KEY', ''),
                                     type="password")
    
    # Database connection
    if st.sidebar.button("Connect to Database"):
        try:
            conn = mysql.connector.connect(**db_config)
            st.sidebar.success("✅ Connected to database")
            st.session_state['connection'] = conn
        except Exception as e:
            st.sidebar.error(f"❌ Connection failed: {e}")
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 100, 5)
    with col2:
        page_size = st.number_input("Artifacts per page", 10, 100, 50)
    
    if st.button("🚀 Run ETL Pipeline"):
        if not api_key:
            st.error("Please provide API key")
            return
        
        progress_bar = st.progress(0)
        status_text = st.empty()
        
        all_artifacts = []
        for page in range(1, num_pages + 1):
            status_text.text(f"Fetching page {page}/{num_pages}")
            artifacts, _ = fetch_artifacts(api_key, page, page_size)
            all_artifacts.extend(artifacts)
            progress_bar.progress(page / num_pages)
        
        st.success(f"✅ Fetched {len(all_artifacts)} artifacts")
        
        # Transform
        status_text.text("Transforming data...")
        metadata_df, media_df, colors_df = transform_artifacts(all_artifacts)
        
        # Load
        if 'connection' in st.session_state:
            status_text.text("Loading to database...")
            load_to_sql(st.session_state['connection'], 
                       metadata_df, media_df, colors_df)
            st.success("✅ ETL pipeline completed!")
        else:
            st.warning("Connect to database to load data")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        if 'connection' not in st.session_state:
            st.error("Please connect to database first")
            return
        
        results_df = execute_query(st.session_state['connection'], query_name)
        
        st.subheader("Query Results")
        st.dataframe(results_df)
        
        # Auto-generate visualization
        if len(results_df.columns) >= 2:
            st.subheader("Visualization")
            fig = px.bar(results_df, 
                        x=results_df.columns[0], 
                        y=results_df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
        
        # Download option
        csv = results_df.to_csv(index=False)
        st.download_button("📥 Download CSV", csv, 
                          f"{query_name.replace(' ', '_')}.csv")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_api_key
export DB_HOST=localhost
export DB_USER=root
export DB_PASSWORD=your_password
export DB_NAME=harvard_artifacts

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Complete ETL Workflow

```python
def full_etl_pipeline(api_key, db_config, num_pages=5):
    """Complete ETL pipeline execution"""
    # Extract
    all_artifacts = []
    for page in range(1, num_pages + 1):
        artifacts, _ = fetch_artifacts(api_key, page, size=100)
        all_artifacts.extend(artifacts)
        print(f"Fetched page {page}")
    
    # Transform
    metadata_df, media_df, colors_df = transform_artifacts(all_artifacts)
    
    # Load
    connection = mysql.connector.connect(**db_config)
    create_tables(connection)
    load_to_sql(connection, metadata_df, media_df, colors_df)
    connection.close()
    
    print(f"ETL completed: {len(metadata_df)} artifacts processed")
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
import time
time.sleep(0.5)  # 500ms delay between API calls
```

**Database Connection Issues**: Check credentials and network
```python
try:
    conn = mysql.connector.connect(**db_config, connect_timeout=10)
except mysql.connector.Error as e:
    print(f"Error: {e}")
```

**Memory Issues with Large Datasets**: Process in batches
```python
def batch_load(df, connection, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df[i:i+batch_size]
        # Insert batch
        connection.commit()
```

**Streamlit Session State**: Persist database connections
```python
if 'connection' not in st.session_state:
    st.session_state['connection'] = mysql.connector.connect(**db_config)
```
