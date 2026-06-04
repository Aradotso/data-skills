---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and museum data
  - structure Harvard museum API data into SQL tables
  - visualize art collection data with Plotly
  - implement pagination for Harvard Art Museums API
  - analyze artifact metadata with SQL queries
  - build data engineering project with museum API
---

# Harvard Art Museums Data Pipeline & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates how to build end-to-end data engineering pipelines using the Harvard Art Museums API. It extracts artifact data, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics through Streamlit dashboards.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into normalized SQL tables
- **Database Design**: Creates relational schema (metadata, media, colors)
- **Analytics**: Runs 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly charts in Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
```

Or configure in Streamlit secrets (`.streamlit/secrets.toml`):
```toml
HARVARD_API_KEY = "your_api_key_here"
```

### Database Configuration

```python
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=50, page=1)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### 2. Paginated Data Collection

```python
def collect_all_artifacts(api_key, max_records=1000):
    """
    Collect multiple pages of artifacts with rate limiting
    """
    import time
    
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            data = fetch_artifacts(api_key, size=size, page=page)
            records = data.get('records', [])
            
            if not records:
                break
            
            all_artifacts.extend(records)
            print(f"Page {page}: Collected {len(all_artifacts)} artifacts")
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts[:max_records]
```

### 3. ETL Pipeline

```python
import pandas as pd
import mysql.connector

def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Metadata table
        metadata_records.append({
            'object_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Media table
        for img in artifact.get('images', []):
            media_records.append({
                'object_id': artifact.get('id'),
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height')
            })
        
        # Colors table
        for color in artifact.get('colors', []):
            color_records.append({
                'object_id': artifact.get('id'),
                'color_name': color.get('color'),
                'hex_value': color.get('hex'),
                'percentage': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 4. Database Schema Creation

```python
def create_tables(db_config):
    """
    Create normalized SQL tables for artifacts
    """
    conn = mysql.connector.connect(**db_config)
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
            department VARCHAR(200),
            dated VARCHAR(200),
            url VARCHAR(500)
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url VARCHAR(500),
            format VARCHAR(50),
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
            hex_value VARCHAR(10),
            percentage FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 5. Data Loading

```python
def load_to_database(df_metadata, df_media, df_colors, db_config):
    """
    Batch insert data into SQL tables
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (object_id, title, culture, period, century, classification, department, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Load media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (object_id, image_id, base_url, format, width, height)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (object_id, color_name, hex_value, percentage)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 6. Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Top 10 Cultures": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors Used": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Images per Artifact": """
        SELECT 
            CASE 
                WHEN img_count = 0 THEN 'No Images'
                WHEN img_count = 1 THEN '1 Image'
                WHEN img_count BETWEEN 2 AND 5 THEN '2-5 Images'
                ELSE '5+ Images'
            END as image_range,
            COUNT(*) as artifacts
        FROM (
            SELECT m.object_id, COUNT(med.media_id) as img_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia med ON m.object_id = med.object_id
            GROUP BY m.object_id
        ) as img_stats
        GROUP BY image_range
    """
}

def run_query(query, db_config):
    """Execute SQL query and return DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 7. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", value=os.getenv('HARVARD_API_KEY'))
    
    # Data collection
    if st.sidebar.button("Fetch New Data"):
        with st.spinner("Collecting artifacts..."):
            artifacts = collect_all_artifacts(api_key, max_records=500)
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            load_to_database(df_meta, df_media, df_colors, DB_CONFIG)
            st.success(f"Loaded {len(artifacts)} artifacts!")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        df_result = run_query(query, DB_CONFIG)
        
        # Display table
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, 
                        x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_max_object_id(db_config):
    """Get the latest object_id already in database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(object_id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_load(api_key, db_config):
    """Only load new artifacts"""
    max_id = get_max_object_id(db_config)
    # Fetch only artifacts with id > max_id
    # Implementation depends on API filtering capabilities
```

### Pattern 2: Error Handling & Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_fetch(api_key, retries=3):
    """Fetch with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key)
        except Exception as e:
            logger.error(f"Attempt {attempt + 1} failed: {e}")
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)
```

## Troubleshooting

### API Rate Limiting
- Add `time.sleep()` between requests
- Check API documentation for rate limits
- Use batch requests when available

### Database Connection Issues
```python
# Test connection
try:
    conn = mysql.connector.connect(**DB_CONFIG)
    print("Database connected successfully!")
    conn.close()
except mysql.connector.Error as e:
    print(f"Error: {e}")
```

### Missing Data Fields
```python
# Use .get() with defaults
culture = artifact.get('culture', 'Unknown')
images = artifact.get('images', [])
```

### Streamlit Deployment
```bash
# Run locally
streamlit run app.py

# Deploy to Streamlit Cloud
# Add secrets in dashboard settings
```

## Running the Complete Pipeline

```bash
# Set environment variables
export HARVARD_API_KEY="your_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"

# Run Streamlit app
streamlit run app.py
```
