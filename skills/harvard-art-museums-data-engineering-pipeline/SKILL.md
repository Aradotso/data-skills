---
name: harvard-art-museums-data-engineering-pipeline
description: Build end-to-end ETL pipelines with Harvard Art Museums API data using Python, SQL, and Streamlit analytics dashboards
triggers:
  - build an ETL pipeline with Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data
  - build data engineering project with museum collections
  - set up SQL analytics for art museum data
  - create Streamlit visualization for artifact data
  - implement batch data processing for Harvard API
  - design relational database for museum artifacts
---

# Harvard Art Museums Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL patterns, relational database design, SQL analytics, and interactive visualization using Streamlit.

**Architecture Flow:** API → ETL → SQL Database → Analytics → Visualization

## Installation

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
export DB_NAME="your_database_name"
```

### Required Dependencies

```txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Configuration

### API Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

def get_db_connection():
    return mysql.connector.connect(**db_config)
```

## Database Schema

### Table Definitions

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT,
    technique VARCHAR(500),
    creditline TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    imagecopyright TEXT,
    renditionnumber VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    try:
        response = requests.get(BASE_URL, params=params)
        response.raise_for_status()
        data = response.json()
        return data.get('records', []), data.get('info', {})
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
        return [], {}

def collect_all_artifacts(api_key, total_records=500):
    """
    Collect artifacts with rate limiting
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        records, info = fetch_artifacts(api_key, page, size)
        if not records:
            break
        all_artifacts.extend(records)
        page += 1
        time.sleep(0.5)  # Rate limiting
    
    return all_artifacts[:total_records]
```

### Transform: Process JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON artifacts into relational dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'division': artifact.get('division', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0),
            'technique': artifact.get('technique', '')[:500],
            'creditline': artifact.get('creditline', '')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        images = artifact.get('images', [])
        for image in images:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl', '')[:1000],
                'iiifbaseuri': image.get('iiifbaseuri', '')[:1000],
                'imagecopyright': image.get('copyright', ''),
                'renditionnumber': image.get('renditionnumber', '')
            }
            media_records.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_data)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Batch Insert into Database

```python
def batch_insert_metadata(df, connection):
    """
    Batch insert artifact metadata
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, division, department, 
     dated, accessionyear, totalpageviews, totaluniquepageviews, technique, creditline)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")

def batch_insert_media(df, connection):
    """
    Batch insert media records
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, iiifbaseuri, imagecopyright, renditionnumber)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")

def batch_insert_colors(df, connection):
    """
    Batch insert color records
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")
```

## SQL Analytics Queries

### Common Analytical Patterns

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "most_viewed_artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Images' ELSE 'No Images' END as status,
            COUNT(*) as count
        FROM (
            SELECT a.id, COUNT(m.media_id) as media_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) as img_stats
        GROUP BY status
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "department_statistics": """
        SELECT department, COUNT(*) as artifact_count, 
               AVG(totalpageviews) as avg_views
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

def execute_analytics_query(query_name, connection):
    """
    Execute an analytics query and return results as DataFrame
    """
    cursor = connection.cursor()
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Data Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics Dashboard", "Data Visualization"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "Analytics Dashboard":
        show_analytics_dashboard()
    else:
        show_visualization_page()

def show_data_collection_page():
    st.header("📥 Data Collection")
    
    num_records = st.number_input("Number of records to collect", 
                                   min_value=100, max_value=5000, 
                                   value=500, step=100)
    
    if st.button("Start ETL Process"):
        with st.spinner("Collecting data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            artifacts = collect_all_artifacts(api_key, num_records)
            st.success(f"Collected {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            conn = get_db_connection()
            batch_insert_metadata(metadata_df, conn)
            batch_insert_media(media_df, conn)
            batch_insert_colors(colors_df, conn)
            conn.close()
            st.success("Data loaded successfully!")

def show_analytics_dashboard():
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analytics Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = execute_analytics_query(query_name, conn)
        conn.close()
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=f"{query_name.replace('_', ' ').title()}")
            st.plotly_chart(fig, use_container_width=True)

def show_visualization_page():
    st.header("📈 Data Visualizations")
    
    conn = get_db_connection()
    
    col1, col2 = st.columns(2)
    
    with col1:
        df_culture = execute_analytics_query("artifacts_by_culture", conn)
        fig1 = px.bar(df_culture, x='culture', y='count',
                     title="Top Cultures by Artifact Count")
        st.plotly_chart(fig1, use_container_width=True)
    
    with col2:
        df_colors = execute_analytics_query("color_distribution", conn)
        fig2 = px.pie(df_colors, names='color', values='count',
                     title="Color Distribution")
        st.plotly_chart(fig2, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Launch Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for rate limits
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        return result[0] == 1
    except Exception as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handle Missing Data
```python
# Safely extract nested fields
def safe_get(data, keys, default=''):
    """
    Safely navigate nested dictionary
    """
    for key in keys:
        if isinstance(data, dict):
            data = data.get(key, default)
        else:
            return default
    return data if data is not None else default

# Usage
culture = safe_get(artifact, ['culture'], 'Unknown')
```

### Memory Optimization for Large Datasets
```python
# Process data in chunks
def process_in_chunks(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        
        conn = get_db_connection()
        batch_insert_metadata(metadata_df, conn)
        batch_insert_media(media_df, conn)
        batch_insert_colors(colors_df, conn)
        conn.close()
```

## Best Practices

1. **Use connection pooling** for database operations
2. **Implement proper error handling** for API calls
3. **Cache API responses** to reduce redundant calls
4. **Validate data** before inserting into database
5. **Use indexes** on frequently queried columns
6. **Monitor API quota** usage
7. **Implement logging** for debugging ETL processes
