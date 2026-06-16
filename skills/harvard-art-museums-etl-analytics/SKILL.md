---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API, featuring ETL workflows, SQL analytics, and Streamlit visualization dashboards.
triggers:
  - build an ETL pipeline for museum art data
  - create a data analytics dashboard with Harvard Art Museums API
  - set up artifact collection data engineering workflow
  - analyze art museum data with SQL and Python
  - build a Streamlit app for art collection analytics
  - extract and transform Harvard museum API data
  - create visualizations for art artifacts data
  - design a data pipeline for museum collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

**Key capabilities:**
- Dynamic artifact data collection from Harvard Art Museums API
- ETL pipeline for transforming nested JSON to relational tables
- SQL database with normalized schema (artifacts, media, colors)
- 20+ predefined analytical SQL queries
- Interactive Streamlit dashboards with Plotly visualizations

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# MySQL or TiDB Cloud database access
```

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Running the Application

```bash
# Launch Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Project Structure

```
├── app.py                 # Main Streamlit application
├── etl_pipeline.py        # ETL logic
├── database_setup.py      # SQL schema and connections
├── analytics_queries.py   # Predefined SQL queries
├── requirements.txt       # Python dependencies
└── README.md
```

## Configuration

### API Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
import requests

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

# Test API connection
response = requests.get(
    BASE_URL,
    params={
        'apikey': API_KEY,
        'size': 10,
        'page': 1
    }
)
print(response.status_code)  # Should return 200
```

### Database Configuration

```python
import mysql.connector
import os

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Test connection
conn = get_db_connection()
cursor = conn.cursor()
cursor.execute("SELECT VERSION()")
print(cursor.fetchone())
conn.close()
```

## Database Schema

### Create Tables

```python
import mysql.connector

def create_tables(connection):
    """Create normalized database schema"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            dated VARCHAR(200),
            datebegin INT,
            dateend INT,
            description TEXT,
            dimensions VARCHAR(500),
            medium VARCHAR(500),
            technique VARCHAR(500),
            accession_number VARCHAR(100),
            provenance TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Media/images table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(1000),
            image_width INT,
            image_height INT,
            media_type VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_percent DECIMAL(5,2),
            color_name VARCHAR(100),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline

### Extract Data from API

```python
import requests
import time

def extract_artifacts(api_key, num_pages=5, page_size=100):
    """Extract artifacts from Harvard Art Museums API"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={
                'apikey': api_key,
                'size': page_size,
                'page': page,
                'hasimage': 1  # Only artifacts with images
            }
        )
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{num_pages}")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            
        # Rate limiting
        time.sleep(1)
    
    return all_artifacts
```

### Transform Data

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform nested JSON to structured dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'datebegin': artifact.get('datebegin'),
            'dateend': artifact.get('dateend'),
            'description': artifact.get('description'),
            'dimensions': artifact.get('dimensions'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'accession_number': artifact.get('accessionyear'),
            'provenance': artifact.get('provenance')
        })
        
        # Extract media/images
        if artifact.get('images'):
            for image in artifact['images']:
                media_records.append({
                    'artifact_id': artifact['id'],
                    'image_url': image.get('baseimageurl'),
                    'image_width': image.get('width'),
                    'image_height': image.get('height'),
                    'media_type': 'image'
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact['id'],
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent'),
                    'color_name': color.get('color')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load Data to SQL

```python
def load_to_database(metadata_df, media_df, colors_df, connection):
    """Batch insert data into SQL database"""
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, division, dated, datebegin, dateend, 
             description, dimensions, medium, technique, 
             accession_number, provenance)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, image_url, image_width, image_height, media_type)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percent, color_name)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

### Complete ETL Workflow

```python
def run_etl_pipeline():
    """Execute full ETL pipeline"""
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = extract_artifacts(api_key, num_pages=10)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    print(f"Transformed to {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} colors")
    
    # Load
    conn = get_db_connection()
    create_tables(conn)
    load_to_database(metadata_df, media_df, colors_df, conn)
    conn.close()
    
    print("ETL pipeline completed successfully!")
```

## Analytics Queries

### Predefined SQL Queries

```python
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Artifacts with Most Images": """
        SELECT a.title, a.culture, COUNT(m.media_id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title, a.culture
        ORDER BY image_count DESC
        LIMIT 15
    """,
    
    "Top 10 Colors Used": """
        SELECT color_name, COUNT(*) as usage_count
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Average Color Diversity per Artifact": """
        SELECT AVG(color_count) as avg_colors
        FROM (
            SELECT artifact_id, COUNT(*) as color_count
            FROM artifactcolors
            GROUP BY artifact_id
        ) as color_stats
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}
```

### Execute Queries

```python
import pandas as pd

def execute_analytics_query(query_name, connection):
    """Run analytics query and return results"""
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        return None
    
    df = pd.read_sql_query(query, connection)
    return df
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar
    st.sidebar.header("Navigation")
    page = st.sidebar.radio("Select Page", ["ETL Pipeline", "Analytics", "Visualizations"])
    
    conn = get_db_connection()
    
    if page == "ETL Pipeline":
        st.header("ETL Pipeline Management")
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                run_etl_pipeline()
            st.success("ETL completed!")
    
    elif page == "Analytics":
        st.header("SQL Analytics")
        
        query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Execute Query"):
            df = execute_analytics_query(query_name, conn)
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1], title=query_name)
                st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Visualizations":
        st.header("Data Visualizations")
        
        # Culture distribution
        df_culture = execute_analytics_query("Top 10 Cultures by Artifact Count", conn)
        fig1 = px.bar(df_culture, x='culture', y='artifact_count', 
                      title="Top 10 Cultures")
        st.plotly_chart(fig1, use_container_width=True)
        
        # Color usage
        df_colors = execute_analytics_query("Top 10 Colors Used", conn)
        fig2 = px.pie(df_colors, names='color_name', values='usage_count',
                      title="Color Distribution")
        st.plotly_chart(fig2, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Error Handling

```python
def safe_api_call(url, params, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Batch Processing

```python
def batch_insert(data, connection, batch_size=1000):
    """Insert data in batches for performance"""
    cursor = connection.cursor()
    
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        # Insert batch
        cursor.executemany("INSERT INTO ...", batch)
        connection.commit()
        print(f"Inserted batch {i // batch_size + 1}")
```

## Troubleshooting

**API Rate Limiting:**
```python
# Add delays between requests
time.sleep(1)  # 1 second between calls
```

**Database Connection Timeouts:**
```python
# Increase connection timeout
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    connection_timeout=30
)
```

**Memory Issues with Large Datasets:**
```python
# Process in chunks
chunk_size = 100
for chunk in pd.read_sql_query(query, conn, chunksize=chunk_size):
    process_chunk(chunk)
```

**Missing API Key:**
```python
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY environment variable not set")
```

## Best Practices

1. **Always use environment variables for credentials**
2. **Implement proper error handling and logging**
3. **Use batch inserts for large datasets**
4. **Add indexes on foreign keys for query performance**
5. **Cache API responses to avoid redundant calls**
6. **Validate data before database insertion**
7. **Use connection pooling for concurrent requests**
