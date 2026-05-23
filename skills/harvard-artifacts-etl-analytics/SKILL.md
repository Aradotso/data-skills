---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering workflow with museum artifacts
  - set up SQL analytics for Harvard API data
  - visualize Harvard Art Museums collection data
  - implement artifact data pipeline with Streamlit
  - extract and analyze museum metadata from Harvard API
  - build a museum data warehouse with Python
  - create interactive dashboards for art collection analytics
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-ready ETL patterns, relational database design, SQL analytics, and interactive visualization using Streamlit.

**Architecture Flow:** API → ETL → SQL Database → Analytics → Streamlit Dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the application
streamlit run app.py
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

Create a `.env` file or set environment variables:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Harvard API Key

Obtain an API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):
1. Register for API access
2. Store key in environment variable `HARVARD_API_KEY`
3. Respect rate limits (typically 2500 requests/day)

### Database Setup

The application supports MySQL or TiDB Cloud. Create the database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;

USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    technique VARCHAR(500),
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    imageid INT,
    baseimageurl VARCHAR(500),
    alttext TEXT,
    width INT,
    height INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

def collect_multiple_pages(api_key, num_pages=5):
    """
    Collect artifacts across multiple pages
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            print(f"Collected page {page}: {len(artifacts)} artifacts")
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw artifact data into structured metadata
    """
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
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'technique': artifact.get('technique'),
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """
    Extract and transform media/image data
    """
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_records.append({
                'artifact_id': artifact_id,
                'imageid': image.get('imageid'),
                'baseimageurl': image.get('baseimageurl'),
                'alttext': image.get('alttext'),
                'width': image.get('width'),
                'height': image.get('height'),
                'format': image.get('format')
            })
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """
    Extract color palette data
    """
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
```

### 3. Database Operations

```python
import mysql.connector
from mysql.connector import Error

def get_database_connection():
    """
    Create database connection using environment variables
    """
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

def batch_insert_metadata(df, connection):
    """
    Batch insert artifact metadata with upsert logic
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     dated, description, technique, accessionyear, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), totalpageviews=VALUES(totalpageviews)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(df, connection):
    """
    Batch insert media records
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, imageid, baseimageurl, alttext, width, height, format)
    VALUES (%s, %s, %s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries for the dashboard

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
            COUNT(am.id) as total_media_items,
            AVG(am.width) as avg_width,
            AVG(am.height) as avg_height
        FROM artifactmedia am
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Most Viewed Artifacts": """
        SELECT title, culture, department, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_analytics_query(query_name, connection):
    """
    Execute analytical query and return results as DataFrame
    """
    cursor = connection.cursor()
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        return None
    
    try:
        cursor.execute(query)
        columns = [desc[0] for desc in cursor.description]
        results = cursor.fetchall()
        df = pd.DataFrame(results, columns=columns)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        cursor.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                 value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("Data Collection")
        num_pages = st.slider("Number of pages to collect", 1, 20, 5)
        
        if st.button("🔄 Collect Data"):
            with st.spinner("Collecting artifacts..."):
                artifacts = collect_multiple_pages(api_key, num_pages)
                st.success(f"Collected {len(artifacts)} artifacts")
    
    # Main analytics section
    st.header("📊 SQL Analytics")
    
    connection = get_database_connection()
    
    if connection:
        query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Run Query"):
            df = execute_analytics_query(query_name, connection)
            
            if df is not None and not df.empty:
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=query_name)
                    st.plotly_chart(fig, use_container_width=True)
        
        connection.close()
    else:
        st.error("Database connection failed")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_full_etl_pipeline(api_key, num_pages=5):
    """
    Execute complete ETL pipeline from API to database
    """
    # Extract
    print("Step 1: Extracting data from API...")
    artifacts = collect_multiple_pages(api_key, num_pages)
    
    # Transform
    print("Step 2: Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Step 3: Loading data to database...")
    connection = get_database_connection()
    
    if connection:
        batch_insert_metadata(metadata_df, connection)
        batch_insert_media(media_df, connection)
        batch_insert_colors(colors_df, connection)
        connection.close()
        print("ETL pipeline completed successfully!")
    else:
        print("Failed to connect to database")
```

### Error Handling and Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """
    Fetch data with retry logic and rate limiting
    """
    for attempt in range(max_retries):
        try:
            time.sleep(0.5)  # Rate limiting
            data = fetch_artifacts(api_key, page)
            return data
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Retry {attempt + 1} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

## Troubleshooting

### API Connection Issues

- **401 Unauthorized**: Check `HARVARD_API_KEY` environment variable
- **429 Rate Limited**: Implement delays between requests (0.5-1 second)
- **Empty results**: Verify API parameters and check museum API status

### Database Issues

- **Connection refused**: Verify `DB_HOST`, `DB_USER`, `DB_PASSWORD` values
- **Duplicate key errors**: Use `ON DUPLICATE KEY UPDATE` for upserts
- **Foreign key violations**: Ensure metadata is inserted before media/colors

### Streamlit Performance

- Use `@st.cache_data` for expensive operations:
  ```python
  @st.cache_data
  def load_artifacts_cached(api_key, num_pages):
      return collect_multiple_pages(api_key, num_pages)
  ```

- Limit result set sizes for large queries
- Use pagination for displaying large datasets

### Data Quality

- Handle `None` values in transformations
- Validate data types before database insertion
- Log transformation errors for debugging

```python
def safe_transform(artifact, field, default=None):
    """
    Safely extract field with default fallback
    """
    try:
        return artifact.get(field, default)
    except Exception as e:
        print(f"Transform error for field {field}: {e}")
        return default
```
