---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build etl pipeline for harvard art museums data
  - create analytics dashboard for museum artifacts
  - extract harvard art museums api data
  - set up streamlit data engineering app
  - implement museum data warehouse pipeline
  - query harvard artifacts collection database
  - visualize museum artifact analytics
  - design sql schema for museum data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a production-grade data engineering pipeline that:
- Extracts artifact data from Harvard Art Museums API
- Transforms nested JSON into relational database tables
- Loads data into MySQL/TiDB Cloud
- Provides SQL analytics queries
- Visualizes insights via Streamlit dashboards

The application handles pagination, rate limiting, batch inserts, and proper foreign key relationships across multiple tables.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

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

### Database Setup

```python
import mysql.connector
import os

# Connection configuration
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

### Create Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    objectnumber VARCHAR(255),
    title TEXT,
    dated VARCHAR(255),
    century VARCHAR(255),
    culture VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique TEXT,
    medium TEXT,
    description TEXT,
    provenance TEXT,
    url TEXT,
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Artifact Media Table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    image_width INT,
    image_height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
    INDEX idx_artifact (artifact_id)
);

-- Artifact Colors Table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
    INDEX idx_artifact (artifact_id)
);
```

## API Integration

### Extracting Data from Harvard Art Museums API

```python
import requests
import time
import os

class HarvardAPIExtractor:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
        
    def fetch_artifacts(self, page=1, size=100):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(self.base_url, params=params)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            print(f"API Error: {e}")
            return None
    
    def fetch_all_artifacts(self, max_pages=10):
        """Fetch multiple pages with rate limiting"""
        all_artifacts = []
        
        for page in range(1, max_pages + 1):
            data = self.fetch_artifacts(page=page)
            if data and 'records' in data:
                all_artifacts.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} artifacts")
                time.sleep(0.5)  # Rate limiting
            else:
                break
                
        return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
extractor = HarvardAPIExtractor(api_key)
artifacts = extractor.fetch_all_artifacts(max_pages=5)
```

## ETL Pipeline

