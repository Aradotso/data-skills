---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API using ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL pipeline for museum artifacts data
  - create analytics dashboard for Harvard Art Museums
  - extract and visualize museum collection data
  - build streamlit app with Harvard museum API
  - query and analyze art artifacts with SQL
  - implement batch data loading for museum collections
  - visualize art museum data with plotly
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build production-ready data engineering applications using the Harvard Art Museums API. It covers ETL pipeline construction, SQL database design, analytics query execution, and interactive visualization with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App demonstrates a complete data engineering workflow:

1. **Extract** artifact data from Harvard Art Museums API with pagination and rate limiting
2. **Transform** nested JSON into normalized relational tables
3. **Load** data into SQL databases (MySQL/TiDB Cloud) with batch operations
4. **Analyze** using 20+ predefined SQL queries
5. **Visualize** results through interactive Streamlit dashboards with Plotly charts

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
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to your `.env` file

## Database Schema

The application uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    technique VARCHAR(500),
    period VARCHAR(200),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    base_image_url TEXT,
    has_image BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Code Patterns

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
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
        
        return {
            'records': data.get('records', []),
            'total_pages': data.get('info', {}).get('pages', 0),
            'total_records': data.get('info', {}).get('totalrecords', 0)
        }
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

# Fetch multiple pages
def collect_artifacts(max_pages=10):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        result = fetch_artifacts(page=page)
        if result:
            all_artifacts.extend(result['records'])
            print(f"Fetched page {page}/{result['total_pages']}")
        else:
            break
    
    return all_artifacts
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data into structured metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'dated': artifact.get('dated', '')[:200],
            'department': artifact.get('department', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', '')[:200],
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media information"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        primary_image = artifact.get('primaryimageurl')
        images = artifact.get('images', [])
        
        media = {
            'artifact_id': artifact_id,
            'image_url': primary_image,
            'base_image_url': artifact.get('baseimageurl'),
            'has_image': 1 if primary_image else 0
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color data"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percentage': float(color.get('percent', 0))
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_database_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def batch_insert_metadata(df, batch_size=100):
    """Batch insert artifact metadata"""
    connection = get_database_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, dated, department, classification, technique, period, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    try:
        for i in range(0, len(df), batch_size):
            batch = df.iloc[i:i+batch_size]
            data = [tuple(row) for row in batch.values]
            cursor.executemany(insert_query, data)
            connection.commit()
            print(f"Inserted batch {i//batch_size + 1}")
        
        return True
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. Analytics Queries

```python
def execute_analytics_query(query_name):
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
            LIMIT 15
        """,
        
        'top_cultures': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'media_coverage': """
            SELECT 
                has_image,
                COUNT(*) as count,
                ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM artifactmedia), 2) as percentage
            FROM artifactmedia
            GROUP BY has_image
        """,
        
        'color_distribution': """
            SELECT 
                color,
                COUNT(*) as usage_count,
                ROUND(AVG(percentage), 2) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        'department_breakdown': """
            SELECT 
                department,
                COUNT(*) as total_artifacts,
                COUNT(DISTINCT classification) as unique_classifications
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
        """
    }
    
    connection = get_database_connection()
    if not connection:
        return None
    
    try:
        cursor = connection.cursor(dictionary=True)
        cursor.execute(queries.get(query_name, queries['artifacts_by_century']))
        results = cursor.fetchall()
        return pd.DataFrame(results)
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        cursor.close()
        connection.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualization_page()

def show_data_collection_page():
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = collect_artifacts(max_pages=num_pages)
            st.success(f"Collected {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            batch_insert_metadata(metadata_df)
            st.success("Data loaded to database")

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    query_options = {
        "Artifacts by Century": "artifacts_by_century",
        "Top Cultures": "top_cultures",
        "Media Coverage": "media_coverage",
        "Color Distribution": "color_distribution",
        "Department Breakdown": "department_breakdown"
    }
    
    selected_query = st.selectbox("Select Analytics Query", list(query_options.keys()))
    
    if st.button("Execute Query"):
        df = execute_analytics_query(query_options[selected_query])
        
        if df is not None and not df.empty:
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                           title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Use Cases

### Full ETL Pipeline Execution

```python
# Complete pipeline from API to database
artifacts = collect_artifacts(max_pages=10)
metadata_df = transform_artifact_metadata(artifacts)
media_df = transform_artifact_media(artifacts)
colors_df = transform_artifact_colors(artifacts)

batch_insert_metadata(metadata_df)
# Insert media and colors similarly
```

### Custom Analytics Query

```python
def custom_query(sql):
    """Execute custom SQL query"""
    connection = get_database_connection()
    cursor = connection.cursor(dictionary=True)
    cursor.execute(sql)
    results = cursor.fetchall()
    cursor.close()
    connection.close()
    return pd.DataFrame(results)

# Example usage
df = custom_query("""
    SELECT c.culture, AVG(col.percentage) as avg_color_usage
    FROM artifactmetadata c
    JOIN artifactcolors col ON c.id = col.artifact_id
    GROUP BY c.culture
    HAVING COUNT(*) > 10
    ORDER BY avg_color_usage DESC
""")
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting**
```python
import time

def fetch_with_retry(page, retries=3):
    for attempt in range(retries):
        result = fetch_artifacts(page)
        if result:
            return result
        time.sleep(2 ** attempt)  # Exponential backoff
    return None
```

**Database Connection Issues**
```python
# Test connection
connection = get_database_connection()
if connection and connection.is_connected():
    print("Database connected successfully")
    connection.close()
else:
    print("Check DB_HOST, DB_USER, DB_PASSWORD in .env")
```

**Memory Management for Large Datasets**
```python
# Process in chunks
def process_large_dataset(max_pages=100, chunk_size=10):
    for chunk_start in range(1, max_pages, chunk_size):
        artifacts = collect_artifacts_range(chunk_start, chunk_start + chunk_size)
        df = transform_artifact_metadata(artifacts)
        batch_insert_metadata(df)
        print(f"Processed pages {chunk_start} to {chunk_start + chunk_size}")
```

This skill enables complete data engineering workflows with museum collection data, from API extraction through visualization.
