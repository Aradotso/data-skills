---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create ETL pipeline with Harvard Art Museums API
  - set up museum data analytics dashboard
  - implement artifact collection data engineering
  - build streamlit analytics app for art data
  - query and visualize Harvard museum collections
  - create SQL database for art museum artifacts
  - design ETL workflow for cultural heritage data
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering and analytics application that demonstrates ETL pipelines, SQL database design, and interactive visualization using the Harvard Art Museums API. It showcases real-world patterns for collecting, transforming, storing, and analyzing cultural artifacts data.

## What This Project Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with batch inserts
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

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

Get your API key from: https://www.harvardartmuseums.org/collections/api

```bash
# Create .env file
echo "HARVARD_API_KEY=your_api_key_here" > .env
```

### 2. Database Configuration

```python
# config.py or .env
DB_CONFIG = {
    'host': 'your_db_host',
    'port': 4000,  # or 3306 for MySQL
    'user': 'your_username',
    'password': 'your_password',
    'database': 'harvard_artifacts'
}
```

For TiDB Cloud:
```bash
echo "DB_HOST=gateway01.your-region.prod.aws.tidbcloud.com" >> .env
echo "DB_PORT=4000" >> .env
echo "DB_USER=your_username" >> .env
echo "DB_PASSWORD=your_password" >> .env
echo "DB_NAME=harvard_artifacts" >> .env
```

## Database Schema Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

def create_database_schema():
    """Initialize the Harvard artifacts database schema"""
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    cursor = conn.cursor()
    
    # Create database
    cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
    cursor.execute(f"USE {os.getenv('DB_NAME')}")
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dated VARCHAR(255),
            division VARCHAR(255),
            accession_number VARCHAR(100),
            provenance TEXT,
            description TEXT,
            url VARCHAR(500)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(1000),
            thumbnail_url VARCHAR(1000),
            alt_text TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_name VARCHAR(100),
            hex_code VARCHAR(10),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print("Database schema created successfully!")

if __name__ == "__main__":
    create_database_schema()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time
from typing import List, Dict