### Transform and Load Data

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class ArtifactETL:
    def __init__(self, db_config):
        self.db_config = db_config
        
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract and transform artifact metadata"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'objectnumber': artifact.get('objectnumber'),
                'title': artifact.get('title'),
                'dated': artifact.get('dated'),
                'century': artifact.get('century'),
                'culture': artifact.get('culture'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'technique': artifact.get('technique'),
                'medium': artifact.get('medium'),
                'description': artifact.get('description'),
                'provenance': artifact.get('provenance'),
                'url': artifact.get('url')
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media/image data"""
        media_data = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for image in images:
                media_data.append({
                    'artifact_id': artifact_id,
                    'image_url': image.get('baseimageurl'),
                    'image_width': image.get('width'),
                    'image_height': image.get('height')
                })
        
        return pd.DataFrame(media_data)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color data"""
        color_data = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_data.append({
                    'artifact_id': artifact_id,
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent')
                })
        
        return pd.DataFrame(color_data)
    
    def load_to_database(self, df: pd.DataFrame, table_name: str):
        """Batch insert data into database"""
        conn = mysql.connector.connect(**self.db_config)
        cursor = conn.cursor()
        
        try:
            # Prepare batch insert
            columns = ', '.join(df.columns)
            placeholders = ', '.join(['%s'] * len(df.columns))
            query = f"REPLACE INTO {table_name} ({columns}) VALUES ({placeholders})"
            
            # Convert DataFrame to list of tuples
            data = [tuple(row) for row in df.values]
            
            # Execute batch insert
            cursor.executemany(query, data)
            conn.commit()
            print(f"Loaded {len(df)} rows into {table_name}")
            
        except Exception as e:
            print(f"Load error: {e}")
            conn.rollback()
        finally:
            cursor.close()
            conn.close()

# Usage
etl = ArtifactETL(db_config)

# Transform
metadata_df = etl.transform_metadata(artifacts)
media_df = etl.transform_media(artifacts)
colors_df = etl.transform_colors(artifacts)

# Load
etl.load_to_database(metadata_df, 'artifactmetadata')
etl.load_to_database(media_df, 'artifactmedia')
etl.load_to_database(colors_df, 'artifactcolors')
```

## SQL Analytics Queries

### Example Analytical Queries

```python
class ArtifactAnalytics:
    def __init__(self, db_config):
        self.db_config = db_config
    
    def execute_query(self, query: str) -> pd.DataFrame:
        """Execute SQL query and return DataFrame"""
        conn = mysql.connector.connect(**self.db_config)
        try:
            df = pd.read_sql(query, conn)
            return df
        finally:
            conn.close()
    
    # Query 1: Artifacts by Century
    def artifacts_by_century(self):
        query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10;
        """
        return self.execute_query(query)
    
    # Query 2: Top Cultures
    def top_cultures(self):
        query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15;
        """
        return self.execute_query(query)
    
    # Query 3: Media Availability
    def media_availability(self):
        query = """
        SELECT 
            CASE WHEN media_count > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM (
            SELECT a.id, COUNT(m.media_id) as media_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) as media_stats
        GROUP BY media_status;
        """
        return self.execute_query(query)
    
    # Query 4: Color Distribution
    def top_colors(self):
        query = """
        SELECT color_hex, COUNT(*) as frequency, AVG(color_percent) as avg_percent
        FROM artifactcolors
        WHERE color_hex IS NOT NULL
        GROUP BY color_hex
        ORDER BY frequency DESC
        LIMIT 20;
        """
        return self.execute_query(query)
    
    # Query 5: Department Distribution
    def artifacts_by_department(self):
        query = """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC;
        """
        return self.execute_query(query)

# Usage
analytics = ArtifactAnalytics(db_config)
century_stats = analytics.artifacts_by_century()
print(century_stats)
```

## Streamlit Dashboard

### Complete Dashboard Application

```python
import streamlit as st
import plotly.express as px
import os

# Page configuration
st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Initialize components
api_key = os.getenv('HARVARD_API_KEY')
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

analytics = ArtifactAnalytics(db_config)

# Sidebar
st.sidebar.title("Harvard Art Museums Analytics")
page = st.sidebar.selectbox(
    "Select Analysis",
    ["Overview", "Century Analysis", "Culture Insights", "Color Patterns", "Department Stats"]
)

# Main content
st.title("🎨 Harvard Art Museums Collection Analytics")

if page == "Overview":
    st.header("Collection Overview")
    
    # Total artifacts
    total_query = "SELECT COUNT(*) as total FROM artifactmetadata"
    total_df = analytics.execute_query(total_query)
    
    col1, col2, col3 = st.columns(3)
    with col1:
        st.metric("Total Artifacts", total_df['total'].iloc[0])
    
    # Media stats
    media_df = analytics.media_availability()
    st.subheader("Media Availability")
    fig = px.pie(media_df, values='count', names='media_status', title='Artifacts with/without Media')
    st.plotly_chart(fig)

elif page == "Century Analysis":
    st.header("Artifacts by Century")
    
    df = analytics.artifacts_by_century()
    st.dataframe(df)
    
    fig = px.bar(df, x='century', y='count', title='Artifact Distribution by Century')
    st.plotly_chart(fig, use_container_width=True)

elif page == "Culture Insights":
    st.header("Top Cultures Represented")
    
    df = analytics.top_cultures()
    
    fig = px.bar(df, x='culture', y='artifact_count', 
                 title='Top 15 Cultures by Artifact Count',
                 labels={'artifact_count': 'Number of Artifacts'})
    fig.update_xaxes(tickangle=45)
    st.plotly_chart(fig, use_container_width=True)

elif page == "Color Patterns":
    st.header("Color Usage in Artifacts")
    
    df = analytics.top_colors()
    
    # Create color visualization
    fig = px.bar(df, x='color_hex', y='frequency',
                 color='color_hex',
                 title='Most Frequent Colors in Collection',
                 color_discrete_map={row['color_hex']: row['color_hex'] for _, row in df.iterrows()})
    st.plotly_chart(fig, use_container_width=True)

elif page == "Department Stats":
    st.header("Artifacts by Department")
    
    df = analytics.artifacts_by_department()
    
    fig = px.pie(df, values='count', names='department', 
                 title='Department Distribution')
    st.plotly_chart(fig, use_container_width=True)

# Run with: streamlit run app.py
```

## Common Patterns

### Data Collection Pipeline

```python
def run_full_pipeline(api_key, db_config, num_pages=10):
    """Execute complete ETL pipeline"""
    # Extract
    extractor = HarvardAPIExtractor(api_key)
    artifacts = extractor.fetch_all_artifacts(max_pages=num_pages)
    
    # Transform & Load
    etl = ArtifactETL(db_config)
    
    metadata_df = etl.transform_metadata(artifacts)
    etl.load_to_database(metadata_df, 'artifactmetadata')
    
    media_df = etl.transform_media(artifacts)
    etl.load_to_database(media_df, 'artifactmedia')
    
    colors_df = etl.transform_colors(artifacts)
    etl.load_to_database(colors_df, 'artifactcolors')
    
    return len(artifacts)

# Execute
total = run_full_pipeline(os.getenv('HARVARD_API_KEY'), db_config, num_pages=5)
print(f"Pipeline complete: {total} artifacts processed")
```

### Incremental Data Updates

```python
def get_latest_artifact_id(db_config):
    """Get the latest artifact ID in database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    conn.close()
    return result[0] if result[0] else 0

def incremental_update(api_key, db_config):
    """Only fetch and load new artifacts"""
    latest_id = get_latest_artifact_id(db_config)
    # Implement logic to fetch only artifacts with id > latest_id
    pass
```

## Troubleshooting

### API Rate Limiting

```python
# Add exponential backoff
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
# Connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return db_pool.get_connection()
```

### Memory Optimization for Large Datasets

```python
def process_in_chunks(artifacts, chunk_size=1000):
    """Process artifacts in smaller batches"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df = etl.transform_metadata(chunk)
        etl.load_to_database(df, 'artifactmetadata')
        print(f"Processed chunk {i//chunk_size + 1}")
```

### Handling NULL Values

```python
# Clean data before loading
def clean_dataframe(df):
    """Replace None with NULL-safe values"""
    return df.where(pd.notnull(df), None)

# Use in transform
metadata_df = clean_dataframe(etl.transform_metadata(artifacts))
```
