---
name: harvard-artifacts-etl-analytics
description: Build end-to-end data pipelines from Harvard Art Museums API with ETL, SQL storage, and Streamlit analytics dashboards
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and SQL
  - connect to Harvard Art Museums API with Python
  - implement batch insert for artifact data
  - visualize museum collection data with Plotly
  - design SQL schema for museum artifacts
  - query Harvard artifacts with pagination
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering pipeline for Harvard Art Museums data, implementing:
- API extraction with pagination and rate limiting
- ETL transformations from nested JSON to relational tables
- SQL database storage (MySQL/TiDB Cloud)
- Interactive Streamlit analytics dashboard
- Plotly visualizations for insights

Perfect for learning real-world data engineering patterns, building portfolio projects, or creating museum data analytics applications.

## Installation

```bash
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App
pip install -r requirements.txt
```

**Key Dependencies:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform

### Database Schema

The project uses three core tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will launch at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

Fetch artifacts with pagination:

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### 2. ETL Pipeline

Transform nested JSON to relational format:

```python
import pandas as pd

def transform_artifacts(records):
    """Transform API records into normalized dataframes"""
    
    # Metadata extraction
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Extract metadata
        metadata = {
            'id': record.get('id'),
            'title': record.get('title', 'Unknown'),
            'culture': record.get('culture', 'Unknown'),
            'period': record.get('period', 'Unknown'),
            'century': record.get('century', 'Unknown'),
            'classification': record.get('classification', 'Unknown'),
            'department': record.get('department', 'Unknown'),
            'division': record.get('division', 'Unknown'),
            'dated': record.get('dated', 'Unknown'),
            'url': record.get('url'),
            'totalpageviews': record.get('totalpageviews', 0),
            'totaluniquepageviews': record.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        if 'images' in record and record['images']:
            for img in record['images']:
                media = {
                    'artifact_id': record.get('id'),
                    'baseimageurl': img.get('baseimageurl'),
                    'format': img.get('format', 'Unknown')
                }
                media_list.append(media)
        
        # Extract colors
        if 'colors' in record and record['colors']:
            for color in record['colors']:
                color_data = {
                    'artifact_id': record.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percentage': color.get('percent', 0)
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Usage
df_metadata, df_media, df_colors = transform_artifacts(data['records'])
```

### 3. Database Operations

Batch insert with MySQL:

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return connection

def batch_insert_metadata(df_metadata):
    """Insert metadata in batches"""
    connection = create_connection()
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, 
     department, division, dated, url, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(row) for row in df_metadata.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} records")
    except Error as e:
        print(f"Error: {e}")
        connection.rollback()
    finally:
        cursor.close()
        connection.close()

# Usage
batch_insert_metadata(df_metadata)
```

### 4. Analytics Queries

Example SQL queries for insights:

```python
def execute_analytics_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = create_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

df_cultures = execute_analytics_query(query_cultures)

# Artifacts with most colors
query_colorful = """
SELECT 
    m.title,
    m.culture,
    COUNT(c.color_id) as color_count
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
GROUP BY m.id, m.title, m.culture
ORDER BY color_count DESC
LIMIT 10
"""

df_colorful = execute_analytics_query(query_colorful)

# Department distribution
query_departments = """
SELECT 
    department,
    COUNT(*) as count,
    AVG(totalpageviews) as avg_views
FROM artifactmetadata
WHERE department != 'Unknown'
GROUP BY department
ORDER BY count DESC
"""

df_departments = execute_analytics_query(query_departments)
```

### 5. Streamlit Dashboard

Create interactive visualizations:

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Select Analysis")
query_options = [
    "Top Cultures",
    "Department Distribution",
    "Color Analysis",
    "Century Timeline",
    "Media Format Analysis"
]

selected_query = st.sidebar.selectbox("Choose Query", query_options)

# Execute and display results
if selected_query == "Top Cultures":
    df = execute_analytics_query(query_cultures)
    
    st.subheader("Top 10 Cultures by Artifact Count")
    
    col1, col2 = st.columns([2, 1])
    
    with col1:
        fig = px.bar(
            df, 
            x='culture', 
            y='artifact_count',
            title='Artifact Distribution by Culture',
            labels={'culture': 'Culture', 'artifact_count': 'Count'}
        )
        st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        st.dataframe(df, use_container_width=True)

# Add metrics
if not df.empty:
    total_artifacts = df['artifact_count'].sum()
    avg_per_culture = df['artifact_count'].mean()
    
    st.metric("Total Artifacts", f"{total_artifacts:,}")
    st.metric("Average per Culture", f"{avg_per_culture:.0f}")
```

## Common Patterns

### Paginated API Collection

```python
def collect_all_artifacts(api_key, max_pages=10):
    """Collect multiple pages of artifacts"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page, size=100)
        all_records.extend(data['records'])
        
        # Check if more pages exist
        total_pages = data['info']['pages']
        if page >= total_pages:
            break
    
    return all_records
```

### Error Handling for API Requests

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

## Troubleshooting

**API Rate Limiting:**
- Harvard API has rate limits; add delays between requests
- Use `time.sleep(0.5)` between paginated calls

**Database Connection Issues:**
- Verify environment variables are loaded: `from dotenv import load_dotenv; load_dotenv()`
- Check firewall/whitelist settings for TiDB Cloud
- Ensure correct port (3306 for MySQL, 4000 for TiDB)

**Missing Data in Queries:**
- Some artifacts lack certain fields (culture, period, etc.)
- Always filter or handle 'Unknown' values in analytics
- Use `WHERE field IS NOT NULL AND field != 'Unknown'`

**Streamlit Caching:**
```python
@st.cache_data(ttl=3600)
def cached_query(query):
    return execute_analytics_query(query)
```

**Memory Issues with Large Datasets:**
- Process data in chunks for large collections
- Use pagination limits wisely
- Consider data sampling for exploratory analysis
