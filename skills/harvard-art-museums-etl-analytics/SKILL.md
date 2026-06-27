---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifact data
  - query Harvard Art Museums API and load to database
  - visualize museum collection data with Streamlit
  - set up data engineering pipeline for art museum artifacts
  - analyze Harvard Art Museums collection with SQL
  - build museum data warehouse with Python and MySQL
  - create interactive art collection analytics app
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for extracting, transforming, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-ready ETL patterns, SQL analytics, and interactive visualization using Streamlit.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational database tables
- **SQL Analytics**: Provides 20+ predefined analytical queries for museum collection insights
- **Interactive Dashboards**: Visualizes query results using Streamlit and Plotly
- **Database Design**: Implements proper relational schema with foreign key relationships

## Architecture

```
Harvard API → Extract → Transform → Load → MySQL/TiDB → Analytics → Streamlit Dashboard
```

**Database Schema:**
- `artifactmetadata` - Core artifact information
- `artifactmedia` - Media files and images
- `artifactcolors` - Color analysis data

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

### API Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### Database Connection

```python
import mysql.connector

def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{num_pages}")
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts
```

### Transform: Normalize JSON to Relational Tables

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Metadata table
        metadata = {
            'artifact_id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'object_number': artifact.get('objectnumber'),
            'accession_year': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
        
        # Media table
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact_id,
                    'image_url': img.get('baseimageurl'),
                    'image_id': img.get('imageid'),
                    'width': img.get('width'),
                    'height': img.get('height'),
                    'format': img.get('format')
                }
                media_list.append(media)
        
        # Colors table
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact_id,
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

### Load: Batch Insert to Database

```python
def create_tables(connection):
    """
    Create database schema
    """
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            medium TEXT,
            dated VARCHAR(200),
            period VARCHAR(200),
            object_number VARCHAR(100),
            accession_year INT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(500),
            image_id INT,
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent DECIMAL(5,2),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()

def load_to_database(metadata_df, media_df, colors_df, connection):
    """
    Batch insert dataframes to database
    """
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, image_url, image_id, width, height, format)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
```

## SQL Analytics Queries

### Common Analytical Patterns

```python
# Query 1: Artifacts by culture
query_by_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 20
"""

# Query 2: Department distribution
query_by_department = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    GROUP BY department
    ORDER BY count DESC
"""

# Query 3: Media availability
query_media_stats = """
    SELECT 
        COUNT(DISTINCT am.artifact_id) as total_artifacts,
        COUNT(DISTINCT media.artifact_id) as with_media,
        ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.artifact_id), 2) as media_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia media ON am.artifact_id = media.artifact_id
"""

# Query 4: Color analysis
query_color_distribution = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 5: Century distribution
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
    LIMIT 20
"""

def execute_query(query, connection):
    """
    Execute SQL query and return results as DataFrame
    """
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection & ETL")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=100, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(API_KEY, num_pages=num_pages)
            st.success(f"Fetched {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = get_db_connection()
            create_tables(conn)
            load_to_database(metadata_df, media_df, colors_df, conn)
            conn.close()
            st.success("Data loaded to database")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Department Distribution": query_by_department,
        "Media Statistics": query_media_stats,
        "Color Analysis": query_color_distribution,
        "Century Distribution": query_by_century
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        results = execute_query(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) == 2:
            fig = px.bar(results, x=results.columns[0], y=results.columns[1])
            st.plotly_chart(fig)

def show_visualizations():
    st.header("📈 Interactive Visualizations")
    
    conn = get_db_connection()
    
    # Culture distribution
    df_culture = execute_query(query_by_culture, conn)
    fig1 = px.bar(df_culture, x='culture', y='artifact_count', 
                  title='Top 20 Cultures by Artifact Count')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color distribution
    df_colors = execute_query(query_color_distribution, conn)
    fig2 = px.pie(df_colors, names='color', values='usage_count',
                  title='Color Distribution in Collection')
    st.plotly_chart(fig2, use_container_width=True)
    
    conn.close()

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

### Incremental Data Loading

```python
def get_max_artifact_id(connection):
    """Get the latest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key, connection):
    """Load only new artifacts"""
    max_id = get_max_artifact_id(connection)
    
    # Fetch artifacts with ID greater than max_id
    params = {
        'apikey': api_key,
        'q': f'id:>{max_id}',
        'size': 100
    }
    
    response = requests.get(BASE_URL, params=params)
    new_artifacts = response.json().get('records', [])
    
    # Transform and load
    metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
    load_to_database(metadata_df, media_df, colors_df, connection)
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(api_key, num_pages, connection):
    """ETL with comprehensive error handling"""
    try:
        logger.info(f"Starting ETL for {num_pages} pages")
        raw_data = fetch_artifacts(api_key, num_pages)
        
        if not raw_data:
            raise ValueError("No data fetched from API")
        
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        
        logger.info(f"Transformed {len(metadata_df)} artifacts")
        
        load_to_database(metadata_df, media_df, colors_df, connection)
        
        logger.info("ETL completed successfully")
        return True
        
    except requests.RequestException as e:
        logger.error(f"API request failed: {e}")
        return False
    except mysql.connector.Error as e:
        logger.error(f"Database error: {e}")
        return False
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return False
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for rate limits
from time import sleep

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 429:  # Too Many Requests
            wait_time = 2 ** attempt
            logger.warning(f"Rate limited. Waiting {wait_time}s")
            sleep(wait_time)
            continue
        return response
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_pooled_connection():
    return db_pool.get_connection()
```

### Memory Issues with Large Datasets
```python
# Process in chunks
def chunked_load(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_to_database(metadata_df, media_df, colors_df, connection)
        logger.info(f"Processed chunk {i//chunk_size + 1}")
```

### Null Value Handling
```python
# Clean data before insertion
def clean_dataframe(df):
    """Replace None with appropriate defaults"""
    return df.fillna({
        'title': 'Untitled',
        'culture': 'Unknown',
        'century': 'Unknown',
        'accession_year': 0
    })
```

This skill enables AI agents to build complete ETL pipelines for museum data, implement SQL analytics, and create interactive dashboards using the Harvard Art Museums API.
