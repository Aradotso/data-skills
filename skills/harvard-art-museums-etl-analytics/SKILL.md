---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and load Harvard museum collection data
  - analyze art museum data with SQL queries
  - set up data engineering pipeline with Streamlit
  - visualize Harvard art collection with plotly
  - query museum artifact metadata and colors
  - build museum data warehouse with Python
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for the Harvard Art Museums API. It demonstrates how to build production-grade ETL pipelines that extract artifact data, transform nested JSON into relational structures, load into SQL databases, and visualize analytics through interactive Streamlit dashboards.

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

Key capabilities:
- Extract artifact metadata, media, and color data via API pagination
- Transform nested JSON to normalized relational tables
- Load data into MySQL/TiDB Cloud with batch inserts
- Execute 20+ analytical SQL queries
- Generate interactive Plotly visualizations

## Installation

```bash
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App
pip install -r requirements.txt
```

**Required Dependencies:**
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

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

### 2. Database Setup

Set up MySQL or TiDB Cloud database with connection credentials.

### 3. Environment Variables

Create a `.env` file or configure via Streamlit secrets:

```python
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_art_db
DB_PORT=3306
```

Or use `~/.streamlit/secrets.toml`:

```toml
HARVARD_API_KEY = "your_api_key_here"
DB_HOST = "your_database_host"
DB_USER = "your_database_user"
DB_PASSWORD = "your_database_password"
DB_NAME = "harvard_art_db"
DB_PORT = 3306
```

## Database Schema

The ETL pipeline creates three related tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    provenance TEXT,
    description TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
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
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Usage

### Extract Data from API

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)
artifacts = data['records']
total_pages = data['info']['pages']
```

### Transform Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact records to flat structure"""
    metadata = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'provenance': artifact.get('provenance'),
            'description': artifact.get('description'),
            'url': artifact.get('url')
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Extract media information"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'artifact_id': artifact_id,
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            })
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color data"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            port=int(os.getenv('DB_PORT', 3306))
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def batch_insert_metadata(df, connection):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, 
     dated, period, technique, medium, dimensions, provenance, description, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    cursor.close()

def batch_insert_media(df, connection):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, baseimageurl, format, height, width)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    cursor.close()

def batch_insert_colors(df, connection):
    """Batch insert color data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    cursor.close()
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=5):
    """Run complete ETL pipeline"""
    api_key = os.getenv('HARVARD_API_KEY')
    connection = create_connection()
    
    if not connection:
        print("Failed to connect to database")
        return
    
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, size=100, page=page)
        all_artifacts.extend(data['records'])
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(all_artifacts)
    media_df = transform_artifact_media(all_artifacts)
    colors_df = transform_artifact_colors(all_artifacts)
    
    # Load
    print("Loading to database...")
    batch_insert_metadata(metadata_df, connection)
    batch_insert_media(media_df, connection)
    batch_insert_colors(colors_df, connection)
    
    connection.close()
    print(f"ETL complete! Processed {len(all_artifacts)} artifacts")

# Run the pipeline
run_etl_pipeline(num_pages=10)
```

## SQL Analytics Queries

### Common Analytical Queries

```python
def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results, columns=columns)

# Top 10 cultures by artifact count
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts with most images
query_images = """
SELECT m.title, m.culture, COUNT(media.media_id) as image_count
FROM artifactmetadata m
JOIN artifactmedia media ON m.id = media.artifact_id
GROUP BY m.id, m.title, m.culture
ORDER BY image_count DESC
LIMIT 10
"""

# Color distribution across artifacts
query_colors = """
SELECT color, COUNT(*) as frequency
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 15
"""

# Century-wise artifact distribution
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Department breakdown
query_department = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""

# Execute queries
connection = create_connection()
df_culture = execute_query(query_culture, connection)
df_images = execute_query(query_images, connection)
connection.close()
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for configuration
st.sidebar.header("Configuration")

# API key input
api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))

# Database connection inputs
db_host = st.sidebar.text_input("DB Host", value=os.getenv('DB_HOST', ''))
db_user = st.sidebar.text_input("DB User", value=os.getenv('DB_USER', ''))
db_password = st.sidebar.text_input("DB Password", type="password", 
                                   value=os.getenv('DB_PASSWORD', ''))
db_name = st.sidebar.text_input("DB Name", value=os.getenv('DB_NAME', ''))

# ETL Section
st.header("📥 ETL Pipeline")

num_pages = st.slider("Number of pages to fetch", 1, 50, 5)

if st.button("Run ETL Pipeline"):
    with st.spinner("Running ETL..."):
        try:
            run_etl_pipeline(num_pages)
            st.success(f"✅ ETL completed! Fetched {num_pages * 100} artifacts")
        except Exception as e:
            st.error(f"❌ Error: {str(e)}")

# Analytics Section
st.header("📊 Analytics")

query_options = {
    "Top 10 Cultures": query_culture,
    "Artifacts with Most Images": query_images,
    "Color Distribution": query_colors,
    "Century Distribution": query_century,
    "Department Breakdown": query_department
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))

if st.button("Run Analysis"):
    connection = create_connection()
    if connection:
        df = execute_query(query_options[selected_query], connection)
        connection.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=0.5):
    """Fetch with rate limiting to avoid API throttling"""
    artifacts = []
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, page=page)
        artifacts.extend(data['records'])
        time.sleep(delay)  # Respect rate limits
    return artifacts
```

### Incremental ETL

```python
def get_latest_artifact_id(connection):
    """Get the highest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    return result or 0

def incremental_etl(connection):
    """Only fetch new artifacts"""
    last_id = get_latest_artifact_id(connection)
    # Fetch artifacts with id > last_id
    # Implementation depends on API filtering capabilities
```

## Troubleshooting

**API Key Issues:**
- Verify key at https://www.harvardartmuseums.org/collections/api
- Check environment variable is loaded: `echo $HARVARD_API_KEY`

**Database Connection Errors:**
- Verify credentials and network access
- For TiDB Cloud, ensure IP whitelist is configured
- Test connection: `mysql -h $DB_HOST -u $DB_USER -p`

**Missing Data in Tables:**
- Some artifacts may not have images/colors (use NULL handling)
- Check API response structure for schema changes

**Performance Issues:**
- Use batch inserts (executemany) instead of row-by-row
- Add indexes on frequently queried columns
- Increase page size up to API limit (100)

**Streamlit Secrets:**
- Ensure `.streamlit/secrets.toml` exists in project root
- Access via `st.secrets["KEY_NAME"]`
