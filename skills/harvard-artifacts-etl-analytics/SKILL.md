---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit for museum data
  - query Harvard artifacts collection with SQL
  - visualize museum artifact data with Plotly
  - set up data pipeline for Harvard Art Museums
  - analyze artifact metadata by culture and century
  - extract and transform museum API data
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates:

- **ETL Pipeline**: Extract artifact data from Harvard API, transform nested JSON into relational format, load into SQL database
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations
- **Real-world Architecture**: API → ETL → SQL → Analytics → Visualization

The application handles pagination, rate limiting, relational database design with foreign keys, and batch inserts for performance.

## Installation

### Prerequisites

```bash
# Python 3.8+
# MySQL or TiDB Cloud account
# Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)
```

### Setup

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

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    division VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    objectnumber VARCHAR(100),
    url TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    primaryimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(
            host=self.db_config['host'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
        return self.connection.cursor()
    
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform artifact JSON to metadata dataframe"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', ''),
                'culture': artifact.get('culture', ''),
                'century': artifact.get('century', ''),
                'division': artifact.get('division', ''),
                'department': artifact.get('department', ''),
                'classification': artifact.get('classification', ''),
                'dated': artifact.get('dated', ''),
                'objectnumber': artifact.get('objectnumber', ''),
                'url': artifact.get('url', '')
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract media information"""
        media = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            media.append({
                'artifact_id': artifact_id,
                'baseimageurl': artifact.get('baseimageurl', ''),
                'iiifbaseuri': artifact.get('iiifbaseuri', ''),
                'primaryimageurl': artifact.get('primaryimageurl', '')
            })
        
        return pd.DataFrame(media)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Extract color information from nested arrays"""
        colors = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            color_data = artifact.get('colors', [])
            
            for color in color_data:
                colors.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent', 0.0)
                })
        
        return pd.DataFrame(colors)
    
    def load_data(self, df: pd.DataFrame, table_name: str):
        """Batch insert data into SQL table"""
        cursor = self.connect_db()
        
        # Generate INSERT statement
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        # Batch insert
        data_tuples = [tuple(x) for x in df.to_numpy()]
        cursor.executemany(insert_query, data_tuples)
        
        self.connection.commit()
        cursor.close()
        print(f"Loaded {len(df)} records into {table_name}")

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

etl = HarvardETL(db_config)

# Extract
artifacts = fetch_artifacts(api_key, page=1, size=100)['records']

# Transform
metadata_df = etl.transform_metadata(artifacts)
media_df = etl.transform_media(artifacts)
colors_df = etl.transform_colors(artifacts)

# Load
etl.load_data(metadata_df, 'artifactmetadata')
etl.load_data(media_df, 'artifactmedia')
etl.load_data(colors_df, 'artifactcolors')
```

### 3. SQL Analytics Queries

```python
def get_artifacts_by_culture(cursor):
    """Count artifacts by culture"""
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """
    cursor.execute(query)
    return cursor.fetchall()

def get_century_distribution(cursor):
    """Artifact distribution by century"""
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """
    cursor.execute(query)
    return cursor.fetchall()

def get_color_analysis(cursor):
    """Most common colors across artifacts"""
    query = """
        SELECT c.color, c.spectrum, COUNT(DISTINCT c.artifact_id) as artifact_count,
               AVG(c.percent) as avg_percent
        FROM artifactcolors c
        GROUP BY c.color, c.spectrum
        HAVING artifact_count > 5
        ORDER BY artifact_count DESC
        LIMIT 15
    """
    cursor.execute(query)
    return cursor.fetchall()

def get_media_availability(cursor):
    """Analyze media/image availability"""
    query = """
        SELECT 
            COUNT(*) as total_artifacts,
            SUM(CASE WHEN primaryimageurl != '' THEN 1 ELSE 0 END) as with_primary_image,
            SUM(CASE WHEN iiifbaseuri != '' THEN 1 ELSE 0 END) as with_iiif
        FROM artifactmedia
    """
    cursor.execute(query)
    return cursor.fetchall()

