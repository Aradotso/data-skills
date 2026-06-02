---
name: harvard-artifacts-etl-analytics
description: Build end-to-end data pipelines from Harvard Art Museums API with ETL, SQL storage, and Streamlit analytics dashboards
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering project with Harvard artifacts collection
  - set up analytics dashboard for museum artifacts data
  - extract and load Harvard museum data into SQL database
  - build streamlit app for art collection analytics
  - implement ETL pipeline for Harvard Art Museums API
  - create SQL analytics for museum artifact metadata
  - visualize Harvard art collection data with plotly
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates production-grade ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection application:
- Fetches artifact data from the Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations to transform nested JSON into relational database tables
- Stores structured data in MySQL/TiDB Cloud with proper schema design
- Executes 20+ analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### 2. Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    technique VARCHAR(300),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(50),
    rank INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetch Data from API

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
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
def fetch_all_artifacts(max_pages=10):
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(records)
        print(f"Fetched page {page}/{info['pages']}")
        
        if page >= info['pages']:
            break
    
    return all_artifacts
```

### Transform: Clean and Structure Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into metadata table format
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'accession_number': artifact.get('accessionnumber'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for idx, image in enumerate(images):
            media = {
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl'),
                'media_type': 'image',
                'rank': idx + 1
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color information from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color_data in colors:
            color = {
                'artifact_id': artifact_id,
                'color': color_data.get('color'),
                'spectrum': color_data.get('spectrum'),
                'percentage': color_data.get('percent')
            }
            color_list.append(color)
    
    return pd.DataFrame(color_list)
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection using environment variables
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """
    Bulk insert metadata into database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, department, classification, technique, dated, accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(data)} metadata records")

def load_media(df_media):
    """
    Bulk insert media data
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, media_type, rank)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(data)} media records")

def load_colors(df_colors):
    """
    Bulk insert color data
    """
    conn = get_db_connection()
    cursor = cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_colors.values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(data)} color records")
```

## Complete ETL Workflow

```python
def run_etl_pipeline(max_pages=5):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(max_pages=max_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL pipeline completed successfully!")

# Execute
if __name__ == "__main__":
    run_etl_pipeline(max_pages=10)
```

## Streamlit Analytics Dashboard

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")
st.markdown("Interactive analytics on Harvard's artifact collection")

# Sidebar for configuration
with st.sidebar:
    st.header("Configuration")
    
    # ETL Section
    st.subheader("Data Collection")
    num_pages = st.number_input("Pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            run_etl_pipeline(max_pages=num_pages)
            st.success("ETL completed!")
    
    st.divider()
    
    # Query selection
    st.subheader("Analytics Queries")
    query_options = {
        "Artifacts by Culture": 1,
        "Artifacts by Century": 2,
        "Top Departments": 3,
        "Classification Distribution": 4,
        "Color Analysis": 5,
        "Media Availability": 6,
        "Artifacts per Period": 7,
        "Technique Usage": 8
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))

# Main content area
tab1, tab2, tab3 = st.tabs(["📊 Analytics", "📋 Data Explorer", "🔧 Database Stats"])

with tab1:
    st.header(selected_query)
    
    # Execute selected query
    query_id = query_options[selected_query]
    df_result = execute_analytical_query(query_id)
    
    # Display results
    col1, col2 = st.columns([1, 2])
    
    with col1:
        st.dataframe(df_result, use_container_width=True)
    
    with col2:
        if len(df_result) > 0:
            # Auto-generate visualization
            fig = create_visualization(df_result, selected_query)
            st.plotly_chart(fig, use_container_width=True)

with tab2:
    st.header("Raw Data Explorer")
    
    table_choice = st.selectbox("Select Table", 
                                ["artifactmetadata", "artifactmedia", "artifactcolors"])
    
    df_table = fetch_table_data(table_choice)
    st.dataframe(df_table, use_container_width=True)
    
    # Download option
    csv = df_table.to_csv(index=False)
    st.download_button("Download CSV", csv, f"{table_choice}.csv", "text/csv")

with tab3:
    st.header("Database Statistics")
    
    stats = get_database_stats()
    
    col1, col2, col3 = st.columns(3)
    col1.metric("Total Artifacts", stats['total_artifacts'])
    col2.metric("Total Images", stats['total_media'])
    col3.metric("Color Records", stats['total_colors'])
