---
name: harvard-artifacts-collection-analytics-etl
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts data
  - create a data engineering pipeline with Harvard Art Museums API
  - set up artifact collection analytics with SQL and visualization
  - extract and transform Harvard museum data into database
  - build a Streamlit dashboard for art museum analytics
  - implement museum artifact data pipeline with Python
  - create SQL analytics for Harvard Art Museums collection
  - visualize museum artifact data with interactive dashboards
---

# Harvard Artifacts Collection Analytics ETL

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates a production-ready ETL pipeline that extracts artifact data, transforms nested JSON structures into relational tables, loads data into SQL databases (MySQL/TiDB), and provides interactive analytics through a Streamlit dashboard with Plotly visualizations.

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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

**Required packages**:
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Project Structure

```
harvard-artifacts-collection/
├── app.py                    # Main Streamlit application
├── etl_pipeline.py          # ETL logic for data extraction & transformation
├── database_setup.py        # SQL schema and connection management
├── analytics_queries.py     # Predefined SQL analytical queries
├── config.py               # Configuration and environment variables
└── requirements.txt
```

## Database Schema

The pipeline creates three relational tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    division VARCHAR(255),
    medium VARCHAR(500),
    period VARCHAR(255),
    technique VARCHAR(500),
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## ETL Pipeline Implementation

### 1. Extract Data from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def extract_artifacts(num_artifacts=100):
    """Extract artifact data from Harvard Art Museums API with pagination."""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'page': page,
            'size': min(size, num_artifacts - len(artifacts))
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if not data.get('records') or len(artifacts) >= num_artifacts:
                break
                
            page += 1
        else:
            print(f"API Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]
```

### 2. Transform Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes."""
    
    # Transform metadata
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata = {
            'artifact_id': artifact_id,
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'description': artifact.get('description', ''),
            'division': artifact.get('division', '')[:255],
            'medium': artifact.get('medium', '')[:500],
            'period': artifact.get('period', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'url': artifact.get('url', '')[:500]
        }
        metadata_records.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact_id,
            'baseimageurl': artifact.get('baseimageurl', '')[:500],
            'iiifbaseuri': artifact.get('iiifbaseuri', '')[:500],
            'primaryimageurl': artifact.get('primaryimageurl', '')[:500]
        }
        media_records.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color_data in colors:
            color_record = {
                'artifact_id': artifact_id,
                'color': color_data.get('color', '')[:50],
                'percentage': color_data.get('percent', 0.0)
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Load Data to SQL

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection."""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_data(metadata_df, media_df, colors_df):
    """Batch insert data into SQL tables."""
    connection = create_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, department, 
         dated, description, division, medium, period, technique, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        print("Data loaded successfully!")
        return True
        
    except Error as e:
        print(f"Load error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## Analytics Queries

```python
# Example analytical queries

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' 
                 ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Classification Overview": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query_name):
    """Execute analytical query and return results as DataFrame."""
    connection = create_connection()
    if not connection:
        return None
    
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        connection.close()
```

## Streamlit Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for ETL operations
    st.sidebar.header("ETL Pipeline")
    
    num_artifacts = st.sidebar.number_input(
        "Number of artifacts to collect", 
        min_value=10, 
        max_value=1000, 
        value=100
    )
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            raw_data = extract_artifacts(num_artifacts)
            st.success(f"Extracted {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            success = load_data(metadata_df, media_df, colors_df)
            if success:
                st.success("Data loaded to SQL database!")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Analysis"):
        results_df = execute_query(query_choice)
        
        if results_df is not None and not results_df.empty:
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) == 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=query_choice
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def extract_artifacts_with_rate_limit(num_artifacts=100, delay=0.5):
    """Extract with rate limiting to avoid API throttling."""
    artifacts = []
    page = 1
    
    while len(artifacts) < num_artifacts:
        # Make request
        data = fetch_page(page)
        artifacts.extend(data)
        page += 1
        
        # Rate limit
        time.sleep(delay)
    
    return artifacts
```

### Handling Missing Data

```python
def safe_extract(artifact, key, default='', max_length=None):
    """Safely extract values with defaults and length constraints."""
    value = artifact.get(key, default)
    if max_length and isinstance(value, str):
        return value[:max_length]
    return value
```

## Troubleshooting

**API Key Error**: Ensure `HARVARD_API_KEY` is set in environment variables. Get your key from https://docs.harvardartmuseums.org/

**Database Connection Issues**: Verify database credentials and ensure the database exists:
```python
# Test connection
connection = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)
cursor = connection.cursor()
cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
```

**Memory Issues with Large Datasets**: Process data in batches:
```python
def batch_load(df, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df[i:i+batch_size]
        load_batch_to_db(batch)
```

**Empty Query Results**: Check if ETL pipeline has run and data exists in tables:
```python
# Verify data
cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
count = cursor.fetchone()[0]
print(f"Total artifacts in database: {count}")
```
