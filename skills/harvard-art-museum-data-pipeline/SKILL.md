---
name: harvard-art-museum-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for museum artifacts data
  - set up analytics dashboard for Harvard art collection
  - extract and analyze Harvard museum artifacts
  - build Streamlit app with art museum data
  - query Harvard Art Museums API and store in SQL
  - visualize museum collection data with Plotly
  - design relational schema for artifact metadata
---

# Harvard Art Museum Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. Extract artifact metadata, transform nested JSON into relational tables, load into SQL databases, and create interactive analytics dashboards with Streamlit.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transform nested JSON into normalized relational tables
- **SQL Storage**: Store artifacts, media, and color data in MySQL/TiDB Cloud
- **Analytics Queries**: 20+ predefined SQL queries for insights
- **Interactive Visualization**: Streamlit dashboards with Plotly charts

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages**:
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key from: https://docs.api.harvardartmuseums.org/

### Database Setup

Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def fetch_all_artifacts(max_records=500):
    """Fetch multiple pages with pagination"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
    return all_artifacts[:max_records]
```

### 2. ETL Pipeline - Transform

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into structured DataFrames"""
    metadata_list = []
    media_list = []
    color_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media information
        for image in artifact.get('images', []):
            media_list.append({
                'artifact_id': artifact.get('id'),
                'media_url': image.get('baseimageurl'),
                'media_type': 'image'
            })
        
        # Extract color data
        for color in artifact.get('colors', []):
            color_list.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(color_list)
    )
```

### 3. ETL Pipeline - Load to SQL

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_to_sql(metadata_df, media_df, colors_df):
    """Load DataFrames to SQL database"""
    conn = create_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    try:
        # Insert metadata (with IGNORE to skip duplicates)
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT IGNORE INTO artifactmetadata 
                (id, title, culture, period, century, classification, department, dated, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, media_url, media_type)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        return True
    except Error as e:
        print(f"Error loading data: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytics Queries

```python
def execute_analytics_query(query_name):
    """Execute predefined analytics queries"""
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        'artifacts_by_department': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            GROUP BY department
            ORDER BY count DESC
        """,
        'media_availability': """
            SELECT 
                COUNT(DISTINCT m.artifact_id) as with_media,
                (SELECT COUNT(*) FROM artifactmetadata) as total,
                ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
                      (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
            FROM artifactmedia m
        """,
        'top_colors': """
            SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
            FROM artifactcolors
            GROUP BY color_hex
            ORDER BY usage_count DESC
            LIMIT 10
        """
    }
    
    conn = create_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Visualization

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    st.sidebar.header("Navigation")
    page = st.sidebar.selectbox("Choose a page", 
                                 ["Data Collection", "Analytics", "Visualizations"])
    
    if page == "Data Collection":
        st.header("Collect Artifact Data")
        
        num_records = st.number_input("Number of records to fetch", 
                                      min_value=10, max_value=1000, value=100)
        
        if st.button("Fetch Data"):
            with st.spinner("Fetching data from API..."):
                raw_data = fetch_all_artifacts(max_records=num_records)
                
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(raw_data)
                
            with st.spinner("Loading to database..."):
                success = load_to_sql(metadata_df, media_df, colors_df)
                
            if success:
                st.success(f"Successfully loaded {len(metadata_df)} artifacts!")
                st.dataframe(metadata_df.head())
    
    elif page == "Analytics":
        st.header("SQL Analytics")
        
        query_choice = st.selectbox("Select Analysis", [
            "artifacts_by_culture",
            "artifacts_by_century",
            "artifacts_by_department",
            "media_availability",
            "top_colors"
        ])
        
        if st.button("Run Query"):
            results_df = execute_analytics_query(query_choice)
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(results_df, 
                            x=results_df.columns[0], 
                            y=results_df.columns[1],
                            title=f"Analysis: {query_choice}")
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def fetch_with_rate_limit(total_records, batch_size=100, delay=1):
    """Fetch data with rate limiting to avoid API throttling"""
    all_data = []
    num_batches = (total_records // batch_size) + 1
    
    for i in range(num_batches):
        page = i + 1
        data = fetch_artifacts(page=page, size=batch_size)
        all_data.extend(data.get('records', []))
        
        if len(all_data) >= total_records:
            break
            
        time.sleep(delay)  # Rate limiting
    
    return all_data[:total_records]
```

### Incremental Loading

```python
def get_last_artifact_id():
    """Get the last artifact ID in database"""
    conn = create_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_load():
    """Only load new artifacts"""
    last_id = get_last_artifact_id()
    # Fetch only artifacts with ID > last_id
    # Implementation depends on API filtering capabilities
```

## Troubleshooting

**API Key Issues**:
```python
# Verify API key is loaded
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY not found in environment")
```

**Database Connection Errors**:
```python
# Test connection
conn = create_db_connection()
if conn and conn.is_connected():
    print("Database connected successfully")
else:
    print("Check DB credentials in .env file")
```

**Empty Results**:
- Verify API key has correct permissions
- Check if `hasimage=1` parameter is too restrictive
- Confirm database tables exist before loading

**Memory Issues with Large Datasets**:
```python
# Process in smaller chunks
def chunked_load(artifacts, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df, media_df, colors_df = transform_artifacts(chunk)
        load_to_sql(metadata_df, media_df, colors_df)
```