```

## Analytical SQL Queries

```python
def execute_analytical_query(query_id):
    """
    Execute predefined analytical queries
    """
    queries = {
        1: """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 15
        """,
        2: """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        3: """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        4: """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 10
        """,
        5: """
            SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 20
        """,
        6: """
            SELECT 
                COUNT(DISTINCT m.id) as artifacts_with_media,
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
                ROUND(COUNT(DISTINCT m.id) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
            FROM artifactmetadata m
            INNER JOIN artifactmedia am ON m.id = am.artifact_id
        """,
        7: """
            SELECT period, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE period IS NOT NULL
            GROUP BY period
            ORDER BY artifact_count DESC
            LIMIT 15
        """,
        8: """
            SELECT technique, COUNT(*) as count
            FROM artifactmetadata
            WHERE technique IS NOT NULL
            GROUP BY technique
            ORDER BY count DESC
            LIMIT 10
        """
    }
    
    conn = get_db_connection()
    df = pd.read_sql(queries[query_id], conn)
    conn.close()
    
    return df

def fetch_table_data(table_name, limit=1000):
    """
    Fetch data from specified table
    """
    conn = get_db_connection()
    query = f"SELECT * FROM {table_name} LIMIT {limit}"
    df = pd.read_sql(query, conn)
    conn.close()
    return df

def get_database_stats():
    """
    Get overall database statistics
    """
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    stats = {}
    
    cursor.execute("SELECT COUNT(*) as count FROM artifactmetadata")
    stats['total_artifacts'] = cursor.fetchone()['count']
    
    cursor.execute("SELECT COUNT(*) as count FROM artifactmedia")
    stats['total_media'] = cursor.fetchone()['count']
    
    cursor.execute("SELECT COUNT(*) as count FROM artifactcolors")
    stats['total_colors'] = cursor.fetchone()['count']
    
    cursor.close()
    conn.close()
    
    return stats
```

## Visualization Helpers

```python
def create_visualization(df, query_name):
    """
    Auto-generate appropriate visualization based on query results
    """
    if df.empty:
        return None
    
    columns = df.columns.tolist()
    
    # Bar chart for count/distribution queries
    if 'count' in columns or 'artifact_count' in columns:
        x_col = columns[0]
        y_col = [c for c in columns if 'count' in c.lower()][0]
        
        fig = px.bar(
            df.head(15),
            x=x_col,
            y=y_col,
            title=query_name,
            labels={x_col: x_col.replace('_', ' ').title(), 
                    y_col: y_col.replace('_', ' ').title()},
            color=y_col,
            color_continuous_scale='Viridis'
        )
        fig.update_layout(xaxis_tickangle=-45)
        
    # Pie chart for percentage queries
    elif 'percentage' in columns:
        fig = px.pie(
            df,
            names=columns[0],
            values='percentage',
            title=query_name
        )
    
    else:
        # Default to bar chart
        fig = px.bar(df.head(15), x=columns[0], y=columns[1], title=query_name)
    
    return fig
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """
    Get the highest artifact ID already in database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) as max_id FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """
    Only fetch new artifacts not in database
    """
    last_id = get_last_artifact_id()
    # Fetch artifacts with ID > last_id
    # Implementation depends on API filtering capabilities
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline(max_pages=5):
    """
    ETL with comprehensive error handling
    """
    try:
        artifacts = fetch_all_artifacts(max_pages=max_pages)
        logger.info(f"Extracted {len(artifacts)} artifacts")
        
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
        
        load_metadata(df_metadata)
        load_media(df_media)
        load_colors(df_colors)
        
        logger.info("ETL completed successfully")
        return True
        
    except requests.RequestException as e:
        logger.error(f"API request failed: {e}")
        return False
    except Error as e:
        logger.error(f"Database error: {e}")
        return False
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return False
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(page, size=100, delay=0.5):
    """
    Add delay between requests to respect rate limits
    """
    time.sleep(delay)
    return fetch_artifacts(page, size)
```

### Database Connection Issues

```python
def test_db_connection():
    """
    Test database connectivity
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def batch_load_artifacts(max_pages=100, batch_size=10):
    """
    Load data in batches to manage memory
    """
    for batch_start in range(1, max_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, max_pages + 1)
        
        artifacts = []
        for page in range(batch_start, batch_end):
            records, _ = fetch_artifacts(page=page)
            artifacts.extend(records)
        
        # Process batch
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata(df_metadata)
        
        print(f"Processed batch {batch_start}-{batch_end}")
        
        # Clear memory
        del artifacts, df_metadata
```

This skill provides comprehensive coverage of building production-grade ETL pipelines and analytics dashboards using the Harvard Art Museums API, suitable for data engineering and analytics projects.
