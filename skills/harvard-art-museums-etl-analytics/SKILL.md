---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app using Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build etl pipeline for harvard art museums
  - create data engineering app with streamlit
  - extract harvard museum artifacts into sql
  - analyze art museum data with python
  - set up artifact collection analytics
  - implement museum api data pipeline
  - visualize harvard art data
  - create museum data warehouse
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization using Streamlit.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational SQL tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations
- **Database Design**: Normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

## Architecture

```
Harvard Art Museums API → Python ETL → MySQL/TiDB → SQL Analytics → Streamlit Dashboard
```

## Installation

### Prerequisites

```bash
# Required Python packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Database Setup

Choose MySQL or TiDB Cloud:

```sql
CREATE DATABASE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    provenance TEXT,
    description TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### Environment Configuration

Create `.env` file:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your API key from: https://www.harvardartmuseums.org/collections/api

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
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
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
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        print(f"Fetched page {page}/{info['pages']}")
    return all_artifacts
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata table format"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'accessionyear': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance', ''),
            'description': artifact.get('description', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image data"""
    media_list = []
    
    for artifact in artifacts:
        if 'primaryimageurl' in artifact or 'images' in artifact:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl', ''),
                'iiifbaseuri': artifact.get('iiifbaseuri', ''),
                'primaryimageurl': artifact.get('primaryimageurl', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color data"""
    colors_list = []
    
    for artifact in artifacts:
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                }
                colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### 3. SQL Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
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
        (id, title, culture, period, century, classification, 
         department, dated, accessionyear, provenance, description)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = df_metadata.values.tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    print(f"Loaded {len(data)} metadata records")

def load_media(df_media):
    """Load media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
        VALUES (%s, %s, %s, %s)
    """
    
    data = df_media.values.tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    print(f"Loaded {len(data)} media records")

def load_colors(df_colors):
    """Load color data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, hue, percent)
        VALUES (%s, %s, %s, %s)
    """
    
    data = df_colors.values.tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    print(f"Loaded {len(data)} color records")
```

### 4. Complete ETL Pipeline

```python
def run_etl_pipeline(max_pages=5):
    """Execute complete ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("1. Extracting data from API...")
    artifacts = fetch_all_artifacts(max_pages=max_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("2. Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    # Load
    print("3. Loading data into database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL Pipeline Complete!")
    return {
        'artifacts': len(artifacts),
        'metadata_rows': len(df_metadata),
        'media_rows': len(df_media),
        'color_rows': len(df_colors)
    }
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Artifacts by culture
query_by_culture = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 10
"""

# Query 2: Artifacts by century
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Artifacts with images
query_with_images = """
    SELECT 
        COUNT(DISTINCT am.artifact_id) as artifacts_with_images,
        COUNT(*) as total_images
    FROM artifactmedia am
"""

# Query 4: Top colors
query_top_colors = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 10
"""

# Query 5: Department distribution
query_by_department = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""

# Query 6: Classification breakdown
query_by_classification = """
    SELECT classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE classification IS NOT NULL
    GROUP BY classification
    ORDER BY count DESC
    LIMIT 15
"""

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    max_pages = st.number_input(
        "Number of pages to fetch",
        min_value=1,
        max_value=50,
        value=5
    )
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            results = run_etl_pipeline(max_pages=max_pages)
            st.success("ETL Complete!")
            st.json(results)

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Image Statistics": query_with_images,
        "Top Colors": query_top_colors,
        "Department Distribution": query_by_department,
        "Classification Breakdown": query_by_classification
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(queries[selected_query])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    st.header("📈 Data Visualizations")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Culture Distribution")
        df_culture = execute_query(query_by_culture)
        fig = px.pie(df_culture, names='culture', values='count')
        st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        st.subheader("Century Timeline")
        df_century = execute_query(query_by_century)
        fig = px.bar(df_century, x='century', y='count')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] or 0

def fetch_new_artifacts_only():
    """Fetch only artifacts not yet in database"""
    max_id = get_max_artifact_id()
    # Implement logic to fetch artifacts with ID > max_id
```

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s...")
            time.sleep(wait_time)
```

### Data Quality Checks

```python
def validate_data_quality():
    """Run data quality checks"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    checks = {
        'null_titles': "SELECT COUNT(*) FROM artifactmetadata WHERE title IS NULL",
        'orphan_media': """
            SELECT COUNT(*) FROM artifactmedia am
            LEFT JOIN artifactmetadata meta ON am.artifact_id = meta.id
            WHERE meta.id IS NULL
        """,
        'missing_images': """
            SELECT COUNT(*) FROM artifactmetadata meta
            LEFT JOIN artifactmedia am ON meta.id = am.artifact_id
            WHERE am.id IS NULL
        """
    }
    
    results = {}
    for check_name, query in checks.items():
        cursor.execute(query)
        results[check_name] = cursor.fetchone()[0]
    
    cursor.close()
    conn.close()
    return results
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(page, requests_per_minute=60):
    """Respect API rate limits"""
    time.sleep(60 / requests_per_minute)
    return fetch_artifacts(page)
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Memory Optimization for Large Datasets

```python
def batch_process_artifacts(batch_size=100):
    """Process artifacts in batches to manage memory"""
    page = 1
    while True:
        artifacts = fetch_artifacts(page=page, size=batch_size)
        if not artifacts:
            break
        
        # Process batch
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata(df_metadata)
        
        page += 1
```

### Missing Environment Variables

```python
def validate_environment():
    """Ensure all required environment variables are set"""
    required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
    missing = [var for var in required_vars if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing environment variables: {', '.join(missing)}")
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Run specific analytics query
python analytics.py --query culture_distribution
```

This skill provides everything needed to build, deploy, and maintain a complete data engineering and analytics pipeline for museum artifact data.
