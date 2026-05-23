---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL storage, and Streamlit visualization
triggers:
  - build a data pipeline for Harvard Art Museums API
  - create ETL process for museum artifact data
  - set up Harvard artifacts analytics dashboard
  - implement museum data engineering pipeline
  - query and visualize Harvard Art Museums collection
  - design SQL schema for artifact metadata
  - build Streamlit app for museum data analytics
  - process Harvard API artifact responses
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming nested JSON into relational tables, loading into SQL databases (MySQL/TiDB Cloud), and visualizing insights through a Streamlit dashboard. It includes pagination handling, batch inserts, foreign key relationships, and 20+ predefined analytical queries.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages (typical requirements.txt content):
# streamlit
# pandas
# requests
# mysql-connector-python
# plotly
# python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema

The project uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata (primary table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    verificationlevel INT,
    primaryimageurl TEXT,
    description TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
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
# Start the Streamlit app
streamlit run app.py

# Access the dashboard at http://localhost:8501
```

## Key Components

### 1. API Integration

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Example: Fetch multiple pages
def collect_all_artifacts(max_pages=10):
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(records)
        
        print(f"Fetched page {page}/{info['pages']}")
        
        if page >= info['pages']:
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def extract_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract artifact metadata into structured DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'verificationlevel': artifact.get('verificationlevel'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'description': artifact.get('description')
        })
    
    return pd.DataFrame(metadata)

def extract_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract nested media data"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_records.append({
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            })
    
    return pd.DataFrame(media_records)

def extract_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract color data from artifacts"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)

def load_to_database(df: pd.DataFrame, table_name: str, conn):
    """Batch insert DataFrame into SQL table"""
    cursor = conn.cursor()
    
    # Get column names
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    
    sql = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Batch insert
    data = [tuple(row) for row in df.values]
    cursor.executemany(sql, data)
    conn.commit()
    
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
```

### 3. Database Connection

```python
def get_database_connection():
    """Create MySQL/TiDB connection"""
    load_dotenv()
    
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    return connection

# Usage
conn = get_database_connection()
```

### 4. Analytical Queries

```python
# Sample analytical queries
QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
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
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Department Analysis": """
        SELECT department, 
               COUNT(*) as total_artifacts,
               COUNT(primaryimageurl) as with_images
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "High Resolution Images": """
        SELECT a.title, a.culture, m.width, m.height
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        WHERE m.width > 2000 AND m.height > 2000
        ORDER BY m.width DESC
        LIMIT 20
    """
}

def execute_query(query: str, conn):
    """Execute SQL query and return results as DataFrame"""
    cursor = conn.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    return pd.DataFrame(results, columns=columns)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(QUERIES.keys())
    )
    
    # Database connection
    try:
        conn = get_database_connection()
        
        # Execute selected query
        if st.sidebar.button("Run Query"):
            with st.spinner("Executing query..."):
                df = execute_query(QUERIES[query_name], conn)
                
                # Display results
                st.subheader(f"Results: {query_name}")
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df.head(20), 
                        x=df.columns[0], 
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
                    
    except Exception as e:
        st.error(f"Error: {str(e)}")
    finally:
        if 'conn' in locals():
            conn.close()

if __name__ == "__main__":
    main()
```

## Complete ETL Workflow

```python
def run_complete_pipeline(max_pages=5):
    """Execute full ETL pipeline"""
    
    # Extract
    print("Extracting data from API...")
    artifacts = collect_all_artifacts(max_pages=max_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Load
    print("Loading to database...")
    conn = get_database_connection()
    
    load_to_database(metadata_df, 'artifactmetadata', conn)
    load_to_database(media_df, 'artifactmedia', conn)
    load_to_database(colors_df, 'artifactcolors', conn)
    
    conn.close()
    print("ETL pipeline completed successfully!")

# Run the pipeline
run_complete_pipeline(max_pages=10)
```

## Common Patterns

### Rate Limiting

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch artifacts with rate limiting"""
    records, info = fetch_artifacts(page=page)
    time.sleep(delay)  # Respect API rate limits
    return records, info
```

### Error Handling

```python
def safe_etl_pipeline():
    """ETL with error handling and logging"""
    try:
        artifacts = collect_all_artifacts(max_pages=10)
        
        if not artifacts:
            raise ValueError("No artifacts collected")
        
        conn = get_database_connection()
        
        # Process in batches
        batch_size = 100
        for i in range(0, len(artifacts), batch_size):
            batch = artifacts[i:i+batch_size]
            metadata_df = extract_metadata(batch)
            load_to_database(metadata_df, 'artifactmetadata', conn)
            print(f"Processed batch {i//batch_size + 1}")
        
        conn.close()
        
    except Exception as e:
        print(f"Pipeline failed: {str(e)}")
        raise
```

## Troubleshooting

**API Key Issues**: Ensure `HARVARD_API_KEY` is set in `.env` and valid. Get a key from https://www.harvardartmuseums.org/collections/api

**Database Connection Errors**: Verify `DB_HOST`, `DB_USER`, `DB_PASSWORD` are correct. Check firewall rules for TiDB Cloud connections.

**Missing Data**: Some API fields may be null. Use `.get()` method with defaults when extracting fields.

**Memory Issues**: Process artifacts in batches rather than loading all at once. Use pagination effectively.

**Foreign Key Violations**: Ensure `artifactmetadata` is loaded before `artifactmedia` and `artifactcolors` tables.
