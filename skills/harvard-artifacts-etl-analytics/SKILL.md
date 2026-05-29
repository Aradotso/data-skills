---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, MySQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and museum data
  - query Harvard artifacts collection database
  - set up SQL database for art museum artifacts
  - visualize museum artifact analytics with Plotly
  - implement batch data collection from Harvard API
  - design relational schema for artifact metadata
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into MySQL/TiDB, and building interactive analytics dashboards with Streamlit.

## What It Does

- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Creates interactive Plotly charts from query results

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Set up your credentials using environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Database Setup

Create the MySQL/TiDB database:

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
    accession_number VARCHAR(100),
    division VARCHAR(200)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    base_url TEXT,
    alt_text TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Patterns

### Extracting Data from Harvard API

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Collect multiple pages
def collect_artifacts(num_pages=5):
    all_artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
    return all_artifacts
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(artifacts):
    """Transform nested JSON to relational format"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'period': artifact.get('period', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'accession_number': artifact.get('accessionyear', 'Unknown')[:100],
            'division': artifact.get('division', 'Unknown')[:200]
        }
        metadata_list.append(metadata)
        
        # Extract media
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'base_url': img.get('baseimageurl', ''),
                'alt_text': img.get('alttext', '')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
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

### Loading Data to SQL

```python
def load_to_sql(df_metadata, df_media, df_colors):
    """Batch insert transformed data to MySQL"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_sql = """
    INSERT IGNORE INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, dated, accession_number, division)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(metadata_sql, df_metadata.values.tolist())
    
    # Insert media
    media_sql = """
    INSERT INTO artifactmedia (artifact_id, media_type, base_url, alt_text)
    VALUES (%s, %s, %s, %s)
    """
    cursor.executemany(media_sql, df_media.values.tolist())
    
    # Insert colors
    colors_sql = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    cursor.executemany(colors_sql, df_colors.values.tolist())
    
    connection.commit()
    cursor.close()
    connection.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

st.title("Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Choose a section",
    ["Data Collection", "SQL Analytics", "Visualizations"]
)

if page == "Data Collection":
    st.header("Collect Artifact Data")
    
    num_pages = st.number_input("Number of pages to collect", 1, 20, 5)
    
    if st.button("Start Collection"):
        with st.spinner("Collecting artifacts..."):
            artifacts = collect_artifacts(num_pages)
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            load_to_sql(df_meta, df_media, df_colors)
            st.success(f"Loaded {len(df_meta)} artifacts!")

elif page == "SQL Analytics":
    st.header("Run Analytics Queries")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
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
        "Top Colors in Collection": """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 15
        """,
        "Media Availability": """
            SELECT 
                CASE WHEN EXISTS(SELECT 1 FROM artifactmedia WHERE artifact_id = m.id) 
                     THEN 'Has Media' 
                     ELSE 'No Media' 
                END as media_status,
                COUNT(*) as count
            FROM artifactmetadata m
            GROUP BY media_status
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df_result = pd.read_sql(queries[selected_query], connection)
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig)
        
        connection.close()
```

## Common Analytics Queries

### Department Distribution

```sql
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
GROUP BY department
ORDER BY artifact_count DESC;
```

### Color Spectrum Analysis

```sql
SELECT spectrum, COUNT(DISTINCT artifact_id) as unique_artifacts
FROM artifactcolors
GROUP BY spectrum
ORDER BY unique_artifacts DESC;
```

### Classification by Period

```sql
SELECT classification, period, COUNT(*) as count
FROM artifactmetadata
WHERE classification != 'Unknown' AND period != 'Unknown'
GROUP BY classification, period
ORDER BY count DESC
LIMIT 20;
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests:
```python
import time
time.sleep(0.5)  # 500ms delay between API calls
```

**Connection Pool Exhaustion**: Use connection pooling:
```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)
```

**Large Dataset Memory Issues**: Process in chunks:
```python
chunk_size = 1000
for i in range(0, len(artifacts), chunk_size):
    chunk = artifacts[i:i+chunk_size]
    df_meta, df_media, df_colors = transform_artifacts(chunk)
    load_to_sql(df_meta, df_media, df_colors)
```

**UTF-8 Encoding Errors**: Ensure proper charset:
```python
connection = mysql.connector.connect(
    charset='utf8mb4',
    use_unicode=True,
    # ... other params
)
```
