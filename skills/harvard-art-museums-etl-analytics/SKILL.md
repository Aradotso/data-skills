---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data pipelines with Harvard Art Museums API, ETL processing, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up Harvard artifacts collection analytics app
  - create a data pipeline with Harvard museums API
  - analyze Harvard art collection with SQL and Python
  - build Streamlit dashboard for museum artifacts
  - extract and transform Harvard Art Museums API data
  - set up TiDB analytics for museum collections
  - visualize art museum data with Plotly
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables (artifacts, media, colors)
- Loads data into MySQL/TiDB Cloud with proper schema design
- Executes analytical SQL queries on museum collections
- Visualizes insights using Plotly charts in a Streamlit interface

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

### Environment Variables

Create a `.env` file or configure in your environment:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def create_tables():
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            classification VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            dated VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            accessionyear INT,
            technique VARCHAR(500),
            medium VARCHAR(500),
            provenance TEXT
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            renditionnumber VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
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
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key=None, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = api_key or os.getenv('HARVARD_API_KEY')
    
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(url, params=params)
        response.raise_for_status()
        data = response.json()
        return data.get('records', []), data.get('info', {})
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return [], {}

# Fetch multiple pages
def fetch_all_artifacts(total_pages=5):
    all_artifacts = []
    for page in range(1, total_pages + 1):
        artifacts, info = fetch_artifacts(page=page)
        all_artifacts.extend(artifacts)
        print(f"Fetched page {page}/{total_pages}: {len(artifacts)} artifacts")
    return all_artifacts
```

### Transform: Process JSON Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into structured dataframes"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Metadata table
        metadata = {
            'id': artifact_id,
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'dated': artifact.get('dated', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'provenance': artifact.get('provenance', '')
        }
        metadata_list.append(metadata)
        
        # Media table
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact_id,
                'baseimageurl': artifact.get('primaryimageurl', '')[:500],
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '')[:500] if artifact.get('images') else '',
                'renditionnumber': artifact.get('images', [{}])[0].get('renditionnumber', '')[:50] if artifact.get('images') else ''
            }
            media_list.append(media)
        
        # Colors table
        for color_data in artifact.get('colors', []):
            color = {
                'artifact_id': artifact_id,
                'color': color_data.get('color', '')[:50],
                'spectrum': color_data.get('spectrum', '')[:50],
                'hue': color_data.get('hue', '')[:50],
                'percent': color_data.get('percent')
            }
            colors_list.append(color)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert dataframes into MySQL"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT IGNORE INTO artifactmetadata 
                (id, title, culture, classification, period, century, dated, 
                 department, division, accessionyear, technique, medium, provenance)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, iiifbaseuri, renditionnumber)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts, {len(media_df)} media, {len(colors_df)} colors")
    except Exception as e:
        conn.rollback()
        print(f"Database error: {e}")
    finally:
        cursor.close()
        conn.close()
```

## Analytical SQL Queries

```python
# Sample analytics queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(*) as color_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY color_count DESC
    """,
    
    "Media Coverage": """
        SELECT 
            COUNT(DISTINCT a.id) as total_artifacts,
            COUNT(DISTINCT m.artifact_id) as with_images,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as coverage_percent
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "Top Techniques": """
        SELECT technique, COUNT(*) as usage_count
        FROM artifactmetadata
        WHERE technique IS NOT NULL AND technique != ''
        GROUP BY technique
        ORDER BY usage_count DESC
        LIMIT 10
    """
}

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    except Exception as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        conn.close()
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.sidebar.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            results = execute_query(query)
        
        if not results.empty:
            st.subheader(f"Results: {query_name}")
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results.columns) >= 2:
                col1, col2 = results.columns[0], results.columns[1]
                
                fig = px.bar(
                    results,
                    x=col1,
                    y=col2,
                    title=query_name,
                    labels={col1: col1.replace('_', ' ').title(), 
                           col2: col2.replace('_', ' ').title()}
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Run ETL pipeline only
python etl_pipeline.py

# Run with custom page count
python etl_pipeline.py --pages 10
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """Complete ETL workflow"""
    print("Starting ETL pipeline...")
    
    # Step 1: Create database schema
    create_tables()
    
    # Step 2: Extract
    artifacts = fetch_all_artifacts(total_pages=num_pages)
    
    # Step 3: Transform
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # Step 4: Load
    load_to_database(metadata_df, media_df, colors_df)
    
    print("ETL pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline(num_pages=5)
```

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID already loaded"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] or 0

def incremental_load():
    """Load only new artifacts"""
    max_id = get_max_artifact_id()
    artifacts, _ = fetch_artifacts()
    new_artifacts = [a for a in artifacts if a.get('id', 0) > max_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_with_rate_limit(page, delay=1):
    """Add delay between API calls"""
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Database Connection Issues
```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Missing API Key
```python
def validate_api_key():
    """Ensure API key is configured"""
    api_key = os.getenv('HARVARD_API_KEY')
    if not api_key:
        raise ValueError("HARVARD_API_KEY not found in environment variables")
    return api_key
```

### Handle NULL Values
```python
def safe_get(obj, key, default=''):
    """Safely extract values with null handling"""
    value = obj.get(key, default)
    return value if value is not None else default
```

This skill provides comprehensive guidance for building museum data pipelines with proper ETL practices, SQL analytics, and interactive visualization.
