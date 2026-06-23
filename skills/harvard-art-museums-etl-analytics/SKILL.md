---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - integrate Harvard Art Museums API
  - create a data engineering project with Streamlit
  - extract and analyze art collection data
  - set up SQL analytics for museum artifacts
  - build an interactive data visualization dashboard
  - process API data into relational database
  - create an end-to-end data pipeline
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to build and extend end-to-end data engineering applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact collection data.

## What This Project Does

The Harvard Artifacts Collection application:
- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud databases with batch inserts
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your free API key from: https://www.harvardartmuseums.org/collections/api

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
    classification VARCHAR(255),
    dated VARCHAR(255),
    century VARCHAR(255),
    period VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(255),
    provenance TEXT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_id INT,
    base_url VARCHAR(500),
    alt_text TEXT,
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    spectrum VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_artifacts=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': min(size, num_artifacts - len(artifacts)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data['records'])
            
            if len(data['records']) < size:
                break  # No more pages
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

class HarvardETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        self.connection = mysql.connector.connect(
            host=self.db_config['host'],
            port=self.db_config['port'],
            user=self.db_config['user'],
            password=self.db_config['password'],
            database=self.db_config['database']
        )
        return self.connection.cursor()
    
    def transform_metadata(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform artifact JSON to metadata DataFrame"""
        metadata = []
        
        for artifact in artifacts:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                'culture': artifact.get('culture', '')[:255],
                'classification': artifact.get('classification', '')[:255],
                'dated': artifact.get('dated', '')[:255],
                'century': artifact.get('century', '')[:255],
                'period': artifact.get('period', '')[:255],
                'division': artifact.get('division', '')[:255],
                'department': artifact.get('department', '')[:255],
                'technique': artifact.get('technique', '')[:500],
                'medium': artifact.get('medium', '')[:500],
                'dimensions': artifact.get('dimensions', '')[:500],
                'creditline': artifact.get('creditline', ''),
                'accession_number': artifact.get('accessionnumber', '')[:255],
                'provenance': artifact.get('provenance', ''),
                'url': artifact.get('url', '')[:500]
            })
        
        return pd.DataFrame(metadata)
    
    def transform_media(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform media/images data"""
        media_data = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for img in images:
                media_data.append({
                    'artifact_id': artifact_id,
                    'image_id': img.get('imageid'),
                    'base_url': img.get('baseimageurl', '')[:500],
                    'alt_text': img.get('alttext', ''),
                    'width': img.get('width'),
                    'height': img.get('height')
                })
        
        return pd.DataFrame(media_data)
    
    def transform_colors(self, artifacts: List[Dict]) -> pd.DataFrame:
        """Transform color data"""
        color_data = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_data.append({
                    'artifact_id': artifact_id,
                    'color_hex': color.get('hex', '')[:10],
                    'color_name': color.get('color', '')[:100],
                    'spectrum': color.get('spectrum', '')[:100],
                    'percentage': color.get('percent')
                })
        
        return pd.DataFrame(color_data)
    
    def load_data(self, df: pd.DataFrame, table_name: str):
        """Load DataFrame into SQL table with batch insert"""
        cursor = self.connect_db()
        
        # Prepare INSERT statement
        cols = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_query = f"INSERT IGNORE INTO {table_name} ({cols}) VALUES ({placeholders})"
        
        # Batch insert
        data_tuples = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data_tuples)
        self.connection.commit()
        
        print(f"Loaded {cursor.rowcount} rows into {table_name}")
        cursor.close()
    
    def run_pipeline(self, num_artifacts=100):
        """Execute full ETL pipeline"""
        # Extract
        print("Extracting data from API...")
        artifacts = fetch_artifacts(num_artifacts)
        
        # Transform
        print("Transforming data...")
        metadata_df = self.transform_metadata(artifacts)
        media_df = self.transform_media(artifacts)
        colors_df = self.transform_colors(artifacts)
        
        # Load
        print("Loading data to database...")
        self.load_data(metadata_df, 'artifactmetadata')
        self.load_data(media_df, 'artifactmedia')
        self.load_data(colors_df, 'artifactcolors')
        
        print("ETL pipeline completed!")
        return len(artifacts)
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'Has Images' ELSE 'No Images' END as status,
            COUNT(*) as artifact_count
        FROM (
            SELECT m.id, COUNT(a.media_id) as media_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia a ON m.id = a.artifact_id
            GROUP BY m.id
        ) as subquery
        GROUP BY status
    """,
    
    "Top Color Palettes": """
        SELECT color_name, COUNT(*) as occurrences
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY occurrences DESC
        LIMIT 10
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Average Colors per Artifact": """
        SELECT 
            AVG(color_count) as avg_colors,
            MAX(color_count) as max_colors,
            MIN(color_count) as min_colors
        FROM (
            SELECT artifact_id, COUNT(*) as color_count
            FROM artifactcolors
            GROUP BY artifact_id
        ) as subquery
    """
}

def execute_query(query: str, db_config: dict) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit application"""
    st.set_page_config(
        page_title="Harvard Art Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    db_config = {
        'host': os.getenv('DB_HOST'),
        'port': int(os.getenv('DB_PORT', 3306)),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # ETL Section
    st.header("📥 Data Collection")
    num_artifacts = st.number_input(
        "Number of artifacts to collect",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            etl = HarvardETL(db_config)
            count = etl.run_pipeline(num_artifacts)
            st.success(f"Successfully processed {count} artifacts!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        # Show SQL query
        with st.expander("View SQL Query"):
            st.code(query, language='sql')
        
        # Execute and display results
        df = execute_query(query, db_config)
        
        col1, col2 = st.columns([1, 1])
        
        with col1:
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
        
        with col2:
            st.subheader("Visualization")
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(url, params, delay=0.5):
    """Fetch data with rate limiting"""
    response = requests.get(url, params=params)
    time.sleep(delay)  # Prevent API rate limits
    return response
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default='', max_length=None):
    """Safely extract values with defaults and length limits"""
    value = dictionary.get(key, default)
    if max_length and isinstance(value, str):
        return value[:max_length]
    return value
```

### Batch Processing Large Datasets

```python
def process_in_batches(data, batch_size=1000):
    """Process large datasets in batches"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        yield batch
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Errors

```python
# Test database connection
try:
    connection = mysql.connector.connect(**db_config)
    print("✓ Database connection successful")
    connection.close()
except mysql.connector.Error as err:
    print(f"✗ Database error: {err}")
```

### Memory Issues with Large Datasets

```python
# Process in chunks instead of loading all at once
chunk_size = 100
for i in range(0, total_artifacts, chunk_size):
    artifacts = fetch_artifacts_page(page=i//chunk_size + 1, size=chunk_size)
    etl.process_batch(artifacts)
```

### Empty Query Results

```python
# Validate data exists before querying
def validate_data(db_config):
    """Check if tables have data"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    count = cursor.fetchone()[0]
    
    if count == 0:
        print("No data found. Run ETL pipeline first.")
    else:
        print(f"Found {count} artifacts")
    
    cursor.close()
    conn.close()
```

## Extension Ideas

- Add incremental ETL (only fetch new artifacts)
- Implement data quality checks and validation
- Create custom query builder in Streamlit
- Add export functionality (CSV, JSON, Excel)
- Integrate with other museum APIs
- Add caching for frequently run queries
- Implement user authentication for multi-user access
