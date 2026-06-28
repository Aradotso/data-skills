---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build etl pipeline for harvard art museums api
  - create analytics dashboard for museum artifact data
  - extract and transform harvard art museums data
  - set up sql database for artifact collections
  - visualize museum artifact data with streamlit
  - implement harvard museums api data pipeline
  - analyze art museum collections with sql
  - create interactive art data visualization dashboard
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What This Project Does

The Harvard Art Museums ETL Analytics application:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational database tables
- Loads structured data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and color data
- Visualizes results through interactive Streamlit dashboards with Plotly charts

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Key Dependencies:**
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

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application uses three normalized tables:

**artifactmetadata:**
- `id` (PRIMARY KEY)
- `title`
- `culture`
- `century`
- `classification`
- `department`
- `technique`
- `medium`
- `dated`

**artifactmedia:**
- `id` (AUTO_INCREMENT PRIMARY KEY)
- `artifact_id` (FOREIGN KEY → artifactmetadata.id)
- `media_type`
- `baseimageurl`
- `primaryimageurl`

**artifactcolors:**
- `id` (AUTO_INCREMENT PRIMARY KEY)
- `artifact_id` (FOREIGN KEY → artifactmetadata.id)
- `color`
- `spectrum`
- `hue`
- `percent`

## Core ETL Pipeline

### 1. Extract from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(num_pages=5):
    """
    Collect multiple pages of artifact data
    """
    api_key = os.getenv("HARVARD_API_KEY")
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get("records", []))
        print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
    
    return all_artifacts
```

### 2. Transform to Relational Format

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get("id")
        
        # Extract metadata
        metadata_list.append({
            "id": artifact_id,
            "title": artifact.get("title", ""),
            "culture": artifact.get("culture", ""),
            "century": artifact.get("century", ""),
            "classification": artifact.get("classification", ""),
            "department": artifact.get("department", ""),
            "technique": artifact.get("technique", ""),
            "medium": artifact.get("medium", ""),
            "dated": artifact.get("dated", "")
        })
        
        # Extract media information
        images = artifact.get("images", [])
        if images:
            for img in images:
                media_list.append({
                    "artifact_id": artifact_id,
                    "media_type": "image",
                    "baseimageurl": img.get("baseimageurl", ""),
                    "primaryimageurl": artifact.get("primaryimageurl", "")
                })
        
        # Extract color information
        colors = artifact.get("colors", [])
        for color in colors:
            colors_list.append({
                "artifact_id": artifact_id,
                "color": color.get("color", ""),
                "spectrum": color.get("spectrum", ""),
                "hue": color.get("hue", ""),
                "percent": color.get("percent", 0)
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Load into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL/TiDB connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv("DB_HOST"),
            port=int(os.getenv("DB_PORT", 3306)),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
            database=os.getenv("DB_NAME")
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """
    Create normalized database tables with foreign keys
    """
    cursor = connection.cursor()
    
    # Create metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            technique TEXT,
            medium TEXT,
            dated VARCHAR(255)
        )
    """)
    
    # Create media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(50),
            baseimageurl TEXT,
            primaryimageurl TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Create colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()

def batch_insert_data(connection, metadata_df, media_df, colors_df):
    """
    Batch insert DataFrames into database tables
    """
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_records = metadata_df.to_records(index=False).tolist()
    cursor.executemany("""
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, technique, medium, dated)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """, metadata_records)
    
    # Insert media
    if not media_df.empty:
        media_records = media_df.to_records(index=False).tolist()
        cursor.executemany("""
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, primaryimageurl)
            VALUES (%s, %s, %s, %s)
        """, media_records)
    
    # Insert colors
    if not colors_df.empty:
        colors_records = colors_df.to_records(index=False).tolist()
        cursor.executemany("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, colors_records)
    
    connection.commit()
    cursor.close()
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    st.markdown("### ETL Pipeline & SQL Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose Function",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("Data Collection from Harvard API")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 10, 5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_artifacts = collect_all_artifacts(num_pages)
            st.success(f"Fetched {len(raw_artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = create_database_connection()
            create_tables(conn)
            batch_insert_data(conn, metadata_df, media_df, colors_df)
            conn.close()
            st.success("Data loaded to SQL database")

if __name__ == "__main__":
    main()
```

### SQL Analytics Queries

```python
def show_sql_analytics():
    st.header("SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture != '' 
            GROUP BY culture 
            ORDER BY count DESC 
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE century != '' 
            GROUP BY century 
            ORDER BY count DESC
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY usage_count DESC 
            LIMIT 10
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE department != '' 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Media Availability": """
            SELECT 
                COUNT(DISTINCT artifact_id) as artifacts_with_media,
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts
            FROM artifactmedia
        """
    }
    
    query_name = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = create_database_connection()
        result_df = pd.read_sql_query(queries[query_name], conn)
        conn.close()
        
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(result_df, x=result_df.columns[0], y=result_df.columns[1])
            st.plotly_chart(fig)
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """
    Fetch data with rate limiting to respect API limits
    """
    data = fetch_artifacts(api_key, page)
    time.sleep(delay)  # Wait between requests
    return data
```

### Error Handling in ETL

```python
def safe_etl_pipeline(num_pages=5):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        raw_data = collect_all_artifacts(num_pages)
        
        if not raw_data:
            raise ValueError("No data collected from API")
        
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        
        conn = create_database_connection()
        if not conn:
            raise ConnectionError("Failed to connect to database")
        
        create_tables(conn)
        batch_insert_data(conn, metadata_df, media_df, colors_df)
        conn.close()
        
        return True, "ETL pipeline completed successfully"
    
    except Exception as e:
        return False, f"ETL pipeline failed: {str(e)}"
```

### Dynamic Query Builder

```python
def build_filter_query(culture=None, century=None, department=None):
    """
    Build dynamic SQL queries based on user filters
    """
    query = "SELECT * FROM artifactmetadata WHERE 1=1"
    params = []
    
    if culture:
        query += " AND culture = %s"
        params.append(culture)
    
    if century:
        query += " AND century = %s"
        params.append(century)
    
    if department:
        query += " AND department = %s"
        params.append(department)
    
    return query, params
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in `.env` file
- Verify API key is active at https://www.harvardartmuseums.org/collections/api
- Check for rate limiting (max 2500 requests/day for free tier)

### Database Connection Errors
- Verify database credentials in `.env`
- For TiDB Cloud, ensure SSL/TLS settings are correct
- Check firewall rules allow connection to database port

### Empty Results
- API returns different fields per artifact; handle missing keys with `.get()`
- Filter for `hasimage=1` to ensure media data exists
- Some artifacts have empty culture/century fields

### Memory Issues with Large Datasets
- Use pagination and process in batches
- Don't load all data into memory at once
- Consider chunked DataFrame operations for large inserts

### Streamlit Performance
- Cache database connections with `@st.cache_resource`
- Cache query results with `@st.cache_data`
- Limit initial data loads and use pagination in UI