def get_department_breakdown(cursor):
    """Artifacts by department"""
    query = """
        SELECT department, division, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department, division
        ORDER BY count DESC
    """
    cursor.execute(query)
    return cursor.fetchall()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Sidebar configuration
st.sidebar.title("⚙️ Configuration")
db_host = st.sidebar.text_input("Database Host", value=os.getenv('DB_HOST', ''))
db_user = st.sidebar.text_input("Database User", value=os.getenv('DB_USER', ''))
db_password = st.sidebar.text_input("Database Password", type="password")
db_name = st.sidebar.text_input("Database Name", value=os.getenv('DB_NAME', 'harvard_artifacts'))

# Main dashboard
st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Analytics section
tab1, tab2, tab3, tab4 = st.tabs(["Culture Analysis", "Century Distribution", "Color Insights", "Department Stats"])

with tab1:
    st.header("Artifacts by Culture")
    
    if st.button("Run Culture Analysis"):
        cursor = etl.connect_db()
        results = get_artifacts_by_culture(cursor)
        
        df = pd.DataFrame(results, columns=['Culture', 'Count'])
        
        # Display table
        st.dataframe(df)
        
        # Visualization
        fig = px.bar(df, x='Culture', y='Count', 
                     title='Top 20 Cultures by Artifact Count',
                     color='Count', color_continuous_scale='Viridis')
        st.plotly_chart(fig, use_container_width=True)

with tab2:
    st.header("Century Distribution")
    
    if st.button("Run Century Analysis"):
        cursor = etl.connect_db()
        results = get_century_distribution(cursor)
        
        df = pd.DataFrame(results, columns=['Century', 'Count'])
        
        fig = px.pie(df, values='Count', names='Century',
                     title='Artifact Distribution by Century')
        st.plotly_chart(fig, use_container_width=True)

with tab3:
    st.header("Color Analysis")
    
    if st.button("Run Color Analysis"):
        cursor = etl.connect_db()
        results = get_color_analysis(cursor)
        
        df = pd.DataFrame(results, columns=['Color', 'Spectrum', 'Artifact Count', 'Avg Percent'])
        
        fig = go.Figure(data=[
            go.Bar(name='Artifact Count', x=df['Color'], y=df['Artifact Count'])
        ])
        fig.update_layout(title='Most Common Colors in Artifacts')
        st.plotly_chart(fig, use_container_width=True)

with tab4:
    st.header("Department Statistics")
    
    if st.button("Run Department Analysis"):
        cursor = etl.connect_db()
        results = get_department_breakdown(cursor)
        
        df = pd.DataFrame(results, columns=['Department', 'Division', 'Count'])
        st.dataframe(df)
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_all_artifacts_paginated(api_key, max_pages=10, delay=1):
    """Fetch multiple pages with rate limiting"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            time.sleep(delay)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Incremental ETL Updates

```python
def incremental_load(etl, api_key, last_updated_id):
    """Load only new artifacts since last run"""
    artifacts = fetch_artifacts(api_key)['records']
    
    # Filter new artifacts
    new_artifacts = [a for a in artifacts if a['id'] > last_updated_id]
    
    if new_artifacts:
        metadata_df = etl.transform_metadata(new_artifacts)
        etl.load_data(metadata_df, 'artifactmetadata')
        
        return max([a['id'] for a in new_artifacts])
    
    return last_updated_id
```

## Troubleshooting

### API Rate Limit Errors

```python
# Add exponential backoff
import time

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise
```

### Database Connection Issues

```python
# Use connection pooling
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def get_connection():
    return connection_pool.get_connection()
```

### Handling NULL/Empty Values

```python
def safe_transform(value, default=''):
    """Handle NULL and missing values"""
    return value if value is not None else default

# In transform methods
'culture': safe_transform(artifact.get('culture'), 'Unknown')
```

### Memory Management for Large Datasets

```python
# Process in chunks
def batch_process(artifacts, batch_size=1000):
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        
        metadata_df = etl.transform_metadata(batch)
        etl.load_data(metadata_df, 'artifactmetadata')
        
        print(f"Processed batch {i // batch_size + 1}")
```
