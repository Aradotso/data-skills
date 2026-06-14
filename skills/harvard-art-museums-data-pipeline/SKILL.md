---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - set up analytics dashboard for art collection data
  - process Harvard museum API data with Python
  - build Streamlit app for art museums analytics
  - extract and transform Harvard artifacts data
  - visualize museum collection data with SQL queries
  - implement data engineering pipeline for art data
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project provides an end-to-end data engineering and analytics application built using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization using Streamlit.

## What It Does

The application enables:
- Dynamic artifact data collection from Harvard Art Museums API
- ETL (Extract, Transform, Load) operations on museum data
- Structured storage in SQL databases (MySQL/TiDB Cloud)
- Analytical SQL queries on artifact metadata, media, and colors
- Interactive visualization dashboards with Plotly

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://harvardartmuseums.org/collections/api)

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

### Requirements.txt typically includes:
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Database Setup

Create the database schema with three main tables:

```python
import mysql.connector
import os

# Database connection
def create_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def setup_database():
    conn = create_connection()
    cursor = conn.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            dated VARCHAR(255),
            classification VARCHAR(255),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            department VARCHAR(255),
            division VARCHAR(255),
            creditline TEXT,
            accession_number VARCHAR(100),
            url VARCHAR(500)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(100),
            base_url VARCHAR(500),
            public_caption TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()

setup_database()
```

### API Configuration

```python
import os
import requests
from typing import Dict, List

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org"
        
    def get_objects(self, page: int = 1, size: int = 100) -> Dict:
        """Fetch objects from Harvard Art Museums API"""
        url = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
    
    def get_object_by_id(self, object_id: int) -> Dict:
        """Fetch specific object by ID"""
        url = f"{self.base_url}/object/{object_id}"
        params = {'apikey': self.api_key}
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
```

## ETL Pipeline Implementation

### Extract Phase

```python
import pandas as pd
import time

def extract_artifacts(client: HarvardAPIClient, num_pages: int = 5) -> List[Dict]:
    """Extract artifact data with pagination and rate limiting"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = client.get_objects(page=page, size=100)
            records = data.get('records', [])
            all_artifacts.extend(records)
            
            print(f"Extracted page {page}: {len(records)} artifacts")
            time.sleep(0.5)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform Phase

```python
def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """Transform nested JSON into relational dataframes"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Transform metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        })
        
        # Transform media
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'base_url': img.get('baseimageurl'),
                'public_caption': img.get('publiccaption')
            })
        
        # Transform colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load Phase

```python
def load_to_database(metadata_df: pd.DataFrame, 
                     media_df: pd.DataFrame, 
                     colors_df: pd.DataFrame):
    """Batch insert data into SQL database"""
    conn = create_connection()
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, dated, classification, medium, 
             dimensions, department, division, creditline, accession_number, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Load media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, media_type, base_url, public_caption)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    print("Data loaded successfully")
```

## Streamlit Analytics Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    st.markdown("### ETL Pipeline & SQL Analytics")
    
    # Sidebar for ETL controls
    st.sidebar.header("Data Collection")
    num_pages = st.sidebar.slider("Number of pages to fetch", 1, 10, 5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Extracting data..."):
            client = HarvardAPIClient()
            raw_data = extract_artifacts(client, num_pages)
            st.success(f"Extracted {len(raw_data)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("ETL Complete!")
    
    # Analytics queries
    st.header("Analytics Queries")
    
    query_options = {
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
            ORDER BY count DESC
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Media Availability": """
            SELECT 
                CASE WHEN media_id IS NULL THEN 'No Media' ELSE 'Has Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia med ON m.id = med.artifact_id
            GROUP BY media_status
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as frequency
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Analytics Query", list(query_options.keys()))
    
    if st.button("Execute Query"):
        conn = create_connection()
        df = pd.read_sql(query_options[selected_query], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Analytics Queries

### Complex Analytics Examples

```python
# Artifacts with multiple images
query_multiple_images = """
SELECT m.id, m.title, COUNT(med.id) as image_count
FROM artifactmetadata m
JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY m.id, m.title
HAVING image_count > 1
ORDER BY image_count DESC
"""

# Color diversity analysis
query_color_diversity = """
SELECT m.culture, COUNT(DISTINCT c.color) as unique_colors
FROM artifactmetadata m
JOIN artifactcolors c ON m.id = c.artifact_id
WHERE m.culture IS NOT NULL
GROUP BY m.culture
HAVING unique_colors >= 5
ORDER BY unique_colors DESC
"""

# Medium and classification cross-analysis
query_medium_class = """
SELECT classification, medium, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND medium IS NOT NULL
GROUP BY classification, medium
ORDER BY count DESC
LIMIT 20
"""

# Temporal analysis
query_temporal = """
SELECT 
    century,
    department,
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL AND department IS NOT NULL
GROUP BY century, department
ORDER BY century, artifact_count DESC
"""
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.exceptions import HTTPError

def fetch_with_retry(client, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.get_objects(page=page)
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
# Connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return db_pool.get_connection()
```

### Handling Null Values
```python
# Safe data extraction
def safe_get(obj, key, default=''):
    value = obj.get(key, default)
    return value if value is not None else default
```

### Large Dataset Processing
```python
# Use chunking for large datasets
def load_in_chunks(df, table_name, chunk_size=1000):
    conn = create_connection()
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        chunk.to_sql(table_name, conn, if_exists='append', index=False)
        print(f"Loaded chunk {i//chunk_size + 1}")
    conn.close()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

The dashboard provides an interactive interface to run ETL pipelines, execute SQL analytics queries, and visualize museum artifact data in real-time.
