---
name: harvard-art-museums-data-pipeline
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create ETL workflow for museum artifacts data
  - set up analytics dashboard for art collection data
  - extract and transform Harvard museum API data
  - build Streamlit app for museum data visualization
  - implement SQL analytics on artifact collections
  - how to use Harvard Art Museums API with Python
  - create museum data engineering pipeline
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application provides:

- **API Data Collection**: Paginated retrieval from Harvard Art Museums API
- **ETL Pipeline**: Extract, transform, and load artifact data into relational databases
- **SQL Analytics**: Pre-built queries for artifact insights
- **Interactive Dashboard**: Streamlit-based visualization with Plotly charts
- **Database Design**: Normalized schema for artifacts, media, and color data

## Architecture Flow

```
Harvard API → Python ETL → MySQL/TiDB → SQL Analytics → Streamlit Dashboard
```

## Installation & Setup

### Prerequisites

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

### Configuration

Create a `.env` file for credentials:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema

### Core Tables

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    dated VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    provenance TEXT,
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

## ETL Pipeline Implementation

### 1. Extract Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, num_pages=10, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data['records'])
            print(f"Fetched page {page}/{num_pages}")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_pages=5)
```

### 2. Transform Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'dated': artifact.get('dated'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        })
        
        # Media
        if artifact.get('primaryimageurl'):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri') if artifact.get('images') else None
            })
        
        # Colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

# Usage
df_metadata, df_media, df_colors = transform_artifacts(artifacts)
```

### 3. Load Data to SQL

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_to_database(df_metadata, df_media, df_colors):
    """
    Batch insert dataframes into SQL tables
    """
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, dated, century, classification, department, 
         division, technique, medium, dimensions, creditline, provenance, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Load colors
        color_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(color_query, df_colors.values.tolist())
        
        connection.commit()
        print("Data loaded successfully!")
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## SQL Analytics Queries

### Common Analytics Patterns

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'With Images' ELSE 'Without Images' END as status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY status
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "classification_breakdown": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(query_name):
    """Execute analytics query and return results"""
    connection = create_db_connection()
    if not connection:
        return None
    
    try:
        query = ANALYTICS_QUERIES[query_name]
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        connection.close()
```

## Streamlit Dashboard Implementation

### Main App Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Museum Analytics", layout="wide")

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_etl_page():
    st.header("ETL Pipeline Control")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of Pages", min_value=1, max_value=100, value=5)
        page_size = st.number_input("Records per Page", min_value=10, max_value=100, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(os.getenv('HARVARD_API_KEY'), num_pages, page_size)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            success = load_to_database(df_metadata, df_media, df_colors)
            if success:
                st.success("Data loaded to database!")
            else:
                st.error("Failed to load data")

def show_analytics_page():
    st.header("SQL Analytics")
    
    query_choice = st.selectbox(
        "Select Analytics Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        df_result = execute_query(query_choice)
        
        if df_result is not None and not df_result.empty:
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) == 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=f"Results: {query_choice}"
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations_page():
    st.header("Data Visualizations")
    
    # Culture distribution
    df_culture = execute_query("artifacts_by_culture")
    if df_culture is not None:
        fig = px.bar(df_culture, x='culture', y='count', title='Artifacts by Culture')
        st.plotly_chart(fig, use_container_width=True)
    
    # Color analysis
    df_colors = execute_query("top_colors")
    if df_colors is not None:
        fig = px.pie(df_colors, names='color', values='frequency', title='Color Distribution')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns & Best Practices

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, num_pages, delay=1):
    """Fetch with rate limiting to avoid API throttling"""
    artifacts = []
    
    for page in range(1, num_pages + 1):
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page, 'size': 100}
        )
        
        if response.status_code == 200:
            artifacts.extend(response.json()['records'])
        
        time.sleep(delay)  # Rate limiting
    
    return artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_pages):
    """ETL pipeline with comprehensive error handling"""
    try:
        artifacts = fetch_artifacts(api_key, num_pages)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        if df_metadata.empty:
            raise ValueError("No metadata extracted")
        
        success = load_to_database(df_metadata, df_media, df_colors)
        
        return success, len(artifacts)
    
    except Exception as e:
        print(f"ETL pipeline failed: {e}")
        return False, 0
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params={'apikey': api_key, 'size': 1}
    )
    
    if response.status_code == 401:
        return "Invalid API key"
    elif response.status_code == 200:
        return "API connection successful"
    else:
        return f"Error: {response.status_code}"
```

### Database Connection Issues

```python
# Test database connectivity
def test_db_connection():
    try:
        connection = create_db_connection()
        if connection and connection.is_connected():
            connection.close()
            return "Database connection successful"
        else:
            return "Failed to connect to database"
    except Error as e:
        return f"Database error: {e}"
```

### Empty Results

- Verify API key is valid and has proper permissions
- Check if `hasimage=1` filter is too restrictive
- Ensure database tables exist before loading data
- Validate network connectivity and firewall settings

### Performance Optimization

```python
# Batch processing for large datasets
def batch_load(df, batch_size=1000):
    """Load data in batches for better performance"""
    connection = create_db_connection()
    cursor = connection.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch
        connection.commit()
    
    cursor.close()
    connection.close()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

This skill provides comprehensive coverage of building data engineering pipelines with museum API data, SQL analytics, and interactive visualizations using modern Python tools.
