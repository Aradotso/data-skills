---
name: harvard-art-museums-data-pipeline
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - set up analytics dashboard for Harvard artifacts collection
  - extract and analyze Harvard Art Museums data
  - build a Streamlit app with museum API data
  - query and visualize art collection metadata with SQL
  - implement ETL pipeline for cultural heritage data
  - connect to Harvard Art Museums API and store in database
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for cultural heritage data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Secure connection to Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract artifact metadata, media, and color data; transform nested JSON to relational format; load into SQL databases
- **SQL Analytics**: Pre-built analytical queries for insights on artifacts, cultures, centuries, media, and colors
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations for real-time data exploration

**Architecture**: API → ETL → SQL (MySQL/TiDB) → Analytics → Visualization (Streamlit + Plotly)

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your credentials
```

### Environment Configuration

Create a `.env` file with:

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

Get your Harvard API key at: https://www.harvardartmuseums.org/collections/api

## Database Setup

### SQL Schema Creation

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    technique VARCHAR(500),
    period VARCHAR(255),
    url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    imageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    caption TEXT,
    renditionnumber VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE
);
```

## Core ETL Pipeline

### 1. Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            print(f"Error: {response.status_code}")
            break
            
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return artifacts[:num_records]
```

### 2. Transform: Process and Normalize Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into relational format"""
    
    # Metadata extraction
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'description': artifact.get('description', ''),
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', '')[:255],
            'url': artifact.get('url', '')[:500]
        }
        metadata_records.append(metadata)
        
        # Extract media information
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact['id'],
                    'imageurl': image.get('baseimageurl', '')[:500],
                    'iiifbaseuri': image.get('iiifbaseuri', '')[:500],
                    'caption': image.get('caption', ''),
                    'renditionnumber': image.get('renditionnumber', '')[:50]
                }
                media_records.append(media)
        
        # Extract color information
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact['id'],
                    'color': color.get('color', '')[:50],
                    'spectrum': color.get('spectrum', '')[:50],
                    'hue': color.get('hue', '')[:50],
                    'percent': color.get('percent', 0)
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_data_to_db(metadata_df, media_df, colors_df):
    """Load transformed data into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, description, technique, period, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia 
                (artifact_id, imageurl, iiifbaseuri, caption, renditionnumber)
                VALUES (%s, %s, %s, %s, %s)
            """
            media_values = media_df.values.tolist()
            cursor.executemany(media_query, media_values)
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            colors_values = colors_df.values.tolist()
            cursor.executemany(colors_query, colors_values)
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Data Collection":
        show_data_collection_page()
    elif menu == "SQL Analytics":
        show_analytics_page()
    elif menu == "Visualizations":
        show_visualization_page()

def show_data_collection_page():
    st.header("📥 Data Collection from API")
    
    num_records = st.number_input(
        "Number of artifacts to fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_records)
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
        with st.spinner("Loading to database..."):
            load_data_to_db(metadata_df, media_df, colors_df)
            
        st.success(f"✅ Successfully loaded {len(metadata_df)} artifacts!")
        st.dataframe(metadata_df.head())

if __name__ == "__main__":
    main()
```

## Analytical SQL Queries

### Pre-built Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, 
               ROUND(AVG(percent), 2) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.id) as total_artifacts,
            COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
            ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
        FROM artifactmetadata am
        LEFT JOIN artifactmedia media ON am.id = media.artifact_id
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count,
               GROUP_CONCAT(DISTINCT culture SEPARATOR ', ') as cultures
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    try:
        query = ANALYTICS_QUERIES[query_name]
        cursor.execute(query)
        results = cursor.fetchall()
        return pd.DataFrame(results)
    except Error as e:
        st.error(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        cursor.close()
        conn.close()
```

### Analytics Page with Visualizations

```python
def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            results_df = execute_query(query_name)
        
        if not results_df.empty:
            st.subheader("Query Results")
            st.dataframe(results_df)
            
            # Auto-generate visualization
            st.subheader("Visualization")
            
            if len(results_df.columns) >= 2:
                x_col = results_df.columns[0]
                y_col = results_df.columns[1]
                
                fig = px.bar(
                    results_df.head(20),
                    x=x_col,
                    y=y_col,
                    title=f"{query_name} - Top 20",
                    labels={x_col: x_col.title(), y_col: y_col.title()}
                )
                fig.update_layout(xaxis_tickangle=-45)
                st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns and Usage

### Complete ETL Workflow

```python
def run_complete_etl_pipeline(num_artifacts=100):
    """Execute full ETL pipeline"""
    # Extract
    print(f"Extracting {num_artifacts} artifacts...")
    raw_data = fetch_artifacts(num_artifacts)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Loading to database...")
    load_data_to_db(metadata_df, media_df, colors_df)
    
    print("ETL pipeline completed successfully!")
    return {
        'metadata': len(metadata_df),
        'media': len(media_df),
        'colors': len(colors_df)
    }
```

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only():
    """Fetch only artifacts newer than existing data"""
    max_id = get_max_artifact_id()
    # Implement API filtering logic based on max_id
    # This depends on API capabilities
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < min_interval:
                time.sleep(min_interval - elapsed)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def api_request(url, params):
    return requests.get(url, params=params)
```

### Database Connection Pooling

```python
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

### Handling Missing Data

```python
def safe_get(dictionary, key, default=''):
    """Safely get value from nested dict"""
    value = dictionary.get(key, default)
    return value if value is not None else default

def clean_dataframe(df):
    """Clean DataFrame before DB insertion"""
    # Replace NaN with empty strings
    df = df.fillna('')
    # Truncate long strings
    for col in df.select_dtypes(include=['object']):
        if col in ['title', 'technique']:
            df[col] = df[col].str[:500]
        elif col in ['description']:
            df[col] = df[col].str[:5000]
    return df
```

This skill provides comprehensive guidance for building data engineering pipelines with the Harvard Art Museums API, including ETL workflows, SQL analytics, and interactive dashboards using Streamlit.
