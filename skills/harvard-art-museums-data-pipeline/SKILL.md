---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with MySQL/TiDB and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for art museum data
  - set up Harvard artifacts analytics dashboard
  - extract and analyze Harvard Art Museums collection data
  - build a Streamlit app with museum API data
  - implement SQL analytics for Harvard art collection
  - create data engineering project with museum artifacts
  - visualize Harvard Art Museums data with Plotly
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics application that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud
- Executes analytical SQL queries
- Visualizes results in an interactive Streamlit dashboard

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

### 1. Harvard Art Museums API Key

Get your API key from: https://www.harvardartmuseums.org/collections/api

Set it as an environment variable:
```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or use a `.env` file:
```
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

**MySQL/TiDB Connection:**
```python
import os
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

**Environment variables:**
```bash
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
export DB_PORT="3306"
```

## Database Schema

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    department VARCHAR(255),
    classification VARCHAR(255),
    technique VARCHAR(500),
    dated VARCHAR(255),
    accession_number VARCHAR(100),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
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

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        
        # Rate limiting
        time.sleep(0.5)
        
        return {
            'records': data.get('records', []),
            'total_pages': data.get('info', {}).get('pages', 0),
            'total_records': data.get('info', {}).get('totalrecords', 0)
        }
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

def collect_all_artifacts(api_key, max_pages=10):
    """Collect multiple pages of artifact data"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        result = fetch_artifacts(api_key, page=page)
        
        if result and result['records']:
            all_artifacts.extend(result['records'])
        else:
            break
    
    return all_artifacts
```

### Transform: Data Normalization

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data into metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:100],
            'department': artifact.get('department', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'dated': artifact.get('dated', '')[:255],
            'accession_number': artifact.get('accessionyear', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Transform media data into DataFrame"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': img.get('baseimageurl', '')[:1000],
                'iiifbaseuri': img.get('iiifbaseuri', '')[:1000]
            })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """Transform color data into DataFrame"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### Load: Database Insertion

```python
def load_to_database(conn, metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, department, classification, 
         technique, dated, accession_number, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    for _, row in metadata_df.iterrows():
        cursor.execute(metadata_query, tuple(row))
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia (artifact_id, media_type, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
    """
    
    for _, row in media_df.iterrows():
        cursor.execute(media_query, tuple(row))
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    for _, row in colors_df.iterrows():
        cursor.execute(colors_query, tuple(row))
    
    conn.commit()
    cursor.close()
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for API configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        # Database connection
        if st.button("Test DB Connection"):
            try:
                conn = mysql.connector.connect(**db_config)
                st.success("Database connected!")
                conn.close()
            except Exception as e:
                st.error(f"Connection failed: {e}")
    
    # ETL Section
    st.header("📥 Data Collection")
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.slider("Pages to fetch", 1, 50, 5)
    
    with col2:
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data..."):
                artifacts = collect_all_artifacts(api_key, max_pages=num_pages)
                
                if artifacts:
                    st.success(f"Collected {len(artifacts)} artifacts")
                    
                    # Transform
                    metadata_df = transform_metadata(artifacts)
                    media_df = transform_media(artifacts)
                    colors_df = transform_colors(artifacts)
                    
                    # Load
                    conn = mysql.connector.connect(**db_config)
                    load_to_database(conn, metadata_df, media_df, colors_df)
                    conn.close()
                    
                    st.success("ETL Complete!")

    # Analytics Section
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century 
            ORDER BY count DESC
        """,
        "Color Distribution": """
            SELECT color, AVG(percent) as avg_percent, COUNT(*) as count
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 15
        """,
        "Top Departments": """
            SELECT department, COUNT(*) as artifact_count,
                   AVG(totalpageviews) as avg_views
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        "Media Availability": """
            SELECT 
                COUNT(DISTINCT m.artifact_id) as with_media,
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts
            FROM artifactmedia m
        """
    }
    
    query_choice = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = mysql.connector.connect(**db_config)
        df = pd.read_sql(queries[query_choice], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2 and len(df) > 0:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_choice)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common SQL Analytics Queries

```sql
-- Artifacts with most pageviews
SELECT title, culture, totalpageviews
FROM artifactmetadata
ORDER BY totalpageviews DESC
LIMIT 20;

-- Color palette analysis
SELECT c.color, COUNT(DISTINCT c.artifact_id) as artifact_count,
       AVG(c.percent) as avg_coverage
FROM artifactcolors c
GROUP BY c.color
HAVING artifact_count > 5
ORDER BY artifact_count DESC;

-- Classification breakdown
SELECT classification, COUNT(*) as count,
       AVG(totalpageviews) as avg_popularity
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY count DESC;

-- Media type distribution
SELECT media_type, COUNT(*) as count
FROM artifactmedia
GROUP BY media_type;

-- Period timeline
SELECT period, century, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE period IS NOT NULL AND century IS NOT NULL
GROUP BY period, century
ORDER BY century;
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Troubleshooting

**API Rate Limiting:**
```python
# Add delays between requests
import time
time.sleep(1)  # Wait 1 second between API calls
```

**Database Connection Issues:**
```python
# Test connection separately
try:
    conn = mysql.connector.connect(**db_config)
    print("Connected successfully")
    conn.close()
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

**Memory Issues with Large Datasets:**
```python
# Process data in chunks
def batch_insert(conn, df, table_name, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch
```

**Missing Data Handling:**
```python
# Handle None/null values
df = df.fillna('')
df = df.replace({None: ''})
```

**API Response Validation:**
```python
def safe_get(data, *keys, default=''):
    """Safely navigate nested dictionaries"""
    for key in keys:
        try:
            data = data[key]
        except (KeyError, TypeError):
            return default
    return data

# Usage
title = safe_get(artifact, 'title', default='Untitled')
```
