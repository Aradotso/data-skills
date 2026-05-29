---
name: harvard-artifacts-etl-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data with MySQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data
  - set up artifact data engineering pipeline
  - query Harvard Art Museums collection database
  - visualize museum artifact analytics
  - implement streaming ETL for art collection data
  - analyze Harvard artifacts with SQL queries
---

# Harvard Artifacts ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL analytics, and interactive Streamlit dashboards for artifact collection data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Execute predefined analytical queries on structured artifact data
- **Interactive Visualization**: Streamlit dashboards with Plotly charts for data exploration

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Database Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema():
    """Initialize database schema for artifact storage"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    
    cursor = connection.cursor()
    
    # Create database
    cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
    cursor.execute(f"USE {os.getenv('DB_NAME')}")
    
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
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            division VARCHAR(200),
            verificationlevel INT,
            totalpageviews INT,
            totaluniquepageviews INT
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            imageid INT,
            baseimageurl VARCHAR(1000),
            format VARCHAR(50),
            height INT,
            width INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
    connection.close()
```

## Key API Patterns

### Fetching Data from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'page': page,
        'size': size
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

def fetch_multiple_pages(num_pages=5):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            print(f"Fetched page {page}: {len(data['records'])} artifacts")
        else:
            break
    
    return all_artifacts
```

### ETL Pipeline Implementation

```python
import pandas as pd

def extract_artifact_metadata(artifacts):
    """Extract metadata from raw API response"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'division': artifact.get('division'),
            'verificationlevel': artifact.get('verificationlevel'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        })
    
    return pd.DataFrame(metadata)

def extract_artifact_media(artifacts):
    """Extract media/image data from artifacts"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media.append({
                'artifact_id': artifact_id,
                'imageid': image.get('imageid'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            })
    
    return pd.DataFrame(media)

def extract_artifact_colors(artifacts):
    """Extract color data from artifacts"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### Loading Data to MySQL

```python
def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, dated, 
             period, technique, medium, dimensions, division, verificationlevel, 
             totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Load media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, imageid, baseimageurl, format, height, width)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
    connection.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    """Main Streamlit application"""
    st.title("Harvard Artifacts Collection Analytics")
    st.write("Interactive ETL Pipeline and Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        sql_analytics_page()
    elif page == "Visualizations":
        visualizations_page()

def data_collection_page():
    """Page for collecting data from API"""
    st.header("Data Collection from Harvard API")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=20, value=5)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_multiple_pages(num_pages)
            
            metadata_df = extract_artifact_metadata(artifacts)
            media_df = extract_artifact_media(artifacts)
            colors_df = extract_artifact_colors(artifacts)
            
            load_to_database(metadata_df, media_df, colors_df)
            
            st.success(f"Loaded {len(artifacts)} artifacts successfully!")
            st.dataframe(metadata_df.head())

def sql_analytics_page():
    """Page for running SQL queries"""
    st.header("SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
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
        "Media Availability": """
            SELECT 
                CASE WHEN EXISTS (
                    SELECT 1 FROM artifactmedia WHERE artifact_id = m.id
                ) THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata m
            GROUP BY media_status
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as count 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY count DESC 
            LIMIT 10
        """
    }
    
    query_name = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = pd.read_sql(queries[query_name], connection)
        connection.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Analytical Queries

```python
# Top 10 departments by artifact count
query_departments = """
    SELECT department, COUNT(*) as count 
    FROM artifactmetadata 
    WHERE department IS NOT NULL 
    GROUP BY department 
    ORDER BY count DESC 
    LIMIT 10
"""

# Average image dimensions by classification
query_dimensions = """
    SELECT m.classification, 
           AVG(a.width) as avg_width, 
           AVG(a.height) as avg_height
    FROM artifactmetadata m
    JOIN artifactmedia a ON m.id = a.artifact_id
    WHERE m.classification IS NOT NULL
    GROUP BY m.classification
    ORDER BY avg_width DESC
"""

# Color distribution analysis
query_color_distribution = """
    SELECT color, spectrum, 
           COUNT(*) as frequency,
           AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color, spectrum
    ORDER BY frequency DESC
    LIMIT 20
"""

# Artifacts with most pageviews
query_popular = """
    SELECT title, culture, century, totalpageviews
    FROM artifactmetadata
    WHERE totalpageviews IS NOT NULL
    ORDER BY totalpageviews DESC
    LIMIT 25
"""
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3, delay=2):
    """Fetch with exponential backoff on rate limits"""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(page=page)
            return data
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = delay * (2 ** attempt)
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
def get_database_connection(retries=3):
    """Get database connection with retry logic"""
    for attempt in range(retries):
        try:
            connection = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return connection
        except Error as e:
            if attempt < retries - 1:
                time.sleep(2)
            else:
                st.error(f"Database connection failed: {e}")
                return None
```

### Handling Missing Data

```python
def safe_extract(artifact, key, default=None):
    """Safely extract nested data from artifact"""
    value = artifact.get(key, default)
    return value if value not in [None, '', 'null'] else default

# Usage in extraction
metadata.append({
    'id': safe_extract(artifact, 'id'),
    'title': safe_extract(artifact, 'title', 'Untitled'),
    'culture': safe_extract(artifact, 'culture', 'Unknown')
})
```

This skill enables building production-ready ETL pipelines for museum artifact data with proper error handling, rate limiting, and interactive analytics dashboards.
