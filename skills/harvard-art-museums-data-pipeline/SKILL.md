---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL, SQL, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with museum artifacts
  - set up analytics dashboard for Harvard API
  - implement SQL pipeline for art collection data
  - analyze Harvard museum artifacts with Python
  - build Streamlit app for art museum data
  - create data visualization for museum collections
  - design ETL workflow for artifact metadata
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a production-ready data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics through a Streamlit dashboard.

## Overview

The application implements a complete data workflow:
- **Extract**: Fetch artifact data from Harvard Art Museums API with pagination
- **Transform**: Convert nested JSON into normalized relational tables
- **Load**: Batch insert data into MySQL/TiDB Cloud
- **Analyze**: Execute 20+ predefined SQL queries
- **Visualize**: Interactive Plotly charts in Streamlit UI

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Run the Streamlit app
streamlit run app.py
```

## Configuration

### API Key Setup

Obtain a Harvard Art Museums API key from [https://www.harvardartmuseums.org/collections/api](https://www.harvardartmuseums.org/collections/api).

Store credentials securely:

```python
import os

# Set environment variables
os.environ['HARVARD_API_KEY'] = 'your_api_key_here'
os.environ['DB_HOST'] = 'your_database_host'
os.environ['DB_USER'] = 'your_db_user'
os.environ['DB_PASSWORD'] = 'your_db_password'
os.environ['DB_NAME'] = 'harvard_artifacts'
```

### Database Schema

Create these tables in your MySQL/TiDB database:

```sql
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    provenance TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## Core Components

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
        'fields': 'id,title,culture,century,classification,department,division,dated,period,technique,medium,dimensions,creditline,accessionnumber,provenance,images,colors'
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.environ.get('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get('records', [])
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
from typing import List, Dict, Tuple

def transform_artifacts(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """Transform nested JSON into normalized dataframes"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        metadata_records.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionnumber'),
            'provenance': artifact.get('provenance')
        })
        
        # Extract media
        images = artifact.get('images', [])
        for image in images:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl'),
                'media_type': image.get('format')
            })
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.environ.get('DB_HOST'),
        user=os.environ.get('DB_USER'),
        password=os.environ.get('DB_PASSWORD'),
        database=os.environ.get('DB_NAME')
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Batch insert dataframes into SQL tables"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        insert_metadata = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, department, 
         division, dated, period, technique, medium, dimensions, 
         creditline, accession_number, provenance)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(insert_metadata, df_metadata.values.tolist())
        
        # Insert media
        if not df_media.empty:
            insert_media = """
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
            """
            cursor.executemany(insert_media, df_media.values.tolist())
        
        # Insert colors
        if not df_colors.empty:
            insert_colors = """
            INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
            """
            cursor.executemany(insert_colors, df_colors.values.tolist())
        
        conn.commit()
        print(f"Successfully loaded {len(df_metadata)} artifacts")
        
    except Error as e:
        print(f"Error loading data: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. SQL Analytics Queries

```python
def execute_analytics_query(query_name: str) -> pd.DataFrame:
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE WHEN m.artifact_id IS NOT NULL THEN 'With Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.artifact_id = m.artifact_id
            GROUP BY media_status
        """,
        
        'top_departments': """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'color_distribution': """
            SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
            FROM artifactcolors
            GROUP BY color_hex
            ORDER BY usage_count DESC
            LIMIT 15
        """
    }
    
    conn = get_db_connection()
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Data Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox("Select Page", ["Data Collection", "Analytics Dashboard"])
    
    if page == "Data Collection":
        st.header("📥 Collect Artifacts from API")
        
        api_key = st.text_input("Enter Harvard API Key", type="password")
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=100, value=5)
        
        if st.button("Start ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                all_artifacts = []
                for page in range(1, num_pages + 1):
                    data = fetch_artifacts(api_key, page=page)
                    all_artifacts.extend(data.get('records', []))
                
                st.success(f"Fetched {len(all_artifacts)} artifacts")
                
                # Transform
                df_metadata, df_media, df_colors = transform_artifacts(all_artifacts)
                
                # Load
                load_to_database(df_metadata, df_media, df_colors)
                st.success("✅ ETL Pipeline completed successfully!")
    
    elif page == "Analytics Dashboard":
        st.header("📊 SQL Analytics Dashboard")
        
        query_options = {
            "Artifacts by Culture": "artifacts_by_culture",
            "Artifacts by Century": "artifacts_by_century",
            "Media Availability": "media_availability",
            "Top Departments": "top_departments",
            "Color Distribution": "color_distribution"
        }
        
        selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
        
        if st.button("Run Query"):
            df_result = execute_analytics_query(query_options[selected_query])
            
            # Display table
            st.dataframe(df_result)
            
            # Generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                            title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Workflows

### Full ETL Pipeline Execution

```python
def run_full_pipeline(api_key: str, num_pages: int = 10):
    """Execute complete ETL pipeline"""
    
    # Extract
    all_artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data.get('records', []))
    
    # Transform
    df_metadata, df_media, df_colors = transform_artifacts(all_artifacts)
    
    # Load
    load_to_database(df_metadata, df_media, df_colors)
    
    return len(all_artifacts)

# Usage
api_key = os.environ.get('HARVARD_API_KEY')
total_artifacts = run_full_pipeline(api_key, num_pages=20)
print(f"Processed {total_artifacts} artifacts")
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def verify_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_transform(artifact, field, default=None):
    """Safely extract field with default value"""
    return artifact.get(field, default) or default

# Use in transformation
metadata_records.append({
    'artifact_id': artifact.get('id'),
    'title': safe_transform(artifact, 'title', 'Untitled'),
    'culture': safe_transform(artifact, 'culture', 'Unknown')
})
```
