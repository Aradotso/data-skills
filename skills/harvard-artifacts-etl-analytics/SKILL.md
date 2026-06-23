---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch Harvard Art Museums data
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - query Harvard Art Museums API
  - visualize artifact data with Plotly
  - set up museum data engineering pipeline
  - analyze art collection metadata with Python
  - load Harvard API data into MySQL
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics solution for the Harvard Art Museums API. It demonstrates production-ready ETL pipelines that extract artifact metadata, transform nested JSON into relational tables, load data into SQL databases (MySQL/TiDB), and visualize insights through an interactive Streamlit dashboard.

**Key capabilities:**
- API data collection with pagination and rate limiting
- Multi-table relational database design
- 20+ predefined analytical SQL queries
- Interactive Plotly visualizations
- Real-world data pipeline patterns

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dAd8CVqnhjgMQ/viewform

Store it in environment variables:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Configure MySQL/TiDB connection:

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Database Schema

The project uses three relational tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    period VARCHAR(255),
    provenance TEXT,
    verificationlevel INT
);

-- Media/images table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Color analysis table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core Usage Patterns

### 1. Fetching Data from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        size: Number of records per page (max 100)
        page: Page number for pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=50, page=1)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Records in this page: {len(data['records'])}")
```

### 2. ETL Pipeline - Extract and Transform

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API response into structured dataframes"""
    
    records = raw_data.get('records', [])
    
    # Extract metadata
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        object_id = record.get('objectid')
        
        # Metadata extraction
        metadata = {
            'objectid': object_id,
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:255],
            'century': record.get('century', '')[:100],
            'dated': record.get('dated', '')[:255],
            'classification': record.get('classification', '')[:255],
            'department': record.get('department', '')[:255],
            'division': record.get('division', '')[:255],
            'technique': record.get('technique', '')[:500],
            'medium': record.get('medium', '')[:500],
            'period': record.get('period', '')[:255],
            'provenance': record.get('provenance', ''),
            'verificationlevel': record.get('verificationlevel', 0)
        }
        metadata_list.append(metadata)
        
        # Media extraction
        images = record.get('images', [])
        if images:
            media = {
                'objectid': object_id,
                'baseimageurl': record.get('baseimageurl', ''),
                'primaryimageurl': record.get('primaryimageurl', '')
            }
            media_list.append(media)
        
        # Colors extraction
        colors = record.get('colors', [])
        for color in colors:
            color_data = {
                'objectid': object_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Loading Data into SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """Load transformed data into MySQL database"""
    
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Load metadata (batch insert)
        if not df_metadata.empty:
            metadata_cols = df_metadata.columns.tolist()
            placeholders = ', '.join(['%s'] * len(metadata_cols))
            insert_query = f"""
                INSERT IGNORE INTO artifactmetadata 
                ({', '.join(metadata_cols)}) 
                VALUES ({placeholders})
            """
            cursor.executemany(insert_query, df_metadata.values.tolist())
        
        # Load media
        if not df_media.empty:
            media_cols = df_media.columns.tolist()
            placeholders = ', '.join(['%s'] * len(media_cols))
            insert_query = f"""
                INSERT INTO artifactmedia 
                ({', '.join(media_cols)}) 
                VALUES ({placeholders})
            """
            cursor.executemany(insert_query, df_media.values.tolist())
        
        # Load colors
        if not df_colors.empty:
            colors_cols = df_colors.columns.tolist()
            placeholders = ', '.join(['%s'] * len(colors_cols))
            insert_query = f"""
                INSERT INTO artifactcolors 
                ({', '.join(colors_cols)}) 
                VALUES ({placeholders})
            """
            cursor.executemany(insert_query, df_colors.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} color records")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 4. Analytical SQL Queries

```python
def run_analytics_query(query_name, db_config):
    """Execute predefined analytical queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 15
        """,
        
        'artifacts_with_images': """
            SELECT 
                CASE 
                    WHEN primaryimageurl IS NOT NULL THEN 'Has Image'
                    ELSE 'No Image'
                END as image_status,
                COUNT(*) as count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia med ON m.objectid = med.objectid
            GROUP BY image_status
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(queries[query_name], connection)
    connection.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    st.title("🎨 Harvard Art Museums Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Data collection
    if st.sidebar.button("Fetch Artifacts"):
        with st.spinner("Fetching data..."):
            raw_data = fetch_artifacts(api_key, size=100)
            df_meta, df_media, df_colors = transform_artifacts(raw_data)
            load_to_database(df_meta, df_media, df_colors, DB_CONFIG)
            st.success(f"Loaded {len(df_meta)} artifacts!")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    query_option = st.selectbox(
        "Select Analysis",
        ["artifacts_by_culture", "artifacts_by_century", "top_colors", 
         "department_distribution"]
    )
    
    if st.button("Run Query"):
        df_result = run_analytics_query(query_option, DB_CONFIG)
        
        # Display table
        st.dataframe(df_result)
        
        # Visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(
                df_result, 
                x=df_result.columns[0], 
                y=df_result.columns[1],
                title=f"Analysis: {query_option}"
            )
            st.plotly_chart(fig)

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(api_key, max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, size=100, page=page)
        all_records.extend(data['records'])
        
        # Check if more pages exist
        total_pages = data['info']['pages']
        if page >= total_pages:
            break
    
    return all_records
```

### Error Handling

```python
def safe_api_call(api_key, size=100, page=1, retries=3):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key, size, page)
        except requests.exceptions.RequestException as e:
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Rate Limiting:**
- Harvard API has rate limits; add delays between requests
- Use `time.sleep(0.5)` between paginated calls

**Database Connection Issues:**
- Verify DB_CONFIG credentials
- Check firewall rules for remote databases (TiDB Cloud)
- Ensure database and tables are created before loading

**Missing Data:**
- Not all artifacts have images/colors; handle None values
- Use `.get()` method for safe dictionary access
- Filter for `hasimage=1` in API params if images required

**Memory Issues with Large Datasets:**
- Process data in batches (100-1000 records)
- Use chunked DataFrame inserts
- Clear DataFrames after loading: `del df_metadata`
