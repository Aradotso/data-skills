---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering app with Harvard Art Museums API
  - set up artifact data collection and analytics dashboard
  - implement SQL analytics for art collection data
  - build a Streamlit app for museum data visualization
  - create an end-to-end data pipeline with API integration
  - analyze Harvard Art Museums collection data
  - design relational database for artifact metadata
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming nested JSON into relational structures, loading into SQL databases (MySQL/TiDB), and visualizing insights through an interactive Streamlit dashboard. It includes 20+ predefined analytical queries and automated ETL pipelines with pagination and rate limiting.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

Configure MySQL or TiDB Cloud credentials:

```python
# Database connection parameters (use environment variables)
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

### 3. Environment Variables

Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Database Schema

The ETL pipeline creates three related tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    url TEXT,
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id),
    INDEX idx_artifact (artifact_id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_name VARCHAR(100),
    hex_code VARCHAR(10),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id),
    INDEX idx_artifact (artifact_id)
);
```

## Core Components

### ETL Pipeline

**Extract - API Data Collection:**

```python
import requests
import pandas as pd
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    pages_needed = (num_records + page_size - 1) // page_size
    
    for page in range(1, pages_needed + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            all_artifacts.extend(data.get('records', []))
            
            # Rate limiting
            time.sleep(0.5)
            
            if len(all_artifacts) >= num_records:
                break
                
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts[:num_records]
```

**Transform - Data Normalization:**

```python
def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Metadata table
        metadata_records.append({
            'artifact_id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Media table
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri')
            })
        
        # Colors table
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color_name': color.get('color'),
                'hex_code': color.get('hex'),
                'percentage': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

**Load - Batch Insert to SQL:**

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """
    Load transformed data into MySQL database
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Load metadata
        insert_metadata = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, period, century, classification, 
         department, division, technique, medium, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(insert_metadata, df_metadata.values.tolist())
        
        # Load media
        insert_media = """
        INSERT INTO artifactmedia 
        (artifact_id, media_type, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(insert_media, df_media.values.tolist())
        
        # Load colors
        insert_colors = """
        INSERT INTO artifactcolors 
        (artifact_id, color_name, hex_code, percentage)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(insert_colors, df_colors.values.tolist())
        
        connection.commit()
        print(f"✓ Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### Analytical SQL Queries

**Sample Analytics Queries:**

```python
# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century distribution
query_centuries = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Media availability analysis
query_media_coverage = """
SELECT 
    COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
    (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
    ROUND(COUNT(DISTINCT am.artifact_id) * 100.0 / 
          (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
FROM artifactmedia am
"""

# Top color usage
query_color_usage = """
SELECT 
    color_name,
    COUNT(*) as usage_count,
    AVG(percentage) as avg_percentage
FROM artifactcolors
GROUP BY color_name
ORDER BY usage_count DESC
LIMIT 15
"""

# Department-wise classification breakdown
query_dept_classification = """
SELECT 
    department,
    classification,
    COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC
```

### Streamlit Dashboard

**Main Application Structure:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualization_page()

def show_etl_page():
    st.header("Data Collection & ETL")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_records = st.number_input("Number of records", 10, 1000, 100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            artifacts = fetch_artifacts(api_key, num_records)
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            load_to_database(df_meta, df_media, df_colors, DB_CONFIG)
            st.success(f"✓ Processed {len(df_meta)} artifacts")

def show_analytics_page():
    st.header("SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_centuries,
        "Media Coverage": query_media_coverage,
        "Color Usage": query_color_usage
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        df_result = execute_query(queries[selected_query])
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], 
                        y=df_result.columns[1])
            st.plotly_chart(fig, use_container_width=True)

def execute_query(query):
    connection = mysql.connector.connect(**DB_CONFIG)
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def incremental_load(api_key, last_modified_date):
    """Load only new/updated artifacts"""
    params = {
        'apikey': api_key,
        'modified_after': last_modified_date
    }
    response = requests.get(base_url, params=params)
    return response.json().get('records', [])
```

### Error Handling & Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_run(api_key, num_records):
    try:
        logger.info(f"Starting ETL for {num_records} records")
        artifacts = fetch_artifacts(api_key, num_records)
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        load_to_database(df_meta, df_media, df_colors, DB_CONFIG)
        logger.info("ETL completed successfully")
    except Exception as e:
        logger.error(f"ETL failed: {e}")
        raise
```

## Troubleshooting

**API Rate Limiting:**
```python
# Increase delay between requests
time.sleep(1.0)  # 1 second between calls
```

**Database Connection Issues:**
```python
# Test connection
try:
    connection = mysql.connector.connect(**DB_CONFIG)
    print("✓ Database connected")
except Error as e:
    print(f"✗ Connection failed: {e}")
```

**Missing Data Fields:**
```python
# Use safe dictionary access
culture = artifact.get('culture', 'Unknown')
century = artifact.get('century', 'Undated')
```

**Memory Issues with Large Datasets:**
```python
# Process in smaller batches
batch_size = 100
for i in range(0, len(all_artifacts), batch_size):
    batch = all_artifacts[i:i+batch_size]
    df_meta, df_media, df_colors = transform_artifacts(batch)
    load_to_database(df_meta, df_media, df_colors, DB_CONFIG)
```
