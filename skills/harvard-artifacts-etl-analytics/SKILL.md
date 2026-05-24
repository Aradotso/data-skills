---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I fetch Harvard Art Museums data
  - build an ETL pipeline for museum artifacts
  - query Harvard artifacts collection with SQL
  - create analytics dashboard with Streamlit
  - process Harvard API data into database
  - visualize museum artifact data
  - setup Harvard Art Museums data pipeline
  - analyze artifact metadata with SQL
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates real-world ETL (Extract, Transform, Load) pipelines that collect artifact data, store it in SQL databases, and visualize analytics through an interactive Streamlit dashboard.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

**Key Components:**
- API integration with Harvard Art Museums
- ETL pipeline for nested JSON transformation
- Relational database design (MySQL/TiDB)
- SQL analytics queries
- Interactive Plotly visualizations

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

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://docs.api.harvardartmuseums.org/
2. Request an API key (free)
3. Add to your `.env` file

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dated VARCHAR(200),
    url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    base_image_url VARCHAR(500),
    has_image BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    color_percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetch Data from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Extract artifact data from Harvard Art Museums API
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        return data.get('records', []), data.get('info', {})
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
        return [], {}

# Paginated data collection
def collect_all_artifacts(max_pages=10):
    """
    Collect artifacts across multiple pages
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        artifacts, info = fetch_artifacts(page=page)
        if not artifacts:
            break
        all_artifacts.extend(artifacts)
        print(f"Collected page {page}: {len(artifacts)} artifacts")
    
    return all_artifacts
```

### Transform: Clean and Structure Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform nested JSON into flat metadata structure
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image information
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        primary_image = artifact.get('primaryimageurl', '')
        
        media = {
            'artifact_id': artifact_id,
            'image_url': primary_image,
            'base_image_url': artifact.get('baseimageurl', ''),
            'has_image': 1 if primary_image else 0
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color information
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color_obj in colors:
            if isinstance(color_obj, dict):
                color_data = {
                    'artifact_id': artifact_id,
                    'color': color_obj.get('color', 'Unknown'),
                    'color_percentage': color_obj.get('percent', 0)
                }
                color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df_metadata):
    """
    Load artifact metadata into database
    """
    conn = get_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, medium, technique, dated, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    try:
        data_tuples = [tuple(x) for x in df_metadata.to_numpy()]
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error loading metadata: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()

def load_media(df_media):
    """
    Load artifact media into database
    """
    conn = get_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, image_url, base_image_url, has_image)
        VALUES (%s, %s, %s, %s)
    """
    
    try:
        data_tuples = [tuple(x) for x in df_media.to_numpy()]
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error loading media: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()

def load_colors(df_colors):
    """
    Load artifact colors into database
    """
    conn = get_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, color_percentage)
        VALUES (%s, %s, %s)
    """
    
    try:
        data_tuples = [tuple(x) for x in df_colors.to_numpy()]
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} color records")
        return True
    except Error as e:
        print(f"Error loading colors: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

## SQL Analytics Queries

### Common Analytical Queries

```python
# Query 1: Artifacts by culture
def query_artifacts_by_culture():
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """
    return execute_query(query)

