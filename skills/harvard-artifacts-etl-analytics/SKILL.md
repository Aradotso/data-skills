---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and museum data
  - query Harvard artifacts database with SQL
  - visualize museum collection data with Plotly
  - set up TiDB/MySQL for artifact metadata storage
  - implement paginated API data collection for museums
  - analyze art museum collections by culture and century
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics solution for the Harvard Art Museums API. It demonstrates ETL pipeline design, SQL database integration, analytical querying, and interactive Streamlit dashboards for museum artifact data.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination handling
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with optimized batch inserts
- **Analyzes** collections using 20+ predefined SQL analytical queries
- **Visualizes** results through interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard API Key

Obtain your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```env
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Connection

Configure MySQL or TiDB Cloud connection:

```env
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Database Schema Setup

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    primaryimageurl TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### ETL Pipeline Implementation

**Extract with Pagination:**

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(max_records=1000):
    """Collect artifacts with pagination handling"""
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < max_records:
        data = fetch_artifacts(page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return artifacts[:max_records]
```

**Transform Nested JSON:**

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data to metadata table format"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'url': artifact.get('url', ''),
            'primaryimageurl': artifact.get('primaryimageurl', '')
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Extract media/image data"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'artifact_id': artifact_id,
                'baseimageurl': img.get('baseimageurl', ''),
                'format': img.get('format', '')[:50],
                'height': img.get('height'),
                'width': img.get('width')
            })
    
    return pd.DataFrame(media_records)

def transform_colors(artifacts):
    """Extract color palette data"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

**Load to Database:**

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

def batch_insert_metadata(df):
    """Batch insert metadata with conflict handling"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, dated, url, primaryimageurl)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return len(data)

def batch_insert_media(df):
    """Batch insert media records"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return len(data)
```

### Analytical SQL Queries

**Common Analysis Patterns:**

```python
# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC
"""

# Department distribution
query_departments = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
GROUP BY department
ORDER BY count DESC
"""

# Media format analysis
query_media_formats = """
SELECT format, COUNT(*) as count
FROM artifactmedia
WHERE format IS NOT NULL
GROUP BY format
ORDER BY count DESC
LIMIT 10
"""

# Color spectrum distribution
query_color_spectrum = """
SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
FROM artifactcolors
WHERE spectrum IS NOT NULL
GROUP BY spectrum
ORDER BY count DESC
"""

# Artifacts with multiple images
query_multi_images = """
SELECT a.id, a.title, COUNT(m.media_id) as image_count
FROM artifactmetadata a
JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.id, a.title
HAVING image_count > 3
ORDER BY image_count DESC
LIMIT 20
"""
```

### Streamlit Dashboard

**Main Application Structure:**

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose Analysis",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection from API")
    
    num_records = st.number_input("Number of records", 100, 5000, 500)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_all_artifacts(max_records=num_records)
            
            # Transform
            df_metadata = transform_metadata(artifacts)
            df_media = transform_media(artifacts)
            df_colors = transform_colors(artifacts)
            
            # Load
            inserted_metadata = batch_insert_metadata(df_metadata)
            inserted_media = batch_insert_media(df_media)
            
            st.success(f"✅ Loaded {inserted_metadata} artifacts, {inserted_media} media records")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Department Breakdown": query_departments,
        "Media Formats": query_media_formats,
        "Color Spectrum": query_color_spectrum
    }
    
    selected = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        conn = get_db_connection()
        df = pd.read_sql(queries[selected], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time
from functools import wraps

def rate_limit(delay=0.5):
    """Decorator to add delay between API calls"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            time.sleep(delay)
            return result
        return wrapper
    return decorator

@rate_limit(delay=1.0)
def fetch_artifact_detail(artifact_id):
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object/{artifact_id}"
    response = requests.get(url, params={'apikey': api_key})
    return response.json()
```

### Error Handling for ETL

```python
def safe_etl_pipeline(artifacts):
    """ETL with comprehensive error handling"""
    try:
        # Extract
        st.info("Extracting data...")
        
        # Transform
        st.info("Transforming data...")
        df_metadata = transform_metadata(artifacts)
        
        if df_metadata.empty:
            st.warning("No data to transform")
            return
        
        # Load
        st.info("Loading to database...")
        inserted = batch_insert_metadata(df_metadata)
        st.success(f"Successfully loaded {inserted} records")
        
    except requests.RequestException as e:
        st.error(f"API Error: {str(e)}")
    except mysql.connector.Error as e:
        st.error(f"Database Error: {str(e)}")
    except Exception as e:
        st.error(f"Unexpected Error: {str(e)}")
```

## Troubleshooting

**API Rate Limits:**
- Harvard API has rate limits; add delays between requests
- Use `time.sleep(0.5)` or higher between paginated calls

**Database Connection Issues:**
- Verify `.env` file has correct credentials
- Check firewall rules for TiDB Cloud
- Test connection: `mysql -h HOST -u USER -p`

**Memory Issues with Large Datasets:**
```python
# Process in chunks
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df = transform_metadata(chunk)
        batch_insert_metadata(df)
```

**Missing Data Fields:**
```python
# Safe field extraction
def safe_get(data, key, default=''):
    return data.get(key, default) if data.get(key) is not None else default
```
