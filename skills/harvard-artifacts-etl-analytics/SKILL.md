---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums API
  - create analytics dashboard with Harvard artifacts data
  - set up data engineering pipeline with museum collections
  - extract and transform Harvard Art Museums data
  - visualize Harvard artifacts with SQL analytics
  - build Streamlit dashboard for museum artifact data
  - implement ETL workflow for art collection data
  - analyze Harvard museum data with SQL queries
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact collection data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational SQL tables
- **SQL Analytics**: 20+ predefined analytical queries for insights on artifacts, cultures, centuries, and media
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts
- **Database Management**: Structured storage in MySQL/TiDB Cloud with proper foreign key relationships

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)

### Setup

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
export DB_NAME="harvard_artifacts"
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

### Database Setup

Create the required database schema:

```python
import mysql.connector
import os

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)
cursor = conn.cursor()

# Create database
cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
cursor.execute(f"USE {os.getenv('DB_NAME')}")

# Create artifact metadata table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    url VARCHAR(500)
)
""")

# Create artifact media table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    artifact_id INT,
    media_id INT,
    media_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

# Create artifact colors table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
```

### API Configuration

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

## Core ETL Pipeline

### Extract Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_pages=5, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
        size: Records per page (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} records")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return artifacts
```

### Transform Data

```python
def transform_artifacts(artifacts):
    """
    Transform raw API data into structured DataFrames
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'url': artifact.get('url', '')[:500]
        })
        
        # Extract media
        for media in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_id': media.get('imageid'),
                'media_url': media.get('baseimageurl', '')[:500],
                'media_type': media.get('format', '')[:100]
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'percentage': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load Data to SQL

```python
def load_to_database(metadata_df, media_df, colors_df, conn):
    """
    Load transformed data into SQL database
    """
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """, tuple(row))
    
    # Load media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, media_id, media_url, media_type)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    print(f"Loaded {len(metadata_df)} artifacts, {len(media_df)} media, {len(colors_df)} colors")
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for ETL controls
st.sidebar.header("Data Collection")
num_pages = st.sidebar.slider("Pages to fetch", 1, 10, 5)
if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Fetching data..."):
        artifacts = fetch_artifacts(API_KEY, num_pages=num_pages)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df, conn)
        st.sidebar.success(f"Loaded {len(metadata_df)} artifacts!")

# Analytics queries
st.header("📊 SQL Analytics")

queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 15
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century 
        ORDER BY count DESC
    """,
    "Top Colors Used": """
        SELECT color, SUM(percentage) as total_percentage, COUNT(*) as occurrences
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY total_percentage DESC 
        LIMIT 10
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY count DESC
    """
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    cursor = conn.cursor()
    cursor.execute(queries[selected_query])
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    results_df = pd.DataFrame(results, columns=columns)
    
    st.dataframe(results_df)
    
    # Auto-generate visualization
    if len(results_df.columns) == 2:
        fig = px.bar(results_df, x=results_df.columns[0], y=results_df.columns[1],
                     title=selected_query)
        st.plotly_chart(fig, use_container_width=True)
```

## Common Analytical Queries

### Artifacts with Most Images

```python
query = """
SELECT 
    am.id, 
    am.title, 
    am.culture, 
    COUNT(ame.media_id) as image_count
FROM artifactmetadata am
JOIN artifactmedia ame ON am.id = ame.artifact_id
GROUP BY am.id, am.title, am.culture
ORDER BY image_count DESC
LIMIT 20
"""
```

### Color Distribution Analysis

```python
query = """
SELECT 
    ac.color,
    AVG(ac.percentage) as avg_percentage,
    COUNT(DISTINCT ac.artifact_id) as artifact_count
FROM artifactcolors ac
GROUP BY ac.color
HAVING artifact_count > 10
ORDER BY avg_percentage DESC
"""
```

### Classification by Period

```python
query = """
SELECT 
    classification,
    period,
    COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL 
  AND period IS NOT NULL
GROUP BY classification, period
ORDER BY count DESC
LIMIT 25
"""
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_artifacts_with_retry(api_key, num_pages=5, size=100, delay=1):
    artifacts = []
    for page in range(1, num_pages + 1):
        try:
            response = requests.get(BASE_URL, params={'apikey': api_key, 'size': size, 'page': page})
            response.raise_for_status()
            artifacts.extend(response.json().get('records', []))
            time.sleep(delay)  # Rate limiting delay
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                print(f"Rate limited. Waiting 60 seconds...")
                time.sleep(60)
                continue
            raise
    return artifacts
```

### Database Connection Issues

```python
from contextlib import contextmanager

@contextmanager
def get_db_connection():
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        autocommit=False
    )
    try:
        yield conn
    finally:
        conn.close()

# Usage
with get_db_connection() as conn:
    load_to_database(metadata_df, media_df, colors_df, conn)
```

### Handling NULL Values

```python
def clean_dataframe(df):
    """Replace None/NaN with empty strings for SQL compatibility"""
    return df.fillna('').astype(str)

metadata_df = clean_dataframe(metadata_df)
```
