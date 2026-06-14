---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, Streamlit, and SQL
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - set up analytics dashboard for museum collections
  - extract and transform Harvard Art Museums data
  - build a Streamlit app for art museum analytics
  - query Harvard artifacts using SQL and Python
  - visualize museum collection data with Plotly
  - implement ETL for cultural heritage data
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, and interactive visualization using Streamlit.

## What This Project Does

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:
- Fetches artifact data from the Harvard Art Museums API with pagination
- Transforms nested JSON into relational database tables
- Stores data in MySQL/TiDB Cloud databases
- Runs analytical SQL queries on museum collections
- Visualizes results through interactive Streamlit dashboards

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
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=50)
print(f"Total artifacts available: {info['totalrecords']}")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    def __init__(self, db_config):
        self.db_config = db_config
        
    def extract(self, api_key, num_pages=5):
        """Extract data from API"""
        all_artifacts = []
        
        for page in range(1, num_pages + 1):
            artifacts, _ = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(artifacts)
            
        return all_artifacts
    
    def transform(self, artifacts):
        """Transform nested JSON to relational format"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for artifact in artifacts:
            # Metadata
            metadata = {
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
                'url': artifact.get('url')
            }
            metadata_list.append(metadata)
            
            # Media
            images = artifact.get('images', [])
            for img in images:
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'baseimageurl': img.get('baseimageurl'),
                    'iiifbaseuri': img.get('iiifbaseuri')
                })
            
            # Colors
            colors = artifact.get('colors', [])
            for color in colors:
                colors_list.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum')
                })
        
        return (
            pd.DataFrame(metadata_list),
            pd.DataFrame(media_list),
            pd.DataFrame(colors_list)
        )
    
    def load(self, metadata_df, media_df, colors_df):
        """Load data into MySQL database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        # Load metadata
        for _, row in metadata_df.iterrows():
            query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, division, dated, period, technique, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            query = """
                INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri)
                VALUES (%s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            query = """
                INSERT INTO artifactcolors (artifact_id, color, spectrum)
                VALUES (%s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        conn.commit()
        cursor.close()
        conn.close()

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardETL(db_config)
artifacts = etl.extract(os.getenv('HARVARD_API_KEY'), num_pages=3)
metadata_df, media_df, colors_df = etl.transform(artifacts)
etl.load(metadata_df, media_df, colors_df)
```

### 3. SQL Analytics Queries

```python
def run_analytics_query(query_name, db_config):
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        
        'color_distribution': """
            SELECT color, spectrum, COUNT(*) as count
            FROM artifactcolors
            GROUP BY color, spectrum
            ORDER BY count DESC
            LIMIT 30
        """,
        
        'media_availability': """
            SELECT 
                CASE 
                    WHEN baseimageurl IS NOT NULL THEN 'Has Image'
                    ELSE 'No Image'
                END as media_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY media_status
        """,
        
        'artifacts_with_technique': """
            SELECT technique, COUNT(*) as count
            FROM artifactmetadata
            WHERE technique IS NOT NULL
            GROUP BY technique
            ORDER BY count DESC
            LIMIT 15
        """
    }
    
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df

# Usage
result_df = run_analytics_query('artifacts_by_culture', db_config)
print(result_df)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for configuration
st.sidebar.header("Configuration")
api_key = st.sidebar.text_input("API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))

# ETL Controls
st.sidebar.header("ETL Pipeline")
num_pages = st.sidebar.slider("Number of pages to fetch", 1, 10, 3)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL..."):
        etl = HarvardETL(db_config)
        artifacts = etl.extract(api_key, num_pages=num_pages)
        metadata_df, media_df, colors_df = etl.transform(artifacts)
        etl.load(metadata_df, media_df, colors_df)
        st.success(f"Loaded {len(metadata_df)} artifacts!")

# Analytics Section
st.header("📊 Analytics Queries")

query_options = [
    "Artifacts by Culture",
    "Artifacts by Century",
    "Artifacts by Department",
    "Color Distribution",
    "Media Availability",
    "Artifacts with Technique"
]

selected_query = st.selectbox("Select Analysis", query_options)

query_map = {
    "Artifacts by Culture": "artifacts_by_culture",
    "Artifacts by Century": "artifacts_by_century",
    "Artifacts by Department": "artifacts_by_department",
    "Color Distribution": "color_distribution",
    "Media Availability": "media_availability",
    "Artifacts with Technique": "artifacts_with_technique"
}

if st.button("Run Query"):
    df = run_analytics_query(query_map[selected_query], db_config)
    
    st.subheader("Query Results")
    st.dataframe(df)
    
    # Visualization
    if len(df.columns) == 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=selected_query
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_fetch_artifacts(api_key, total_pages, batch_size=5, delay=1):
    """Fetch artifacts in batches to respect rate limits"""
    all_artifacts = []
    
    for i in range(0, total_pages, batch_size):
        batch_end = min(i + batch_size, total_pages)
        
        for page in range(i + 1, batch_end + 1):
            artifacts, _ = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(artifacts)
            time.sleep(delay)  # Rate limiting
        
        print(f"Fetched pages {i+1} to {batch_end}")
    
    return all_artifacts
```

### Data Quality Checks

```python
def validate_artifacts(df):
    """Validate artifact data quality"""
    issues = []
    
    # Check for missing IDs
    if df['id'].isna().any():
        issues.append("Missing artifact IDs detected")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append(f"Found {df['id'].duplicated().sum()} duplicate IDs")
    
    # Check data completeness
    null_counts = df.isnull().sum()
    for col, count in null_counts.items():
        if count > len(df) * 0.5:
            issues.append(f"Column '{col}' has >50% missing values")
    
    return issues

# Usage
metadata_df, _, _ = etl.transform(artifacts)
issues = validate_artifacts(metadata_df)
if issues:
    print("Data quality issues:", issues)
```

## Troubleshooting

### API Rate Limiting
If you encounter 429 errors, add delays between requests:
```python
time.sleep(1)  # Wait 1 second between requests
```

### Database Connection Issues
Verify connection parameters:
```python
try:
    conn = mysql.connector.connect(**db_config)
    print("Database connection successful")
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

### Empty Results
Check API key validity and query parameters:
```python
response = requests.get(base_url, params=params)
print(f"Status: {response.status_code}")
print(f"Response: {response.json()}")
```

### Memory Issues with Large Datasets
Process in smaller batches:
```python
chunk_size = 100
for i in range(0, len(artifacts), chunk_size):
    chunk = artifacts[i:i+chunk_size]
    # Process chunk
```
