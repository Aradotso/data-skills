---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - build analytics dashboard for Harvard artifacts collection
  - set up SQL database for museum artifact data
  - query Harvard Art Museums API and visualize results
  - create Streamlit app for art museum data analytics
  - implement ETL workflow for cultural heritage data
  - analyze Harvard museum artifacts with Python and SQL
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **SQL Database**: Store artifact metadata, media, and color information with proper relationships
- **Analytics Queries**: 20+ predefined SQL queries for insights on artifacts, cultures, centuries, and media
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations for query results

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
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

Get your free API key from: https://www.harvardartmuseums.org/collections/api

Store it in environment variables:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL/TiDB Cloud connection:

```python
# config.py or environment variables
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

### 3. Database Schema

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    objectnumber VARCHAR(100),
    dated VARCHAR(200),
    description TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
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

## Running the Application

```bash
streamlit run app.py
```

The Streamlit app will launch at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max records per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_records - len(artifacts)),
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) < size:
                break  # No more records
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata
    metadata_list = []
    for artifact in raw_data:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'objectnumber': artifact.get('objectnumber'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'url': artifact.get('url')
        })
    
    df_metadata = pd.DataFrame(metadata_list)
    
    # Media
    media_list = []
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        for image in images:
            media_list.append({
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format')
            })
    
    df_media = pd.DataFrame(media_list)
    
    # Colors
    color_list = []
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            })
    
    df_colors = pd.DataFrame(color_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Load to SQL Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(df_metadata, df_media, df_colors, db_config):
    """Batch insert dataframes into SQL database"""
    
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        insert_metadata = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, department, classification, 
         objectnumber, dated, description, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        
        metadata_values = df_metadata.values.tolist()
        cursor.executemany(insert_metadata, metadata_values)
        
        # Insert media
        insert_media = """
        INSERT INTO artifactmedia (artifact_id, baseimageurl, format)
        VALUES (%s, %s, %s)
        """
        
        media_values = df_media.values.tolist()
        cursor.executemany(insert_media, media_values)
        
        # Insert colors
        insert_colors = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
        """
        
        color_values = df_colors.values.tolist()
        cursor.executemany(insert_colors, color_values)
        
        connection.commit()
        print(f"Inserted {len(metadata_values)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 4. Analytical SQL Queries

```python
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as usage_count
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Media Coverage": """
        SELECT 
            COUNT(DISTINCT am.id) as total_artifacts,
            COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
            ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_coverage_percent
        FROM artifactmetadata am
        LEFT JOIN artifactmedia media ON am.id = media.artifact_id
    """,
    
    "Color Spectrum Distribution": """
        SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY count DESC
    """
}

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    try:
        connection = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(query, connection)
        return df_result
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        if connection.is_connected():
            connection.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    
    # Sidebar - Data Collection
    st.sidebar.header("Data Collection")
    num_records = st.sidebar.number_input("Number of artifacts to fetch", 
                                          min_value=10, max_value=1000, value=100)
    
    if st.sidebar.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(num_records)
            df_metadata, df_media, df_colors = transform_artifacts(raw_data)
            load_to_database(df_metadata, df_media, df_colors, DB_CONFIG)
            st.sidebar.success(f"Loaded {len(df_metadata)} artifacts!")
    
    # Main area - Analytics
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[query_name]
            df_result = execute_query(query, DB_CONFIG)
            
            if df_result is not None and not df_result.empty:
                st.subheader("Results")
                st.dataframe(df_result)
                
                # Auto-generate visualization
                if len(df_result.columns) >= 2:
                    x_col = df_result.columns[0]
                    y_col = df_result.columns[1]
                    
                    fig = px.bar(df_result, x=x_col, y=y_col, 
                                title=query_name)
                    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Pipeline Execution

```python
from dotenv import load_dotenv
import os

load_dotenv()

# Configuration
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Execute ETL
raw_data = fetch_artifacts(num_records=500)
df_metadata, df_media, df_colors = transform_artifacts(raw_data)
load_to_database(df_metadata, df_media, df_colors, DB_CONFIG)

# Run analytics
results = execute_query(ANALYTICS_QUERIES["Top 10 Cultures by Artifact Count"], DB_CONFIG)
print(results)
```

### Incremental Data Loading

```python
def incremental_load(start_page=1, num_pages=5):
    """Load data incrementally page by page"""
    for page in range(start_page, start_page + num_pages):
        artifacts = fetch_artifacts_page(page)
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        load_to_database(df_metadata, df_media, df_colors, DB_CONFIG)
        print(f"Loaded page {page}")
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 200:
            return response
        elif response.status_code == 429:  # Rate limited
            wait_time = 2 ** attempt
            time.sleep(wait_time)
    return None
```

### Database Connection Issues
```python
def test_connection(db_config):
    """Test database connectivity"""
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Missing API Key Error
```python
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

This skill enables comprehensive data engineering workflows from API extraction to interactive analytics dashboards using museum artifact data.
