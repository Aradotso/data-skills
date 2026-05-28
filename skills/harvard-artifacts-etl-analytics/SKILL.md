---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with MySQL/TiDB and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - set up Harvard artifacts data engineering project
  - create analytics dashboard for museum collection data
  - integrate Harvard Art Museums API with SQL database
  - build Streamlit app for artifact analytics
  - extract and transform Harvard museum data
  - query and visualize art collection metadata
  - implement batch data loading for museum artifacts
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates end-to-end data engineering workflows: extracting artifact data from the Harvard Art Museums API, transforming nested JSON into relational schemas, loading into MySQL/TiDB databases, and visualizing analytics through interactive Streamlit dashboards.

## What It Does

- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper foreign key relationships
- **Analytics Queries**: Provides 20+ predefined SQL queries for artifact analysis
- **Visualization**: Generates interactive Plotly charts from query results via Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
# Create .env or set in your environment:
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"

# Run the Streamlit app
streamlit run app.py
```

## Database Schema

The project uses three main tables:

**artifactmetadata**
- `objectid` (PRIMARY KEY)
- `title`, `culture`, `century`, `department`, `classification`, `dated`

**artifactmedia**
- `id` (PRIMARY KEY)
- `objectid` (FOREIGN KEY)
- `baseimageurl`, `format`, `height`, `width`

**artifactcolors**
- `id` (PRIMARY KEY)
- `objectid` (FOREIGN KEY)
- `color`, `spectrum`, `hue`, `percent`

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from typing import Dict, List

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response containing artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Filter for artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### 2. ETL Transform Logic

```python
import pandas as pd
from typing import List, Dict, Tuple

def transform_artifacts(records: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    Transform nested JSON into normalized dataframes.
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Extract metadata
        metadata = {
            'objectid': record.get('objectid'),
            'title': record.get('title', 'Unknown')[:255],
            'culture': record.get('culture', 'Unknown')[:100],
            'century': record.get('century', 'Unknown')[:50],
            'department': record.get('department', 'Unknown')[:100],
            'classification': record.get('classification', 'Unknown')[:100],
            'dated': record.get('dated', 'Unknown')[:100]
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        if record.get('images'):
            for img in record['images']:
                media = {
                    'objectid': record['objectid'],
                    'baseimageurl': img.get('baseimageurl', ''),
                    'format': img.get('format', 'Unknown'),
                    'height': img.get('height', 0),
                    'width': img.get('width', 0)
                }
                media_list.append(media)
        
        # Extract colors
        if record.get('colors'):
            for color in record['colors']:
                color_data = {
                    'objectid': record['objectid'],
                    'color': color.get('color', 'Unknown'),
                    'spectrum': color.get('spectrum', 'Unknown'),
                    'hue': color.get('hue', 'Unknown'),
                    'percent': color.get('percent', 0.0)
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. SQL Database Loading

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd
import os

def create_connection():
    """Create MySQL database connection."""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error: {e}")
        return None

def create_tables(connection):
    """Create database schema if not exists."""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(255),
            culture VARCHAR(100),
            century VARCHAR(50),
            department VARCHAR(100),
            classification VARCHAR(100),
            dated VARCHAR(100)
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            baseimageurl TEXT,
            format VARCHAR(50),
            height INT,
            width INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()

def batch_insert_metadata(connection, df: pd.DataFrame):
    """Batch insert artifact metadata."""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, department, classification, dated)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")
```

### 4. Analytics Queries

```python
def get_artifacts_by_culture(connection, limit: int = 10) -> pd.DataFrame:
    """Get top cultures by artifact count."""
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT %s
    """
    return pd.read_sql(query, connection, params=(limit,))

def get_color_distribution(connection) -> pd.DataFrame:
    """Analyze color usage across artifacts."""
    query = """
        SELECT 
            c.color,
            COUNT(DISTINCT c.objectid) as artifact_count,
            AVG(c.percent) as avg_percent
        FROM artifactcolors c
        GROUP BY c.color
        ORDER BY artifact_count DESC
        LIMIT 15
    """
    return pd.read_sql(query, connection)

def get_media_statistics(connection) -> pd.DataFrame:
    """Get image format and size statistics."""
    query = """
        SELECT 
            format,
            COUNT(*) as image_count,
            AVG(height) as avg_height,
            AVG(width) as avg_width
        FROM artifactmedia
        GROUP BY format
        ORDER BY image_count DESC
    """
    return pd.read_sql(query, connection)
```

### 5. Streamlit Visualization

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit dashboard."""
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    conn = create_connection()
    if not conn:
        st.error("Database connection failed. Check your credentials.")
        return
    
    # Analytics section
    st.header("📊 Analytics Queries")
    
    query_option = st.selectbox(
        "Select Analysis",
        [
            "Artifacts by Culture",
            "Color Distribution",
            "Media Statistics",
            "Century Timeline",
            "Department Breakdown"
        ]
    )
    
    if query_option == "Artifacts by Culture":
        df = get_artifacts_by_culture(conn, limit=15)
        st.dataframe(df)
        
        # Visualization
        fig = px.bar(df, x='culture', y='artifact_count',
                     title='Top Cultures by Artifact Count',
                     labels={'artifact_count': 'Count', 'culture': 'Culture'})
        st.plotly_chart(fig)
    
    elif query_option == "Color Distribution":
        df = get_color_distribution(conn)
        st.dataframe(df)
        
        fig = px.pie(df, values='artifact_count', names='color',
                     title='Color Distribution Across Artifacts')
        st.plotly_chart(fig)

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(num_pages: int = 5):
    """Execute full ETL pipeline."""
    api_key = os.getenv('HARVARD_API_KEY')
    conn = create_connection()
    create_tables(conn)
    
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = fetch_artifacts(api_key, page=page, size=100)
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(data['records'])
        
        all_metadata.append(metadata_df)
        all_media.append(media_df)
        all_colors.append(colors_df)
    
    # Combine and Load
    final_metadata = pd.concat(all_metadata, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    batch_insert_metadata(conn, final_metadata)
    batch_insert_media(conn, final_media)
    batch_insert_colors(conn, final_colors)
    
    conn.close()
    print("ETL pipeline completed successfully!")
```

### Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key: str, pages: int, delay: float = 1.0):
    """Fetch data with rate limiting to avoid API throttling."""
    results = []
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, page=page)
        results.append(data)
        time.sleep(delay)  # Respect API rate limits
    return results
```

## Troubleshooting

**API Authentication Error**
```python
# Verify API key is valid
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': os.getenv('HARVARD_API_KEY'), 'size': 1}
)
print(f"Status: {response.status_code}")
```

**Database Connection Issues**
```python
# Test connection
try:
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    print("Connection successful!")
except Error as e:
    print(f"Error: {e}")
```

**Empty Results**
- Check API pagination limits (max 100 per page)
- Verify `hasimage=1` parameter if filtering for images
- Confirm database tables exist before querying

**Streamlit Not Loading**
```bash
# Clear cache
streamlit cache clear

# Run with debug mode
streamlit run app.py --logger.level=debug
```
