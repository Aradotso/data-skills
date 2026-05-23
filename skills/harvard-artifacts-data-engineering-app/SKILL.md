---
name: harvard-artifacts-data-engineering-app
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create ETL pipeline with Harvard Art Museums API
  - set up artifact collection analytics dashboard
  - integrate Harvard museums API with SQL database
  - build Streamlit analytics app for art collection data
  - create data engineering project with museum API
  - implement ETL workflow for artifact metadata
  - visualize museum collection data with SQL queries
---

# Harvard Artifacts Collection Data Engineering Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is a complete data engineering solution that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases, and provides interactive analytics through a Streamlit dashboard. It demonstrates production-grade ETL pipelines, SQL analytics, and data visualization patterns.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to your `.env` file

### Database Setup

The app supports MySQL and TiDB Cloud. Create the database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    period VARCHAR(200),
    provenance TEXT,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        dict: JSON response containing artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_multiple_pages(api_key, num_pages=10):
    """Collect artifacts from multiple pages"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
        print(f"Collected page {page}, total artifacts: {len(all_artifacts)}")
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    """ETL pipeline for Harvard artifact data"""
    
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(**self.db_config)
        return self.connection
    
    def extract_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract artifact metadata"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:200],
                'classification': artifact.get('classification', '')[:200],
                'century': artifact.get('century', '')[:100],
                'dated': artifact.get('dated', '')[:200],
                'department': artifact.get('department', '')[:200],
                'division': artifact.get('division', '')[:200],
                'period': artifact.get('period', '')[:200],
                'provenance': artifact.get('provenance', ''),
                'description': artifact.get('description', '')
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract artifact media/image data"""
        media = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            media.append({
                'artifact_id': artifact_id,
                'baseimageurl': artifact.get('baseimageurl', ''),
                'primaryimageurl': artifact.get('primaryimageurl', ''),
                'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
            })
        
        return pd.DataFrame(media)
    
    def extract_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color data from artifacts"""
        colors = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            color_list = artifact.get('colors', [])
            
            for color in color_list:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                })
        
        return pd.DataFrame(colors)
    
    def load_to_db(self, df: pd.DataFrame, table_name: str):
        """Load dataframe to SQL table"""
        cursor = self.connection.cursor()
        
        # Generate INSERT statement
        cols = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_sql = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data_tuples = [tuple(row) for row in df.values]
        cursor.executemany(insert_sql, data_tuples)
        self.connection.commit()
        
        print(f"Loaded {cursor.rowcount} rows into {table_name}")
        cursor.close()
    
    def run_pipeline(self, artifacts: List[Dict]):
        """Execute full ETL pipeline"""
        self.connect_db()
        
        # Extract
        metadata_df = self.extract_metadata(artifacts)
        media_df = self.extract_media(artifacts)
        colors_df = self.extract_colors(artifacts)
        
        # Load
        self.load_to_db(metadata_df, 'artifactmetadata')
        self.load_to_db(media_df, 'artifactmedia')
        self.load_to_db(colors_df, 'artifactcolors')
        
        self.connection.close()
        print("ETL pipeline completed successfully")
```

### 3. SQL Analytics Queries

```python
# Example analytical queries

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
    
    "department_distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE 
                WHEN primaryimageurl IS NOT NULL THEN 'Has Image'
                ELSE 'No Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "color_diversity": """
        SELECT artifact_id, COUNT(DISTINCT color) as color_count
        FROM artifactcolors
        GROUP BY artifact_id
        ORDER BY color_count DESC
        LIMIT 20
    """
}

def execute_query(connection, query_name):
    """Execute predefined analytics query"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end data engineering and analytics application")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv("HARVARD_API_KEY", ""))
    
    # Data collection section
    st.header("📥 Data Collection")
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to collect", min_value=1, max_value=50, value=5)
    
    with col2:
        if st.button("🚀 Start Data Collection"):
            with st.spinner("Collecting artifacts..."):
                artifacts = collect_multiple_pages(api_key, num_pages)
                st.success(f"Collected {len(artifacts)} artifacts!")
                
                # Run ETL
                db_config = {
                    'host': os.getenv('DB_HOST'),
                    'port': int(os.getenv('DB_PORT', 3306)),
                    'user': os.getenv('DB_USER'),
                    'password': os.getenv('DB_PASSWORD'),
                    'database': os.getenv('DB_NAME')
                }
                
                etl = ArtifactETL(db_config)
                etl.run_pipeline(artifacts)
                st.success("ETL pipeline completed!")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    query_options = list(ANALYTICS_QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        df = execute_query(connection, selected_query)
        connection.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query.replace('_', ' ').title())
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting and Error Handling

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(api_key, page)
            return data
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load(api_key, db_config):
    """Load only new artifacts"""
    connection = mysql.connector.connect(**db_config)
    last_id = get_last_artifact_id(connection)
    connection.close()
    
    # Fetch artifacts with ID > last_id
    # Process only new records
```

## Troubleshooting

### API Key Issues
- Verify API key is valid at https://harvardartmuseums.org/collections/api
- Check `.env` file is loaded correctly
- Ensure no extra whitespace in API key

### Database Connection Errors
```python
# Test database connection
try:
    connection = mysql.connector.connect(**db_config)
    print("Database connected successfully")
    connection.close()
except mysql.connector.Error as err:
    print(f"Database error: {err}")
```

### Memory Issues with Large Datasets
```python
# Process in chunks
def chunked_etl(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        etl.run_pipeline(chunk)
```

### Duplicate Records
The ETL uses `INSERT IGNORE` to prevent duplicates. For updates, use:
```sql
INSERT INTO table (columns) VALUES (values)
ON DUPLICATE KEY UPDATE column = VALUES(column)
```
