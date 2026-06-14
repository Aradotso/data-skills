---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with Streamlit, SQL, and Plotly visualizations
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data analytics dashboard with museum artifact data
  - extract and transform Harvard Art Museums data into SQL
  - visualize Harvard Art Museums collection data with Streamlit
  - build a data engineering pipeline for museum artifacts
  - query and analyze Harvard Art Museums API data
  - create interactive analytics for art collection data
  - setup ETL workflow for museum artifact metadata
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill teaches AI agents how to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization using Streamlit, following the architecture: **API → ETL → SQL → Analytics → Visualization**.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides a complete data pipeline solution that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** data using predefined SQL analytical queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

Key capabilities include artifact metadata collection, media details extraction, color pattern analysis, and department-wise distribution insights.

## Installation

### Prerequisites

```bash
# Required Python packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

Or using requirements file:

```bash
pip install -r requirements.txt
```

### Database Setup

1. **MySQL/TiDB Cloud Configuration**

```sql
-- Create database
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Create artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    dated VARCHAR(200),
    century VARCHAR(100),
    culture VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    provenance TEXT,
    description TEXT,
    url VARCHAR(500)
);

-- Create artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    image_height INT,
    image_width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### Environment Configuration

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Core Components

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
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages
def collect_artifacts(num_pages=5):
    all_artifacts = []
    for page in range(1, num_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        print(f"Fetched page {page}/{num_pages}: {len(records)} artifacts")
    return all_artifacts
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data into structured metadata"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'dated': artifact.get('dated', ''),
            'century': artifact.get('century', ''),
            'culture': artifact.get('culture', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'provenance': artifact.get('provenance', ''),
            'description': artifact.get('description', ''),
            'url': artifact.get('url', '')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Extract media/image information"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl', ''),
                'image_height': img.get('height', 0),
                'image_width': img.get('width', 0)
            })
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color palette information"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0)
            })
    
    return pd.DataFrame(color_records)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
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

def load_metadata_to_db(df):
    """Batch insert artifact metadata"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, dated, century, culture, classification, department, 
     division, technique, medium, provenance, description, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), dated=VALUES(dated), century=VALUES(century)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_media_to_db(df):
    """Batch insert artifact media data"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, image_url, image_height, image_width)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. SQL Analytics Queries

```python
# Predefined analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Top 10 Cultures": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN image_url IS NOT NULL THEN 'Has Image' 
                 ELSE 'No Image' END as media_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY media_status
    """,
    
    "Top Color Usage": """
        SELECT color_hex, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_analytics_query(query_name):
    """Execute analytical query and return results"""
    connection = get_db_connection()
    if not connection:
        return None
    
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
```

### 5. Streamlit Dashboard

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
    st.markdown("### ETL Pipeline & Interactive Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "ETL Pipeline", "Analytics Dashboard"]
    )
    
    if page == "Data Collection":
        st.header("📥 API Data Collection")
        
        num_pages = st.number_input("Number of pages to fetch", 
                                     min_value=1, max_value=50, value=5)
        
        if st.button("Fetch Artifacts"):
            with st.spinner("Fetching data from API..."):
                artifacts = collect_artifacts(num_pages)
                st.success(f"✅ Fetched {len(artifacts)} artifacts")
                st.session_state['artifacts'] = artifacts
    
    elif page == "ETL Pipeline":
        st.header("🔄 ETL Process")
        
        if 'artifacts' not in st.session_state:
            st.warning("⚠️ Please fetch data first from Data Collection page")
            return
        
        artifacts = st.session_state['artifacts']
        
        col1, col2, col3 = st.columns(3)
        
        with col1:
            if st.button("Transform Metadata"):
                df_meta = transform_artifact_metadata(artifacts)
                st.session_state['metadata'] = df_meta
                st.success(f"✅ Transformed {len(df_meta)} metadata records")
        
        with col2:
            if st.button("Transform Media"):
                df_media = transform_artifact_media(artifacts)
                st.session_state['media'] = df_media
                st.success(f"✅ Transformed {len(df_media)} media records")
        
        with col3:
            if st.button("Transform Colors"):
                df_colors = transform_artifact_colors(artifacts)
                st.session_state['colors'] = df_colors
                st.success(f"✅ Transformed {len(df_colors)} color records")
        
        st.markdown("---")
        
        if st.button("Load All to Database"):
            with st.spinner("Loading data to database..."):
                if 'metadata' in st.session_state:
                    load_metadata_to_db(st.session_state['metadata'])
                if 'media' in st.session_state:
                    load_media_to_db(st.session_state['media'])
                st.success("✅ Data loaded successfully!")
    
    elif page == "Analytics Dashboard":
        st.header("📊 SQL Analytics")
        
        query_name = st.selectbox(
            "Select Analysis",
            list(ANALYTICS_QUERIES.keys())
        )
        
        if st.button("Run Query"):
            with st.spinner("Executing query..."):
                df = execute_analytics_query(query_name)
                
                if df is not None and not df.empty:
                    st.dataframe(df, use_container_width=True)
                    
                    # Auto-generate visualization
                    if len(df.columns) >= 2:
                        fig = px.bar(
                            df, 
                            x=df.columns[0], 
                            y=df.columns[1],
                            title=query_name
                        )
                        st.plotly_chart(fig, use_container_width=True)
                else:
                    st.error("No data returned from query")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
