---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard artifacts
  - set up analytics dashboard for museum collection data
  - implement SQL analytics with Harvard API
  - build Streamlit app for art museum data visualization
  - extract and transform Harvard Art Museums API data
  - create data pipeline for cultural artifact analysis
  - set up museum collection data warehouse
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data engineering solution that demonstrates real-world ETL pipelines. It extracts artifact data from the Harvard Art Museums API, transforms nested JSON into normalized relational tables, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through a Streamlit dashboard with 20+ predefined SQL queries and Plotly visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages include:
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

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Create database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def create_tables(cursor):
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            dated VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            century VARCHAR(255),
            url TEXT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            mediatype VARCHAR(100),
            imageurl TEXT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percentage FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
```

## API Integration

### Fetching Data from Harvard Art Museums API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages
def fetch_all_artifacts(total_pages=10):
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract Phase

```python
def extract_artifact_data(artifacts):
    """
    Extract relevant fields from API response
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'century': artifact.get('century', '')[:255],
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        images = artifact.get('images', [])
        for img in images:
            media = {
                'objectid': artifact.get('objectid'),
                'mediatype': 'image',
                'imageurl': img.get('baseimageurl', '')
            }
            media_list.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'objectid': artifact.get('objectid'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percentage': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return metadata_list, media_list, colors_list
```

### Transform Phase

```python
import pandas as pd

def transform_data(metadata_list, media_list, colors_list):
    """
    Transform lists into pandas DataFrames with data cleaning
    """
    # Create DataFrames
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    # Data cleaning
    df_metadata = df_metadata.drop_duplicates(subset=['objectid'])
    df_metadata = df_metadata.fillna('')
    
    # Convert percentage to float
    if not df_colors.empty:
        df_colors['percentage'] = pd.to_numeric(df_colors['percentage'], errors='coerce').fillna(0.0)
    
    return df_metadata, df_media, df_colors
```

### Load Phase

```python
def load_data(df_metadata, df_media, df_colors):
    """
    Batch insert data into SQL database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (objectid, title, culture, period, dated, classification, 
             department, division, century, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        if not df_media.empty:
            media_query = """
                INSERT INTO artifactmedia (objectid, mediatype, imageurl)
                VALUES (%s, %s, %s)
            """
            cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        if not df_colors.empty:
            colors_query = """
                INSERT INTO artifactcolors (objectid, color, spectrum, percentage)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(colors_query, df_colors.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Exception as e:
        conn.rollback()
        print(f"Error loading data: {e}")
    finally:
        cursor.close()
        conn.close()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department != ''
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors by Frequency": """
        SELECT color, COUNT(*) as color_count, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color != ''
        GROUP BY color
        ORDER BY color_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.objectid IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as artifact_count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY media_status
    """,
    
    "Artifacts by Classification and Culture": """
        SELECT classification, culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification != '' AND culture != ''
        GROUP BY classification, culture
        HAVING count > 5
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_analytics_query(query_name):
    """
    Execute predefined analytics query and return results as DataFrame
    """
    conn = get_db_connection()
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

## Streamlit Dashboard

### Complete Application Structure

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Data Collection":
        show_data_collection()
    elif menu == "SQL Analytics":
        show_sql_analytics()
    elif menu == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection from API")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=100, value=5)
    
    if st.button("Start ETL Process"):
        with st.spinner("Fetching data from Harvard API..."):
            # Extract
            artifacts = fetch_all_artifacts(total_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            # Transform
            metadata_list, media_list, colors_list = extract_artifact_data(artifacts)
            df_metadata, df_media, df_colors = transform_data(metadata_list, media_list, colors_list)
            st.success("Data transformed successfully")
            
            # Load
            load_data(df_metadata, df_media, df_colors)
            st.success("Data loaded to database!")
            
            # Show sample data
            st.subheader("Sample Metadata")
            st.dataframe(df_metadata.head(10))

def show_sql_analytics():
    st.header("📊 SQL Analytics Queries")
    
    query_name = st.selectbox(
        "Select Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df_result = execute_analytics_query(query_name)
            
            st.subheader("Query Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    st.header("📈 Interactive Visualizations")
    
    viz_type = st.selectbox(
        "Select Visualization",
        ["Culture Distribution", "Color Analysis", "Department Overview"]
    )
    
    if viz_type == "Culture Distribution":
        df = execute_analytics_query("Artifacts by Culture")
        fig = px.pie(df, names='culture', values='artifact_count', 
                     title="Artifacts by Culture")
        st.plotly_chart(fig, use_container_width=True)
    
    elif viz_type == "Color Analysis":
        df = execute_analytics_query("Top Colors by Frequency")
        fig = px.bar(df, x='color', y='color_count', 
                     color='avg_percentage',
                     title="Color Distribution in Artifacts")
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, delay=1.0):
    """
    Fetch data with rate limiting to respect API limits
    """
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Incremental Data Loading

```python
def incremental_load(start_page, end_page):
    """
    Load data incrementally to avoid memory issues
    """
    for page in range(start_page, end_page + 1):
        artifacts = fetch_artifacts(page=page)
        metadata_list, media_list, colors_list = extract_artifact_data(artifacts)
        df_metadata, df_media, df_colors = transform_data(metadata_list, media_list, colors_list)
        load_data(df_metadata, df_media, df_colors)
        print(f"Page {page} completed")
```

### Error Handling

```python
def robust_etl_pipeline(total_pages):
    """
    ETL pipeline with comprehensive error handling
    """
    failed_pages = []
    
    for page in range(1, total_pages + 1):
        try:
            artifacts = fetch_artifacts(page=page)
            metadata_list, media_list, colors_list = extract_artifact_data(artifacts)
            df_metadata, df_media, df_colors = transform_data(metadata_list, media_list, colors_list)
            load_data(df_metadata, df_media, df_colors)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            failed_pages.append(page)
            continue
    
    if failed_pages:
        print(f"Failed pages: {failed_pages}")
    
    return failed_pages
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
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        print("Database connection successful")
        cursor.close()
        conn.close()
    except Exception as e:
        print(f"Database connection failed: {e}")
```

### Memory Management for Large Datasets

```python
# Process data in chunks
def chunked_processing(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_list, media_list, colors_list = extract_artifact_data(chunk)
        df_metadata, df_media, df_colors = transform_data(metadata_list, media_list, colors_list)
        load_data(df_metadata, df_media, df_colors)
```

### Empty Query Results

```python
# Handle empty results gracefully
def safe_query_execution(query_name):
    df = execute_analytics_query(query_name)
    if df.empty:
        st.warning("No data found for this query")
        return None
    return df
```

This skill provides complete guidance for building end-to-end data engineering pipelines using the Harvard Art Museums API with ETL processes, SQL analytics, and interactive visualizations.
