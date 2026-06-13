---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - store Harvard API data in SQL database
  - create analytics dashboard with Streamlit
  - visualize museum artifact data with Plotly
  - set up data engineering pipeline for art collections
  - query and analyze Harvard museum artifacts
  - transform nested JSON art data to relational tables
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application using the Harvard Art Museums API. It demonstrates ETL pipelines, SQL analytics, and interactive Streamlit dashboards for museum artifact data.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
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

### API Key Setup

Get your Harvard Art Museums API key from [https://www.harvardartmuseums.org/collections/api](https://www.harvardartmuseums.org/collections/api)

Store credentials in `.env`:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Schema

Create three core tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    has_images BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
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
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data.get('records', []), data.get('info', {})
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Paginated collection
def collect_all_artifacts(max_pages=10):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        
        print(f"Fetched page {page}/{info.get('pages', 'unknown')}")
        
        if page >= info.get('pages', 0):
            break
    
    return all_artifacts
```

### 2. Data Transformation (ETL)

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform nested JSON to relational format"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media info
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'has_images': 1 if artifact.get('images') else 0
        }
        media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert DataFrames into SQL tables"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_tuples = [tuple(x) for x in metadata_df.to_numpy()]
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, period, technique, medium, dimensions, creditline, 
             accessionyear, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_tuples)
        
        # Insert media
        media_tuples = [tuple(x) for x in media_df.to_numpy()]
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, has_images)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_tuples)
        
        # Insert colors
        if not colors_df.empty:
            colors_tuples = [tuple(x) for x in colors_df.to_numpy()]
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_tuples)
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. SQL Analytics Queries

```python
def execute_analytics_query(query_name):
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'images_availability': """
            SELECT has_images, COUNT(*) as count
            FROM artifactmedia
            GROUP BY has_images
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 15
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as artifacts,
                   SUM(CASE WHEN m.has_images = 1 THEN 1 ELSE 0 END) as with_images
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifacts DESC
        """
    }
    
    conn = get_db_connection()
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Image Availability": "images_availability",
        "Top Colors": "top_colors",
        "Department Distribution": "department_distribution"
    }
    
    selected_query = st.sidebar.selectbox(
        "Select Analysis",
        list(query_options.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_analytics_query(query_options[selected_query])
            
            # Display results
            st.subheader(f"Results: {selected_query}")
            st.dataframe(df)
            
            # Visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py
```

## Complete ETL Pipeline Example

```python
from dotenv import load_dotenv
import os

load_dotenv()

def run_etl_pipeline():
    """Complete ETL pipeline execution"""
    
    print("Step 1: Extracting data from API...")
    raw_artifacts = collect_all_artifacts(max_pages=5)
    print(f"Extracted {len(raw_artifacts)} artifacts")
    
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
    print(f"Transformed {len(metadata_df)} records")
    
    print("Step 3: Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    print("ETL pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Add delay between API calls"""
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Error Handling in ETL

```python
def safe_transform(raw_artifacts):
    """Transform with error handling"""
    try:
        return transform_artifacts(raw_artifacts)
    except Exception as e:
        print(f"Transformation error: {e}")
        return pd.DataFrame(), pd.DataFrame(), pd.DataFrame()
```

### Incremental Loading

```python
def get_latest_artifact_id():
    """Get the highest artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    conn.close()
    return result[0] or 0

def fetch_new_artifacts_only():
    """Fetch only artifacts newer than what we have"""
    latest_id = get_latest_artifact_id()
    # Implement filtering logic based on latest_id
    pass
```

## Troubleshooting

**API Authentication Error:**
- Verify `HARVARD_API_KEY` is set correctly in `.env`
- Check API key validity at Harvard Art Museums developer portal

**Database Connection Issues:**
- Ensure database credentials are correct
- Verify database exists and tables are created
- Check firewall/network access to database host

**Memory Issues with Large Datasets:**
- Reduce `max_pages` parameter
- Process data in smaller batches
- Use chunked DataFrame operations

**Missing Data in Queries:**
- Some artifacts may have NULL values for certain fields
- Add `WHERE field IS NOT NULL` clauses
- Handle empty DataFrames in visualization code

**Streamlit Performance:**
- Cache query results with `@st.cache_data`
- Limit result set sizes with SQL `LIMIT` clauses
- Use connection pooling for database access
