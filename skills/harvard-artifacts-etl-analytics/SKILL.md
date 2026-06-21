---
name: harvard-artifacts-etl-analytics
description: Build end-to-end data pipelines with the Harvard Art Museums API using Python ETL, SQL analytics, and Streamlit dashboards
triggers:
  - analyze harvard art museum data
  - build ETL pipeline for museum artifacts
  - create streamlit dashboard for art collections
  - query harvard art museums api
  - set up museum data analytics pipeline
  - visualize artifact collection data
  - extract transform load museum data
  - build art museum analytics dashboard
---

# Harvard Artifacts Collection Data Engineering Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational tables, loading into SQL databases, running analytics queries, and visualizing results through an interactive Streamlit dashboard.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (metadata, media, colors)
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics Queries**: Provides 20+ predefined SQL queries for artifact insights
- **Interactive Dashboard**: Streamlit UI for data collection, query execution, and Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Dependencies** (typical requirements.txt):
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Get Harvard Art Museums API Key

Register at: https://www.harvardartmuseums.org/collections/api

### 2. Set Up Environment Variables

Create a `.env` file:

```bash
# Harvard API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### 3. Database Setup

Create the required tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(20),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will open at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    all_artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(all_artifacts)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
        else:
            raise Exception(f"API Error: {response.status_code}")
    
    return all_artifacts[:num_records]
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into normalized dataframes"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
        
        # Extract media/images
        for img in artifact.get('images', []):
            media_records.append({
                'mediaid': img.get('imageid'),
                'artifact_id': artifact.get('id'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Batch insert dataframes into SQL tables"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, dated, url, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        if not df_media.empty:
            media_query = """
            INSERT INTO artifactmedia 
            (mediaid, artifact_id, baseimageurl, format, height, width)
            VALUES (%s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE baseimageurl=VALUES(baseimageurl)
            """
            cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        if not df_colors.empty:
            color_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(color_query, df_colors.values.tolist())
        
        conn.commit()
        return cursor.rowcount
        
    except Error as e:
        conn.rollback()
        raise e
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytics Queries

```python
def execute_analytics_query(query_name):
    """Execute predefined analytics queries"""
    
    queries = {
        "Top 10 Cultures": """
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
        
        "Department Distribution": """
            SELECT department, COUNT(*) as artifact_count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY artifact_count DESC
        """,
        
        "Most Viewed Artifacts": """
            SELECT title, culture, totalpageviews 
            FROM artifactmetadata 
            ORDER BY totalpageviews DESC 
            LIMIT 20
        """,
        
        "Color Analysis": """
            SELECT c.color, COUNT(*) as usage_count, AVG(c.percent) as avg_percent
            FROM artifactcolors c
            GROUP BY c.color
            ORDER BY usage_count DESC
            LIMIT 15
        """,
        
        "Media Format Distribution": """
            SELECT format, COUNT(*) as count
            FROM artifactmedia
            WHERE format IS NOT NULL
            GROUP BY format
            ORDER BY count DESC
        """
    }
    
    conn = get_db_connection()
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for data collection
    with st.sidebar:
        st.header("Data Collection")
        num_records = st.number_input("Number of records", 10, 1000, 100)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Fetching from API..."):
                raw_data = fetch_artifacts(num_records)
                df_meta, df_media, df_colors = transform_artifacts(raw_data)
                rows = load_to_database(df_meta, df_media, df_colors)
                st.success(f"Loaded {rows} records successfully!")
    
    # Analytics section
    st.header("📊 Analytics Queries")
    
    query_options = [
        "Top 10 Cultures",
        "Artifacts by Century",
        "Department Distribution",
        "Most Viewed Artifacts",
        "Color Analysis",
        "Media Format Distribution"
    ]
    
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        df_result = execute_analytics_query(selected_query)
        
        # Display results
        st.dataframe(df_result, use_container_width=True)
        
        # Visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(
                df_result, 
                x=df_result.columns[0], 
                y=df_result.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Handling API Rate Limits

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limit
            wait_time = 2 ** attempt
            time.sleep(wait_time)
        else:
            response.raise_for_status()
    
    raise Exception("Max retries exceeded")
```

### Data Quality Checks

```python
def validate_data(df_metadata):
    """Perform data quality checks"""
    checks = {
        'null_ids': df_metadata['id'].isnull().sum(),
        'duplicate_ids': df_metadata['id'].duplicated().sum(),
        'missing_titles': df_metadata['title'].isnull().sum()
    }
    
    for check, count in checks.items():
        if count > 0:
            st.warning(f"{check}: {count} records")
    
    return all(count == 0 for count in checks.values())
```

## Troubleshooting

**API Connection Issues**
- Verify API key in `.env` file
- Check API rate limits (Harvard allows 2500 requests/day)
- Ensure internet connectivity

**Database Connection Errors**
- Verify DB credentials in `.env`
- Check if database exists and tables are created
- Ensure MySQL server is running

**Duplicate Key Errors**
- Use `ON DUPLICATE KEY UPDATE` for idempotent inserts
- Clear tables before re-running: `TRUNCATE TABLE artifactmetadata;`

**Memory Issues with Large Datasets**
- Process data in batches (100-500 records)
- Use `chunksize` parameter in pandas operations

**Missing Dependencies**
```bash
pip install --upgrade streamlit pandas mysql-connector-python plotly python-dotenv requests
```
