---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - set up ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit
  - build data engineering pipeline for art collections
  - query Harvard museum data with SQL
  - visualize artifact metadata with plotly
  - extract transform load museum artifacts
  - analyze art collection data with Python
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact metadata visualization.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational SQL tables
- **SQL Analytics**: Executes 20+ predefined analytical queries on artifact collections
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations for real-time insights

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### Environment Variables

Set up your configuration using environment variables:

```bash
# Harvard API Key (get from https://www.harvardartmuseums.org/collections/api)
export HARVARD_API_KEY="your-api-key-here"

# Database Configuration
export DB_HOST="your-db-host"
export DB_PORT="3306"
export DB_USER="your-username"
export DB_PASSWORD="your-password"
export DB_NAME="harvard_artifacts"
```

### Database Setup

Create the database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
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

## Key Commands

### Run the Streamlit App

```bash
streamlit run app.py
```

### Run ETL Pipeline Standalone

```python
import os
import requests
import pandas as pd
import mysql.connector

# API Configuration
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

# Fetch artifacts with pagination
def fetch_artifacts(num_records=100):
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': API_KEY,
            'size': per_page,
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) < per_page:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]

# Transform and load data
def etl_pipeline(num_records=100):
    # Extract
    artifacts = fetch_artifacts(num_records)
    
    # Transform
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Metadata
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        })
        
        # Media
        if artifact.get('primaryimageurl'):
            media_list.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl')
            })
        
        # Colors
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    # Load
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = conn.cursor()
    
    # Insert metadata
    for item in metadata_list:
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, dated, department, division, 
             classification, medium, dimensions, creditline, accessionyear, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(item.values()))
    
    # Insert media
    for item in media_list:
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, primaryimageurl)
            VALUES (%s, %s, %s, %s)
        """, tuple(item.values()))
    
    # Insert colors
    for item in colors_list:
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(item.values()))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return len(metadata_list)

# Execute ETL
if __name__ == "__main__":
    count = etl_pipeline(num_records=500)
    print(f"Successfully loaded {count} artifacts")
```

## Common Analytics Queries

### Artifacts by Culture

```python
import pandas as pd
import mysql.connector
import os

def execute_query(query):
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Top 10 cultures by artifact count
query = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

df = execute_query(query)
print(df)
```

### Artifacts with Images

```python
query = """
SELECT 
    COUNT(DISTINCT a.id) as total_artifacts,
    COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
    ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as percentage
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
"""

df = execute_query(query)
```

### Color Distribution Analysis

```python
query = """
SELECT 
    c.color,
    COUNT(*) as usage_count,
    AVG(c.percent) as avg_percent
FROM artifactcolors c
WHERE c.color IS NOT NULL
GROUP BY c.color
ORDER BY usage_count DESC
LIMIT 15
"""

df = execute_query(query)
```

## Streamlit Dashboard Pattern

```python
import streamlit as st
import plotly.express as px

st.title("Harvard Artifacts Analytics Dashboard")

# Query selection
queries = {
    "Artifacts by Culture": "SELECT culture, COUNT(*) as count FROM artifactmetadata WHERE culture IS NOT NULL GROUP BY culture ORDER BY count DESC LIMIT 10",
    "Department Distribution": "SELECT department, COUNT(*) as count FROM artifactmetadata WHERE department IS NOT NULL GROUP BY department ORDER BY count DESC",
    "Color Analysis": "SELECT color, COUNT(*) as count FROM artifactcolors WHERE color IS NOT NULL GROUP BY color ORDER BY count DESC LIMIT 15"
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

# Execute and display
if st.button("Run Query"):
    df = execute_query(queries[selected_query])
    
    # Show table
    st.dataframe(df)
    
    # Visualize
    if len(df) > 0:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                     title=selected_query)
        st.plotly_chart(fig)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:
            wait_time = (attempt + 1) * 2
            time.sleep(wait_time)
        else:
            break
    return None
```

### Database Connection Issues

```python
def get_db_connection():
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        return conn
    except mysql.connector.Error as err:
        print(f"Database connection error: {err}")
        return None
```

### Handle Missing Data

```python
def safe_get(data, key, default=None):
    """Safely extract data from nested dictionaries"""
    try:
        return data.get(key, default)
    except (AttributeError, KeyError):
        return default

# Usage in ETL
metadata = {
    'id': safe_get(artifact, 'id'),
    'title': safe_get(artifact, 'title', 'Untitled'),
    'culture': safe_get(artifact, 'culture', 'Unknown')
}
```

## Best Practices

1. **Always use parameterized queries** to prevent SQL injection
2. **Batch insert operations** for better performance (100-1000 records per batch)
3. **Cache API responses** to avoid redundant calls
4. **Use connection pooling** for concurrent Streamlit sessions
5. **Handle NULL values** in SQL queries with `COALESCE` or `IS NOT NULL`
6. **Index foreign keys** for faster joins between tables
