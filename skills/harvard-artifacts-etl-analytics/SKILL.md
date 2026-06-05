---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering pipeline with museum artifacts
  - set up analytics dashboard for Harvard API data
  - implement artifact collection data processing
  - build Streamlit app for museum data visualization
  - create SQL analytics for art museum collections
  - process Harvard Art Museums API with Python
  - design relational database for artifact metadata
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for collecting, processing, storing, and visualizing artifact data from the Harvard Art Museums API. It demonstrates:

- **ETL Pipeline**: Extract data from Harvard API, transform nested JSON into relational format, load into SQL database
- **Database Design**: Three normalized tables (artifactmetadata, artifactmedia, artifactcolors) with foreign key relationships
- **Analytics**: 20+ predefined SQL queries for insights on culture, century, media, colors, and departments
- **Visualization**: Streamlit dashboard with Plotly charts for interactive data exploration

**Architecture Flow**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization

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
export DB_NAME="your_database_name"
```

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api

Store credentials in `.env` file:
```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Main artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    objectnumber VARCHAR(100),
    url VARCHAR(500),
    creditline TEXT,
    period VARCHAR(200),
    technique VARCHAR(500)
);

-- Media/images table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Color analysis table
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

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py
```

The app will open at `http://localhost:8501`

## ETL Pipeline Implementation

### Extract: Fetch Data from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Extract artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    if response.status_code == 200:
        data = response.json()
        return data.get('records', []), data.get('info', {})
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Paginated collection
def collect_all_artifacts(max_pages=10):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        print(f"Collected page {page}: {len(records)} artifacts")
        
        # Check if we've reached the last page
        if page >= info.get('pages', 0):
            break
    
    return all_artifacts
```

### Transform: Process and Normalize Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into normalized dataframes"""
    
    # Extract metadata
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Metadata table
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'objectnumber': artifact.get('objectnumber'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique')
        })
        
        # Media table
        if artifact.get('primaryimageurl'):
            media_list.append({
                'artifact_id': artifact.get('id'),
                'iiifbaseuri': artifact.get('iiifbaseuri'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl')
            })
        
        # Colors table
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load dataframes into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch insert for performance)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, objectnumber, url, creditline, period, technique)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, iiifbaseuri, baseimageurl, primaryimageurl)
            VALUES (%s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytics Queries

### Sample SQL Queries for Insights

```python
# Query 1: Artifact distribution by culture
query_culture = """
    SELECT culture, COUNT(*) as count 
    FROM artifactmetadata 
    WHERE culture IS NOT NULL 
    GROUP BY culture 
    ORDER BY count DESC 
    LIMIT 20
"""

# Query 2: Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count 
    FROM artifactmetadata 
    WHERE century IS NOT NULL 
    GROUP BY century 
    ORDER BY count DESC
"""

# Query 3: Most common colors in artifacts
query_colors = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 15
"""

# Query 4: Artifacts with images by department
query_media = """
    SELECT m.department, COUNT(DISTINCT a.artifact_id) as artifacts_with_images
    FROM artifactmetadata m
    JOIN artifactmedia a ON m.id = a.artifact_id
    GROUP BY m.department
    ORDER BY artifacts_with_images DESC
"""

# Query 5: Color spectrum analysis
query_spectrum = """
    SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_coverage
    FROM artifactcolors
    WHERE spectrum IS NOT NULL
    GROUP BY spectrum
    ORDER BY count DESC
"""

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql_query(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Analysis",
        ["ETL Pipeline", "Cultural Analysis", "Color Analysis", "Media Analysis"]
    )
    
    if page == "ETL Pipeline":
        st.header("📥 ETL Data Collection")
        
        num_pages = st.number_input("Number of pages to collect", 1, 100, 5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Collecting artifacts..."):
                artifacts = collect_all_artifacts(max_pages=num_pages)
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                load_to_database(metadata_df, media_df, colors_df)
                
                st.success(f"✅ Loaded {len(metadata_df)} artifacts successfully!")
                st.dataframe(metadata_df.head(10))
    
    elif page == "Cultural Analysis":
        st.header("🌍 Artifacts by Culture")
        
        df = execute_query(query_culture)
        
        col1, col2 = st.columns([1, 2])
        with col1:
            st.dataframe(df)
        with col2:
            fig = px.bar(df, x='culture', y='count', 
                        title='Top 20 Cultures by Artifact Count')
            st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Color Analysis":
        st.header("🎨 Color Distribution Analysis")
        
        df = execute_query(query_colors)
        
        fig = px.bar(df, x='color', y='frequency',
                    title='Most Frequent Colors in Artifacts',
                    color='avg_percent',
                    color_continuous_scale='Viridis')
        st.plotly_chart(fig, use_container_width=True)
        
        # Spectrum analysis
        df_spectrum = execute_query(query_spectrum)
        fig2 = px.pie(df_spectrum, values='count', names='spectrum',
                     title='Color Spectrum Distribution')
        st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(page, rate_limit=1.0):
    """Fetch data with rate limiting (seconds between requests)"""
    time.sleep(rate_limit)
    return fetch_artifacts(page=page)
```

### Error Handling for ETL

```python
def safe_etl_pipeline(max_pages=10):
    """ETL pipeline with comprehensive error handling"""
    try:
        artifacts = collect_all_artifacts(max_pages)
        
        if not artifacts:
            raise ValueError("No artifacts collected")
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate dataframes
        assert not metadata_df.empty, "Metadata is empty"
        
        load_to_database(metadata_df, media_df, colors_df)
        
        return {
            'status': 'success',
            'artifacts_count': len(metadata_df),
            'media_count': len(media_df),
            'colors_count': len(colors_df)
        }
    
    except Exception as e:
        return {'status': 'error', 'message': str(e)}
```

### Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the highest artifact ID currently in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    conn.close()
    return result or 0

def incremental_load():
    """Load only new artifacts not already in database"""
    latest_id = get_latest_artifact_id()
    
    # Fetch and filter new artifacts
    all_artifacts = collect_all_artifacts(max_pages=5)
    new_artifacts = [a for a in all_artifacts if a['id'] > latest_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        return len(new_artifacts)
    return 0
```

## Troubleshooting

### API Key Issues
- Verify API key is valid at https://harvardartmuseums.org/collections/api
- Check `.env` file is in project root and properly formatted
- Ensure `python-dotenv` is installed and `load_dotenv()` is called

### Database Connection Errors
```python
# Test connection
try:
    conn = get_db_connection()
    print("✅ Database connection successful")
    conn.close()
except Error as e:
    print(f"❌ Connection failed: {e}")
```

### Empty Results from API
- Check API rate limits (429 status code)
- Verify `hasimage=1` parameter if expecting image data
- Confirm API endpoint is accessible: `https://api.harvardartmuseums.org/object`

### Duplicate Key Errors
Use `ON DUPLICATE KEY UPDATE` or check existing IDs before insert:
```python
# Check if artifact exists
cursor.execute("SELECT id FROM artifactmetadata WHERE id = %s", (artifact_id,))
if cursor.fetchone():
    # Skip or update
    pass
```

### Memory Issues with Large Datasets
Process in smaller batches:
```python
def batch_load(artifacts, batch_size=100):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        load_to_database(metadata_df, media_df, colors_df)
```
