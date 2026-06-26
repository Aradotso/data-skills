---
name: harvard-art-museum-etl-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - show me how to use the Harvard artifacts collection app
  - create a data engineering pipeline for museum data
  - build a Streamlit analytics dashboard for Harvard art data
  - extract and transform Harvard museum artifact data
  - set up SQL analytics for art museum collections
  - how to visualize Harvard Art Museums API data
  - create an end-to-end data pipeline with museum artifacts
---

# Harvard Art Museum ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection app provides a complete data engineering workflow:

- **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Convert nested JSON into normalized relational tables (metadata, media, colors)
- **Load**: Batch insert data into MySQL/TiDB Cloud with proper foreign key relationships
- **Analyze**: Execute 20+ predefined SQL queries for insights
- **Visualize**: Display results in interactive Plotly charts via Streamlit

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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
export DB_NAME="harvard_artifacts"
```

## Configuration

### API Key Setup

Obtain your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store it securely:

```python
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

Configure your MySQL/TiDB connection:

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

The project uses three normalized tables:

```sql
-- Artifact Metadata
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
    url TEXT,
    primaryimageurl TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    renditionnumber VARCHAR(50),
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

# The app will be available at http://localhost:8501
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

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
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

# Fetch multiple pages
def extract_all_artifacts(max_pages=10):
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        
        # Check if more pages exist
        if page >= info['pages']:
            break
    
    return all_artifacts
```

### Transform: Normalize JSON Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into normalized dataframes"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'url': artifact.get('url'),
            'primaryimageurl': artifact.get('primaryimageurl')
        }
        metadata_records.append(metadata)
        
        # Extract media
        if 'images' in artifact and artifact['images']:
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'baseimageurl': img.get('baseimageurl'),
                    'renditionnumber': img.get('renditionnumber')
                }
                media_records.append(media)
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert into Database

```python
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Batch load data into MySQL database"""
    
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, division, dated, url, primaryimageurl)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia 
                (artifact_id, media_type, baseimageurl, renditionnumber)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytics Queries

### Sample SQL Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
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
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_files
        FROM artifactmedia m
    """,
    
    "Artifacts by Culture and Century": """
        SELECT culture, century, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND century IS NOT NULL
        GROUP BY culture, century
        HAVING count > 5
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query, db_config):
    """Execute analytical query and return results as DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[query_name]
            results_df = execute_query(query, db_config)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(max_pages=5):
    """Complete ETL pipeline execution"""
    
    # Extract
    st.info("Extracting data from Harvard Art Museums API...")
    artifacts = extract_all_artifacts(max_pages=max_pages)
    
    # Transform
    st.info(f"Transforming {len(artifacts)} artifacts...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Load
    st.info("Loading data to database...")
    load_to_database(metadata_df, media_df, colors_df, db_config)
    
    st.success(f"ETL complete! Processed {len(artifacts)} artifacts")
    
    return {
        'artifacts': len(artifacts),
        'media_files': len(media_df),
        'color_records': len(colors_df)
    }
```

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(page, size=100, delay=0.5):
    """Fetch data with rate limiting to avoid API throttling"""
    records, info = fetch_artifacts(page=page, size=size)
    time.sleep(delay)  # Wait between requests
    return records, info
```

## Troubleshooting

### API Connection Issues

```python
# Handle connection errors gracefully
try:
    response = requests.get(url, params=params, timeout=10)
    response.raise_for_status()
except requests.exceptions.Timeout:
    st.error("API request timed out. Please try again.")
except requests.exceptions.RequestException as e:
    st.error(f"API error: {e}")
```

### Database Connection Failures

```python
# Test database connection
def test_db_connection(db_config):
    try:
        conn = mysql.connector.connect(**db_config)
        if conn.is_connected():
            st.success("Database connected successfully")
            conn.close()
            return True
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return False
```

### Missing Data Handling

```python
# Handle missing fields in API response
def safe_get(dictionary, key, default='', max_length=None):
    """Safely extract value from dictionary with defaults"""
    value = dictionary.get(key, default)
    if max_length and value:
        value = str(value)[:max_length]
    return value
```

## Advanced Usage

### Incremental Data Loading

```python
def get_latest_artifact_id(db_config):
    """Get the latest artifact ID from database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    conn.close()
    return max_id

def incremental_etl(db_config):
    """Load only new artifacts since last run"""
    latest_id = get_latest_artifact_id(db_config)
    
    # Fetch artifacts with ID > latest_id
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'q': f'id:>{latest_id}',
        'size': 100
    }
    # Continue with ETL...
```

This skill provides everything needed to build production-ready ETL pipelines for museum data using modern data engineering practices.