def fetch_artifacts(api_key: str, num_pages: int = 5, page_size: int = 100) -> List[Dict]:
    """
    Extract artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
        page_size: Records per page (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}/{num_pages} - {len(artifacts)} artifacts")
            
            # Rate limiting - be respectful of API
            time.sleep(1)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return all_artifacts
```

### Transform: Prepare Data for SQL

```python
import pandas as pd

def transform_artifacts(raw_artifacts: List[Dict]) -> tuple:
    """
    Transform raw API data into normalized dataframes
    
    Returns:
        (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dated': artifact.get('dated', '')[:255],
            'division': artifact.get('division', '')[:255],
            'accession_number': artifact.get('accessionyear', '')[:100],
            'provenance': artifact.get('provenance', ''),
            'description': artifact.get('description', ''),
            'url': artifact.get('url', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': primary_image[:1000],
                'thumbnail_url': artifact.get('baseimageurl', '')[:1000],
                'alt_text': artifact.get('title', '')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color_name': color.get('color', '')[:100],
                'hex_code': color.get('hex', '')[:10],
                'percentage': color.get('percent', 0.0)
            }
            colors_list.append(color_entry)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load: Insert into Database

```python
def load_to_database(metadata_df: pd.DataFrame, media_df: pd.DataFrame, 
                     colors_df: pd.DataFrame, db_config: Dict):
    """
    Load transformed data into MySQL/TiDB using batch inserts
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Load metadata
    metadata_tuples = [tuple(x) for x in metadata_df.to_numpy()]
    metadata_sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         technique, medium, dated, division, accession_number, provenance, 
         description, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    cursor.executemany(metadata_sql, metadata_tuples)
    print(f"Loaded {len(metadata_tuples)} metadata records")
    
    # Load media
    if not media_df.empty:
        media_tuples = [tuple(x) for x in media_df.to_numpy()]
        media_sql = """
            INSERT INTO artifactmedia (artifact_id, image_url, thumbnail_url, alt_text)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_sql, media_tuples)
        print(f"Loaded {len(media_tuples)} media records")
    
    # Load colors
    if not colors_df.empty:
        colors_tuples = [tuple(x) for x in colors_df.to_numpy()]
        colors_sql = """
            INSERT INTO artifactcolors (artifact_id, color_name, hex_code, percentage)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_sql, colors_tuples)
        print(f"Loaded {len(colors_tuples)} color records")
    
    conn.commit()
    cursor.close()
    conn.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px
import pandas as pd
from dotenv import load_dotenv
import os

load_dotenv()

st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

def main():
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar
    st.sidebar.header("Configuration")
    
    # Check API key
    api_key = os.getenv('HARVARD_API_KEY')
    if not api_key:
        st.error("Please set HARVARD_API_KEY in .env file")
        return
    
    # Navigation
    page = st.sidebar.radio(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection(api_key)
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection(api_key: str):
    st.header("📥 Data Collection from API")
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Number of pages", min_value=1, max_value=20, value=5)
    with col2:
        page_size = st.number_input("Page size", min_value=10, max_value=100, value=100)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_artifacts = fetch_artifacts(api_key, num_pages, page_size)
            st.success(f"Fetched {len(raw_artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            db_config = {
                'host': os.getenv('DB_HOST'),
                'port': int(os.getenv('DB_PORT', 3306)),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded to database!")
        
        # Show preview
        st.subheader("Data Preview")
        st.dataframe(metadata_df.head(10))

if __name__ == "__main__":
    main()
```

### SQL Analytics Queries

```python
# Common analytical queries for the dashboard

ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Media Availability Rate": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(DISTINCT a.id) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as media_percentage
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "Top 15 Colors Across Artifacts": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Color Diversity": """
        SELECT a.id, a.title, COUNT(c.color_id) as color_count
        FROM artifactmetadata a
        JOIN artifactcolors c ON a.id = c.artifact_id
        GROUP BY a.id, a.title
        ORDER BY color_count DESC
        LIMIT 10
    """,
    
    "Classification by Period": """
        SELECT classification, period, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND period IS NOT NULL
        GROUP BY classification, period
        ORDER BY count DESC
        LIMIT 20
    """
}

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        db_config = {
            'host': os.getenv('DB_HOST'),
            'port': int(os.getenv('DB_PORT', 3306)),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
        
        conn = mysql.connector.connect(**db_config)
        query = ANALYTICS_QUERIES[query_name]
        
        # Display query
        st.code(query, language='sql')
        
        # Execute and show results
        df = pd.read_sql(query, conn)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], title=query_name)
            st.plotly_chart(fig, use_container_width=True)
        
        conn.close()
```

## Common Patterns

### Incremental ETL Updates

```python
def incremental_load(api_key: str, last_update_timestamp: str):
    """Load only new artifacts since last update"""
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'after': last_update_timestamp,
        'size': 100
    }
    response = requests.get(base_url, params=params)
    return response.json().get('records', [])
```

### Error Handling for API Calls

```python
def robust_api_fetch(api_key: str, page: int, max_retries: int = 3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(
                "https://api.harvardartmuseums.org/object",
                params={'apikey': api_key, 'page': page, 'size': 100},
                timeout=30
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Custom Query Builder

```python
def build_custom_query(filters: Dict) -> str:
    """Build dynamic SQL based on user filters"""
    query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        query += f" AND culture = '{filters['culture']}'"
    if filters.get('century'):
        query += f" AND century = '{filters['century']}'"
    if filters.get('department'):
        query += f" AND department = '{filters['department']}'"
    
    return query
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
time.sleep(1)  # 1 second between calls

# Or use exponential backoff
import time
from functools import wraps

def rate_limit(delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            time.sleep(delay)
            return result
        return wrapper
    return decorator
```

### Database Connection Issues
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Connection successful!")
    conn.close()
except mysql.connector.Error as err:
    print(f"Error: {err}")
    # Check: host, port, credentials, firewall rules
```

### Large Dataset Memory Issues
```python
# Use chunked processing
def process_in_chunks(artifacts, chunk_size=1000):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_to_database(metadata_df, media_df, colors_df, db_config)
```

### Streamlit Caching for Performance
```python
@st.cache_data(ttl=3600)
def cached_query(query: str):
    """Cache query results for 1 hour"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```
