---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering and analytics pipeline using Harvard Art Museums API with ETL, SQL, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data analytics app with museum artifacts
  - implement Harvard Art Museums API integration
  - set up SQL analytics for art collection data
  - build a Streamlit dashboard for museum data
  - extract and transform Harvard Art Museums metadata
  - create artifact visualization pipeline
  - design relational database for museum collections
---

# Harvard Art Museums ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for working with the Harvard Art Museums API. It demonstrates ETL pipeline development, relational database design, SQL analytics, and interactive visualization using Streamlit. The application extracts artifact metadata, transforms nested JSON into normalized tables, loads data into SQL databases, and provides 20+ analytical queries with auto-generated visualizations.

**Key capabilities:**
- API data collection with pagination and rate limiting
- Multi-table ETL pipeline (metadata, media, colors)
- MySQL/TiDB Cloud integration
- Interactive Streamlit analytics dashboard
- Plotly-based visualizations

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
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Harvard Art Museums API Setup

1. Get API key from: https://harvardartmuseums.org/collections/api
2. Store in environment variable or `.env` file:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Configuration

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

def get_db_connection():
    """Establish MySQL/TiDB connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, size: int = 100, page: int = 1) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        size: Number of records per page (max 100)
        page: Page number for pagination
    
    Returns:
        JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(api_key: str, total_records: int = 500) -> List[Dict]:
    """
    Collect multiple pages of artifacts with pagination
    """
    artifacts = []
    page = 1
    size = 100
    
    while len(artifacts) < total_records:
        data = fetch_artifacts(api_key, size=size, page=page)
        artifacts.extend(data.get('records', []))
        
        if len(data.get('records', [])) == 0:
            break
        
        page += 1
    
    return artifacts[:total_records]
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
from typing import List, Dict, Tuple

def transform_artifact_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Transform artifact data into metadata table
    
    Returns DataFrame with columns:
    - objectid, title, dated, culture, period, century, 
      classification, department, technique, medium
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'dated': artifact.get('dated'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract media/image information from artifacts
    """
    media = []
    
    for artifact in artifacts:
        objectid = artifact.get('id')
        
        # Primary image
        if artifact.get('primaryimageurl'):
            media.append({
                'objectid': objectid,
                'mediatype': 'image',
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri')
            })
        
        # Additional images
        for img in artifact.get('images', []):
            media.append({
                'objectid': objectid,
                'mediatype': 'image',
                'baseimageurl': img.get('baseimageurl'),
                'primaryimageurl': img.get('iiifbaseuri'),
                'iiifbaseuri': img.get('iiifbaseuri')
            })
    
    return pd.DataFrame(media)

def transform_artifact_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """
    Extract color information from artifacts
    """
    colors = []
    
    for artifact in artifacts:
        objectid = artifact.get('id')
        
        for color in artifact.get('colors', []):
            colors.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent'),
                'css3': color.get('css3')
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Schema Creation

```python
def create_tables(connection):
    """Create database tables for Harvard artifacts data"""
    cursor = connection.cursor()
    
    # Artifact Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            dated VARCHAR(100),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            technique VARCHAR(500),
            medium VARCHAR(500),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Artifact Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            mediatype VARCHAR(50),
            baseimageurl VARCHAR(500),
            primaryimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            css3 VARCHAR(50),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Data Loading

```python
def load_data_to_sql(df: pd.DataFrame, table_name: str, connection):
    """
    Batch insert DataFrame into SQL table
    """
    cursor = connection.cursor()
    
    # Prepare batch insert
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    # Execute batch insert
    cursor.executemany(query, data)
    connection.commit()
    cursor.close()
    
    return len(data)
```

### 5. Streamlit Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    """Main Streamlit application"""
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Data Collection":
        show_data_collection()
    elif menu == "ETL Pipeline":
        show_etl_pipeline()
    elif menu == "SQL Analytics":
        show_sql_analytics()
    elif menu == "Visualizations":
        show_visualizations()

def show_data_collection():
    """Data collection interface"""
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    num_records = st.slider("Number of Records", 100, 1000, 500)
    
    if st.button("Collect Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_all_artifacts(api_key, num_records)
            st.success(f"Collected {len(artifacts)} artifacts")
            
            # Store in session state
            st.session_state['artifacts'] = artifacts
            
            # Preview
            st.dataframe(pd.DataFrame(artifacts).head())

def show_etl_pipeline():
    """ETL pipeline execution interface"""
    st.header("🔄 ETL Pipeline")
    
    if 'artifacts' not in st.session_state:
        st.warning("Please collect data first")
        return
    
    artifacts = st.session_state['artifacts']
    
    col1, col2, col3 = st.columns(3)
    
    with col1:
        st.subheader("Metadata")
        df_metadata = transform_artifact_metadata(artifacts)
        st.write(f"Records: {len(df_metadata)}")
        st.dataframe(df_metadata.head())
    
    with col2:
        st.subheader("Media")
        df_media = transform_artifact_media(artifacts)
        st.write(f"Records: {len(df_media)}")
        st.dataframe(df_media.head())
    
    with col3:
        st.subheader("Colors")
        df_colors = transform_artifact_colors(artifacts)
        st.write(f"Records: {len(df_colors)}")
        st.dataframe(df_colors.head())
    
    if st.button("Load to Database"):
        conn = get_db_connection()
        create_tables(conn)
        
        load_data_to_sql(df_metadata, 'artifactmetadata', conn)
        load_data_to_sql(df_media, 'artifactmedia', conn)
        load_data_to_sql(df_colors, 'artifactcolors', conn)
        
        conn.close()
        st.success("Data loaded successfully!")
```

### 6. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Color Usage Analysis": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Classification Summary": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_query(query: str, connection) -> pd.DataFrame:
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

def show_sql_analytics():
    """SQL analytics dashboard"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df_result = execute_query(ANALYTICAL_QUERIES[query_name], conn)
        conn.close()
        
        # Display results
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl_pipeline():
    """Execute full ETL pipeline"""
    # 1. Extract
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = collect_all_artifacts(api_key, total_records=1000)
    
    # 2. Transform
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    # 3. Load
    conn = get_db_connection()
    create_tables(conn)
    
    load_data_to_sql(df_metadata, 'artifactmetadata', conn)
    load_data_to_sql(df_media, 'artifactmedia', conn)
    load_data_to_sql(df_colors, 'artifactcolors', conn)
    
    conn.close()
    
    print("ETL pipeline completed successfully!")
```

### Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting:**
```python
import time

def fetch_with_retry(api_key, size, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, size, page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Too many requests
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

**Database Connection Issues:**
```python
def get_db_connection_with_retry():
    """Retry database connection"""
    max_attempts = 3
    for attempt in range(max_attempts):
        try:
            return get_db_connection()
        except mysql.connector.Error as e:
            if attempt == max_attempts - 1:
                raise
            time.sleep(2)
```

**Handling Missing Data:**
```python
def safe_transform(artifacts):
    """Handle missing fields gracefully"""
    df = transform_artifact_metadata(artifacts)
    
    # Fill missing values
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Undated')
    
    return df
```
