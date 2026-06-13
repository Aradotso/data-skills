---
name: harvard-artifacts-etl-streamlit-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data engineering app with Streamlit and Harvard API
  - set up SQL analytics dashboard for museum artifact data
  - extract and analyze Harvard Art Museums collection data
  - build end-to-end data pipeline with API to visualization
  - create interactive analytics for art museum artifacts
  - implement ETL workflow for Harvard museum collections
  - visualize Harvard Art Museums data with Plotly and Streamlit
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates how to build production-ready ETL pipelines that extract artifact data, transform nested JSON into relational tables, load into SQL databases, and create interactive analytics dashboards using Streamlit.

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## API Configuration

### Getting Harvard Art Museums API Key

1. Visit https://harvardartmuseums.org/collections/api
2. Request an API key (free for non-commercial use)
3. Store in environment variable or `.env` file

### Basic API Request Pattern

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Example usage
data = fetch_artifacts(page=1, size=10)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Artifacts in this page: {len(data['records'])}")
```

## ETL Pipeline Implementation

### Extract Phase

```python
import requests
import time

def extract_artifacts_batch(num_pages=5, page_size=100):
    """Extract artifacts with rate limiting"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            response = requests.get(
                BASE_URL,
                params={
                    'apikey': os.getenv('HARVARD_API_KEY'),
                    'page': page,
                    'size': page_size
                },
                timeout=30
            )
            response.raise_for_status()
            data = response.json()
            
            all_artifacts.extend(data.get('records', []))
            print(f"Extracted page {page}: {len(data.get('records', []))} records")
            
            # Rate limiting (API allows 2500 requests/day)
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            continue
    
    return all_artifacts
```

### Transform Phase

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifacts into metadata table"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'object_id': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        })
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image information"""
    media_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        primary_image = artifact.get('primaryimageurl')
        
        if primary_image:
            media_list.append({
                'object_id': object_id,
                'image_url': primary_image,
                'is_primary': True
            })
        
        # Extract additional images
        for image in artifact.get('images', []):
            media_list.append({
                'object_id': object_id,
                'image_url': image.get('baseimageurl'),
                'is_primary': False,
                'width': image.get('width'),
                'height': image.get('height')
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color data from artifacts"""
    color_list = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        
        for color in artifact.get('colors', []):
            color_list.append({
                'object_id': object_id,
                'color_name': color.get('color'),
                'color_hex': color.get('hex'),
                'percentage': color.get('percent'),
                'hue': color.get('hue'),
                'saturation': color.get('saturation')
            })
    
    return pd.DataFrame(color_list)
```

### Load Phase

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create MySQL database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def create_tables():
    """Create database schema"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            division VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            medium TEXT,
            dimensions VARCHAR(500),
            description TEXT,
            provenance TEXT,
            url VARCHAR(500),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_url VARCHAR(1000),
            is_primary BOOLEAN,
            width INT,
            height INT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color_name VARCHAR(100),
            color_hex VARCHAR(10),
            percentage FLOAT,
            hue VARCHAR(50),
            saturation FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()

def load_metadata(df_metadata):
    """Bulk insert metadata with error handling"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (object_id, title, culture, period, century, classification, 
         division, department, dated, medium, dimensions, description, 
         provenance, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    data = df_metadata.fillna('').values.tolist()
    
    try:
        cursor.executemany(insert_query, data)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error loading metadata: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## SQL Analytics Queries

### Example Analytics Queries

```python
# Query 1: Artifacts by Department
query_by_department = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND department != ''
    GROUP BY department
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Century Distribution
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Most Common Colors
query_top_colors = """
    SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 4: Artifacts with Images
query_image_coverage = """
    SELECT 
        COUNT(DISTINCT m.object_id) as total_artifacts,
        COUNT(DISTINCT med.object_id) as artifacts_with_images,
        ROUND(COUNT(DISTINCT med.object_id) * 100.0 / COUNT(DISTINCT m.object_id), 2) as coverage_percentage
    FROM artifactmetadata m
    LEFT JOIN artifactmedia med ON m.object_id = med.object_id
"""

# Query 5: Culture Analysis
query_culture_analysis = """
    SELECT culture, COUNT(*) as count, 
           GROUP_CONCAT(DISTINCT classification SEPARATOR ', ') as classifications
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 20
"""

def execute_analytics_query(query):
    """Execute analytics query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Analysis",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection & ETL")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = extract_artifacts_batch(num_pages=num_pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            st.success("Transformation complete")
        
        with st.spinner("Loading to database..."):
            create_tables()
            load_metadata(df_metadata)
            st.success("Data loaded successfully!")
        
        st.dataframe(df_metadata.head())

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Department": query_by_department,
        "Century Distribution": query_by_century,
        "Top Colors": query_top_colors,
        "Image Coverage": query_image_coverage,
        "Culture Analysis": query_culture_analysis
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        df_result = execute_analytics_query(queries[selected_query])
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    st.header("📈 Interactive Visualizations")
    
    # Department distribution
    df_dept = execute_analytics_query(query_by_department)
    fig1 = px.pie(df_dept, values='artifact_count', names='department', 
                  title='Artifacts by Department')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    df_colors = execute_analytics_query(query_top_colors)
    fig2 = px.bar(df_colors, x='color_name', y='usage_count',
                  title='Most Common Colors in Collection')
    st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting for API Calls

```python
from time import sleep
from functools import wraps

def rate_limit(delay=0.5):
    """Decorator to add delay between API calls"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            sleep(delay)
            return result
        return wrapper
    return decorator

@rate_limit(delay=0.5)
def safe_api_call(url, params):
    return requests.get(url, params=params)
```

### Batch Processing with Progress

```python
def process_in_batches(items, batch_size=100, process_func=None):
    """Process items in batches with progress tracking"""
    total = len(items)
    
    for i in range(0, total, batch_size):
        batch = items[i:i+batch_size]
        process_func(batch)
        print(f"Processed {min(i+batch_size, total)}/{total}")
```

## Troubleshooting

### API Key Issues
```python
# Verify API key is loaded
assert os.getenv('HARVARD_API_KEY'), "API key not found in environment"

# Test API connectivity
def test_api_connection():
    try:
        response = requests.get(
            f"{BASE_URL}?apikey={os.getenv('HARVARD_API_KEY')}&size=1"
        )
        response.raise_for_status()
        print("API connection successful")
        return True
    except Exception as e:
        print(f"API connection failed: {e}")
        return False
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("Database connection successful")
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets
```python
# Process in chunks to avoid memory issues
def etl_pipeline_chunked(total_pages=100, chunk_size=10):
    for start_page in range(1, total_pages, chunk_size):
        end_page = min(start_page + chunk_size, total_pages)
        artifacts = extract_artifacts_batch(num_pages=end_page-start_page+1)
        
        # Transform and load immediately
        df = transform_artifact_metadata(artifacts)
        load_metadata(df)
        
        # Clear memory
        del artifacts, df
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run with custom port
streamlit run app.py --server.port 8501

# Run with auto-reload
streamlit run app.py --server.runOnSave true
```