# Full pipeline execution
def run_complete_etl_pipeline(num_pages=5):
    """Execute complete ETL pipeline"""
    # Extract
    print("Step 1: Extracting data from API...")
    artifacts = collect_artifacts(num_pages)
    
    # Transform
    print("Step 2: Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    # Load
    print("Step 3: Loading to database...")
    load_metadata_to_db(df_metadata)
    load_media_to_db(df_media)
    # Similar for colors
    
    print("✅ ETL Pipeline completed successfully!")
    return {
        'metadata': df_metadata,
        'media': df_media,
        'colors': df_colors
    }
```

### Custom Analytics Query

```python
def run_custom_analytics(custom_query):
    """Execute custom SQL analytics query"""
    connection = get_db_connection()
    
    try:
        df = pd.read_sql(custom_query, connection)
        return df
    except Error as e:
        print(f"Error: {e}")
        return None
    finally:
        connection.close()

# Example: Complex join query
complex_query = """
SELECT 
    am.department,
    am.classification,
    COUNT(DISTINCT am.id) as artifact_count,
    AVG(media.image_width) as avg_image_width
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.id = media.artifact_id
GROUP BY am.department, am.classification
HAVING artifact_count > 5
ORDER BY artifact_count DESC
"""

results = run_custom_analytics(complex_query)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, size=100, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            records, info = fetch_artifacts(page, size)
            return records, info
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            print("✅ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
            count = cursor.fetchone()[0]
            print(f"Total artifacts in database: {count}")
            cursor.close()
            connection.close()
            return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Memory Optimization for Large Datasets

```python
def batch_process_artifacts(num_pages=100, batch_size=10):
    """Process large datasets in batches to avoid memory issues"""
    for batch_start in range(1, num_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, num_pages + 1)
        
        # Process batch
        artifacts = []
        for page in range(batch_start, batch_end):
            records, _ = fetch_artifacts(page)
            artifacts.extend(records)
        
        # Transform and load batch
        df_meta = transform_artifact_metadata(artifacts)
        load_metadata_to_db(df_meta)
        
        print(f"Processed pages {batch_start}-{batch_end-1}")
```

## Key Configuration Options

### API Parameters

- `hasimage`: Filter artifacts with images (0 or 1)
- `size`: Records per page (max 100)
- `page`: Page number for pagination
- `classification`: Filter by classification type
- `culture`: Filter by culture

### Database Optimization

```sql
-- Add indexes for better query performance
CREATE INDEX idx_department ON artifactmetadata(department);
CREATE INDEX idx_century ON artifactmetadata(century);
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_artifact_id_media ON artifactmedia(artifact_id);
CREATE INDEX idx_artifact_id_colors ON artifactcolors(artifact_id);
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards using museum API data.