# Query 2: Artifacts by century
def query_artifacts_by_century():
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """
    return execute_query(query)

# Query 3: Department distribution
def query_artifacts_by_department():
    query = """
        SELECT department, COUNT(*) as total
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total DESC
    """
    return execute_query(query)

# Query 4: Most common colors
def query_popular_colors():
    query = """
        SELECT color, COUNT(*) as frequency, 
               AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """
    return execute_query(query)

# Query 5: Artifacts with most images
def query_artifacts_with_images():
    query = """
        SELECT m.title, m.culture, COUNT(media.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifact_id
        WHERE media.has_image = 1
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 20
    """
    return execute_query(query)

# Query execution helper
def execute_query(query):
    """
    Execute SQL query and return DataFrame
    """
    conn = get_db_connection()
    if not conn:
        return pd.DataFrame()
    
    try:
        df = pd.read_sql(query, conn)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        conn.close()
```

## Streamlit Dashboard Application

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Select Module",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Data Collection":
        data_collection_page()
    elif menu == "SQL Analytics":
        analytics_page()
    elif menu == "Visualizations":
        visualization_page()

def data_collection_page():
    """
    ETL Pipeline interface
    """
    st.header("📥 Data Collection & ETL")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Pages to fetch", min_value=1, max_value=50, value=5)
    
    with col2:
        page_size = st.selectbox("Records per page", [10, 50, 100])
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_all_artifacts(max_pages=num_pages)
            st.success(f"Collected {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            st.success("Transformation complete")
        
        with st.spinner("Loading to database..."):
            load_metadata(df_metadata)
            load_media(df_media)
            load_colors(df_colors)
            st.success("Data loaded successfully!")

def analytics_page():
    """
    SQL Analytics interface
    """
    st.header("📊 SQL Analytics")
    
    query_options = {
        "Artifacts by Culture": query_artifacts_by_culture,
        "Artifacts by Century": query_artifacts_by_century,
        "Department Distribution": query_artifacts_by_department,
        "Popular Colors": query_popular_colors,
        "Artifacts with Images": query_artifacts_with_images
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df_result = query_options[selected_query]()
            
            if not df_result.empty:
                st.dataframe(df_result, use_container_width=True)
                
                # Auto-generate visualization
                if len(df_result.columns) >= 2:
                    fig = px.bar(
                        df_result.head(15),
                        x=df_result.columns[0],
                        y=df_result.columns[1],
                        title=selected_query
                    )
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No results found")

def visualization_page():
    """
    Interactive visualizations
    """
    st.header("📈 Data Visualizations")
    
    # Culture distribution
    df_culture = query_artifacts_by_culture()
    if not df_culture.empty:
        fig1 = px.bar(
            df_culture.head(10),
            x='culture',
            y='artifact_count',
            title='Top 10 Cultures by Artifact Count'
        )
        st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    df_colors = query_popular_colors()
    if not df_colors.empty:
        fig2 = px.pie(
            df_colors.head(8),
            values='frequency',
            names='color',
            title='Color Distribution in Artifacts'
        )
        st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# App will open at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def run_full_etl_pipeline():
    """
    Execute complete ETL workflow
    """
    # Extract
    print("Step 1: Extracting data from API...")
    artifacts = collect_all_artifacts(max_pages=10)
    
    # Transform
    print("Step 2: Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    # Load
    print("Step 3: Loading to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL pipeline completed successfully!")
```

### Incremental Data Updates

```python
def incremental_update():
    """
    Update only new artifacts
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Get max ID from database
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    
    # Fetch only newer artifacts
    artifacts = fetch_artifacts_after_id(max_id)
    
    # Process only new records
    if artifacts:
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata(df_metadata)
    
    cursor.close()
    conn.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, retries=3, delay=2):
    """
    Handle API rate limits with retry logic
    """
    for attempt in range(retries):
        try:
            artifacts, info = fetch_artifacts(page=page)
            return artifacts, info
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = delay * (2 ** attempt)
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return [], {}
```

### Database Connection Issues

```python
def test_db_connection():
    """
    Test database connectivity
    """
    try:
        conn = get_db_connection()
        if conn and conn.is_connected():
            cursor = conn.cursor()
            cursor.execute("SELECT 1")
            result = cursor.fetchone()
            print("✓ Database connection successful")
            cursor.close()
            conn.close()
            return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handle Missing Data

```python
def safe_transform(artifact, field, default='Unknown'):
    """
    Safely extract nested fields with defaults
    """
    value = artifact.get(field, default)
    return value if value else default

# Usage in transformation
metadata = {
    'culture': safe_transform(artifact, 'culture'),
    'century': safe_transform(artifact, 'century'),
    'department': safe_transform(artifact, 'department')
}
```

### Memory Optimization for Large Datasets

```python
def batch_load_artifacts(batch_size=100):
    """
    Process artifacts in batches to manage memory
    """
    page = 1
    
    while True:
        artifacts, info = fetch_artifacts(page=page, size=batch_size)
        
        if not artifacts:
            break
        
        # Transform and load batch
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata(df_metadata)
        
        print(f"Processed batch {page}")
        page += 1
        
        # Optional: limit total pages
        if page > 100:
            break
```

This skill provides comprehensive coverage of building ETL pipelines and analytics dashboards with the Harvard Art Museums API, including data extraction, transformation, SQL storage, and interactive visualization patterns.
