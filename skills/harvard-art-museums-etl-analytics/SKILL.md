---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts
  - create a data analytics dashboard with Harvard Art Museums data
  - set up artifact collection data engineering workflow
  - integrate Harvard Art Museums API with SQL database
  - build Streamlit analytics app for art collection data
  - create museum artifact visualization dashboard
  - implement ETL for Harvard Art Museums API
  - analyze art collection data with SQL queries
---

# Harvard Art Museums ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with the Harvard-Artifacts-Collection-Data-Engineering-Analytics-App, an end-to-end data engineering solution that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases, and provides interactive analytics through Streamlit dashboards.

## What This Project Does

The application implements a complete **API → ETL → SQL → Analytics → Visualization** pipeline:

- **Extracts** artifact metadata, media details, and color information from Harvard Art Museums API
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud with proper schema design
- **Analyzes** using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages (typical requirements.txt)
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

Register at: https://www.harvardartmuseums.org/collections/api

## Database Schema

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    base_url VARCHAR(500),
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color_hex VARCHAR(20),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(max_pages=10):
    """Collect artifacts with pagination"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        
        # Check if more pages exist
        if not data.get('info', {}).get('next'):
            break
            
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> pd.DataFrame:
    """Transform raw API response into structured metadata"""
    metadata = []
    
    for item in raw_data:
        metadata.append({
            'id': item.get('id'),
            'title': item.get('title', 'Unknown'),
            'culture': item.get('culture', 'Unknown'),
            'century': item.get('century', 'Unknown'),
            'classification': item.get('classification', 'Unknown'),
            'department': item.get('department', 'Unknown'),
            'dated': item.get('dated', 'Unknown'),
            'accessionyear': item.get('accessionyear'),
            'technique': item.get('technique', 'Unknown'),
            'medium': item.get('medium', 'Unknown'),
            'dimensions': item.get('dimensions', 'Unknown'),
            'url': item.get('url', '')
        })
    
    return pd.DataFrame(metadata)

def transform_media(raw_data: List[Dict]) -> pd.DataFrame:
    """Extract media information"""
    media = []
    
    for item in raw_data:
        artifact_id = item.get('id')
        for image in item.get('images', []):
            media.append({
                'artifact_id': artifact_id,
                'media_id': image.get('imageid'),
                'base_url': image.get('baseimageurl', ''),
                'format': image.get('format', 'Unknown')
            })
    
    return pd.DataFrame(media)

def transform_colors(raw_data: List[Dict]) -> pd.DataFrame:
    """Extract color information"""
    colors = []
    
    for item in raw_data:
        artifact_id = item.get('id')
        for color in item.get('colors', []):
            colors.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0)
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df: pd.DataFrame):
    """Batch insert artifact metadata"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT IGNORE INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     dated, accessionyear, technique, medium, dimensions, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    cursor.close()
    conn.close()

def load_media(df: pd.DataFrame):
    """Batch insert media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, media_id, base_url, format)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    cursor.close()
    conn.close()

def load_colors(df: pd.DataFrame):
    """Batch insert color data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
    VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Analytics Queries

```python
# Sample analytical queries included in the application

ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as image_count
        FROM artifactmedia
        GROUP BY format
        ORDER BY image_count DESC
    """,
    
    "Top Colors Across All Artifacts": """
        SELECT color_hex, SUM(color_percent) as total_usage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY total_usage DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.id, m.title, COUNT(media.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.id, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics")
    st.write("End-to-end ETL and Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[selected_query]
            df = execute_query(query)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(15),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Controls
    st.sidebar.header("ETL Pipeline")
    if st.sidebar.button("Run ETL"):
        with st.spinner("Collecting data from API..."):
            raw_data = collect_all_artifacts(max_pages=5)
            
            st.info(f"Collected {len(raw_data)} artifacts")
            
            # Transform
            metadata_df = transform_artifacts(raw_data)
            media_df = transform_media(raw_data)
            colors_df = transform_colors(raw_data)
            
            # Load
            load_metadata(metadata_df)
            load_media(media_df)
            load_colors(colors_df)
            
            st.success("ETL Pipeline Completed!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch with rate limiting"""
    data = fetch_artifacts(page=page)
    time.sleep(delay)  # Avoid API rate limits
    return data
```

### Error Handling for ETL

```python
def safe_etl_pipeline(max_pages=10):
    """ETL with error handling"""
    try:
        raw_data = collect_all_artifacts(max_pages)
        
        metadata_df = transform_artifacts(raw_data)
        media_df = transform_media(raw_data)
        colors_df = transform_colors(raw_data)
        
        load_metadata(metadata_df)
        load_media(media_df)
        load_colors(colors_df)
        
        return True, f"Successfully processed {len(raw_data)} artifacts"
    
    except requests.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get highest artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_load():
    """Load only new artifacts"""
    max_id = get_max_artifact_id()
    raw_data = collect_all_artifacts(max_pages=5)
    
    # Filter for new artifacts only
    new_artifacts = [a for a in raw_data if a.get('id', 0) > max_id]
    
    if new_artifacts:
        metadata_df = transform_artifacts(new_artifacts)
        load_metadata(metadata_df)
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is set
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY environment variable not set")
```

### Database Connection Problems

```python
# Test database connection
try:
    conn = get_db_connection()
    conn.close()
    print("Database connection successful")
except mysql.connector.Error as e:
    print(f"Database connection failed: {e}")
```

### Empty Query Results

```python
# Check if tables have data
def check_data_exists():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    count = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    
    if count == 0:
        print("No data in database. Run ETL pipeline first.")
    return count > 0
```

### Handling API Rate Limits

If you encounter `429 Too Many Requests` errors:

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

This skill covers the complete workflow for building ETL pipelines with museum artifact data, from API extraction through SQL analytics to interactive visualization.
