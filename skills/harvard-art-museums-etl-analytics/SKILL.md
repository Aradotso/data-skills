---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - set up Harvard API data collection
  - implement artifact data engineering workflow
  - query and visualize museum collection data
  - design SQL schema for art museum artifacts
  - build Streamlit app for Harvard Art Museums
  - process and analyze cultural heritage data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with the Harvard Art Museums data engineering and analytics application. The project demonstrates a complete ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases, and provides interactive analytics through Streamlit.

## What It Does

- **API Integration**: Collects artifact metadata, media, and color data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational database tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design and relationships
- **Analytics**: Executes predefined SQL queries for insights on artifacts, cultures, centuries, and media
- **Visualization**: Interactive Plotly charts and Streamlit dashboards for data exploration

## Architecture Flow

```
Harvard API → Extract (Python/Requests) → Transform (Pandas) → Load (MySQL) → Query (SQL) → Visualize (Streamlit/Plotly)
```

## Installation

### Prerequisites

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```python
# requirements.txt
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.17.0
python-dotenv>=1.0.0
```

### Environment Configuration

Create a `.env` file with your credentials:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema Design

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    period VARCHAR(255),
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(50),
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## Core ETL Pipeline

### 1. Extract Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Extract artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages
def collect_all_artifacts(max_pages=10):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        
        # Check if more pages exist
        if page >= data.get('info', {}).get('pages', 0):
            break
    
    return all_artifacts
```

### 2. Transform Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into structured dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'technique': artifact.get('technique', ''),
            'period': artifact.get('period', ''),
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'width': image.get('width'),
                'height': image.get('height')
            }
            media_records.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_data)
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

### 3. Load Data into SQL

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

def load_metadata(df_metadata):
    """Load metadata into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         technique, period, accessionyear, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(x) for x in df_metadata.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def load_media(df_media):
    """Load media data into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, format, width, height)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df_media.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def load_colors(df_colors):
    """Load color data into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df_colors.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytics Queries

### Sample SQL Queries

```python
# Query definitions for analytics
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Most Popular Artifacts": """
        SELECT title, culture, century, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Media Format Analysis": """
        SELECT format, COUNT(*) as count, AVG(width) as avg_width, AVG(height) as avg_height
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        conn.close()
```

## Streamlit Application

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics Dashboard", "Data Explorer"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics Dashboard":
        show_analytics()
    else:
        show_data_explorer()

def show_data_collection():
    """ETL pipeline interface"""
    st.header("📥 Data Collection & ETL")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
    
    if st.button("Start ETL Process"):
        with st.spinner("Fetching data from API..."):
            artifacts = collect_all_artifacts(max_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            transformed = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_metadata(transformed['metadata'])
            load_media(transformed['media'])
            load_colors(transformed['colors'])
            st.success("Data loaded to database")

def show_analytics():
    """Analytics dashboard with visualizations"""
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Analysis"):
        query = ANALYTICS_QUERIES[query_name]
        df = execute_query(query)
        
        if not df.empty:
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualizations
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name,
                    color=df.columns[1],
                    color_continuous_scale='viridis'
                )
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No data returned")

def show_data_explorer():
    """Custom SQL query interface"""
    st.header("🔍 Data Explorer")
    
    custom_query = st.text_area(
        "Enter SQL Query",
        height=150,
        placeholder="SELECT * FROM artifactmetadata LIMIT 10"
    )
    
    if st.button("Execute Query"):
        df = execute_query(custom_query)
        if not df.empty:
            st.dataframe(df, use_container_width=True)
            st.info(f"Returned {len(df)} rows")

if __name__ == "__main__":
    main()
```

### Run the Application

```bash
streamlit run app.py
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch data with rate limiting"""
    time.sleep(delay)
    return fetch_artifacts(page=page)

# Usage
for page in range(1, 11):
    data = fetch_with_rate_limit(page)
    # Process data
```

### Batch Processing

```python
def batch_process(artifacts, batch_size=100):
    """Process artifacts in batches"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        transformed = transform_artifacts(batch)
        load_metadata(transformed['metadata'])
        load_media(transformed['media'])
        load_colors(transformed['colors'])
        print(f"Processed batch {i//batch_size + 1}")
```

### Error Handling

```python
def safe_fetch(page, max_retries=3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            print(f"Retry {attempt + 1}/{max_retries}")
            time.sleep(2 ** attempt)
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Errors

```python
# Test database connection
def test_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        print("Database connection successful")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Empty API Responses

```python
# Check API response validity
def validate_response(data):
    if not data.get('records'):
        print("No records in response")
        return False
    if data.get('info', {}).get('totalrecords', 0) == 0:
        print("No total records available")
        return False
    return True
```

### Data Type Mismatches

```python
# Handle None values before loading
def clean_dataframe(df):
    """Clean dataframe before database insert"""
    # Replace None with appropriate defaults
    df = df.fillna({
        'title': '',
        'culture': '',
        'accessionyear': 0,
        'totalpageviews': 0
    })
    return df
```

## Advanced Usage

### Custom Filters

```python
def fetch_filtered_artifacts(classification=None, century=None):
    """Fetch artifacts with specific filters"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100
    }
    
    if classification:
        params['classification'] = classification
    if century:
        params['century'] = century
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    return response.json()
```

### Export Analytics Results

```python
def export_query_results(query_name, format='csv'):
    """Export query results to file"""
    df = execute_query(ANALYTICS_QUERIES[query_name])
    
    if format == 'csv':
        df.to_csv(f"{query_name.replace(' ', '_')}.csv", index=False)
    elif format == 'excel':
        df.to_excel(f"{query_name.replace(' ', '_')}.xlsx", index=False)
    
    return df
```

This skill provides comprehensive guidance for building data engineering pipelines with the Harvard Art Museums API, from ETL to analytics visualization.
