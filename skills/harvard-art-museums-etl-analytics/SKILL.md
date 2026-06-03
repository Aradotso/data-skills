---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit visualizations
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data analytics app with Streamlit and museum artifacts
  - extract and analyze Harvard Art Museums API data
  - set up SQL analytics for art collections data
  - visualize museum artifact data with interactive dashboards
  - implement a real-world data engineering pipeline with Python
  - query and analyze art museum metadata with SQL
  - build a museum data collection analytics application
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a production-ready data engineering workflow that:
- Extracts artifact data from the Harvard Art Museums API
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Provides SQL analytics capabilities with 20+ predefined queries
- Visualizes insights through interactive Streamlit dashboards

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Core dependencies:**
```
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

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(255),
    contact VARCHAR(255),
    description TEXT,
    provenance TEXT,
    commentary TEXT,
    accessionyear INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    imagepermissionlevel INT,
    totalimagestored INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
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
# Start the Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only get artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Collect multiple pages
def collect_all_artifacts(total_pages=5):
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        print(f"Collected page {page}/{total_pages}")
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform nested JSON into normalized dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'contact': artifact.get('contact'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'commentary': artifact.get('commentary'),
            'accessionyear': artifact.get('accessionyear'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'iiifbaseuri': artifact.get('iiifbaseuri'),
            'imagepermissionlevel': artifact.get('imagepermissionlevel'),
            'totalimagestored': len(artifact.get('images', []))
        }
        media_records.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Load transformed data into MySQL database"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (batch insert for performance)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, period, 
         technique, medium, dimensions, creditline, division, contact, description, 
         provenance, commentary, accessionyear, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri, imagepermissionlevel, totalimagestored)
        VALUES (%s, %s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        color_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(color_query, df_colors.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries

QUERIES = {
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Top Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, med.totalimagestored
        FROM artifactmetadata m
        JOIN artifactmedia med ON m.id = med.artifact_id
        WHERE med.totalimagestored > 0
        ORDER BY med.totalimagestored DESC
        LIMIT 20
    """,
    
    "Popular Artifacts by Pageviews": """
        SELECT title, culture, century, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 25
    """
}

def execute_query(query_name):
    """Execute analytical query and return results as DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = QUERIES[query_name]
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(query_name)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Controls
    st.sidebar.header("ETL Pipeline")
    if st.sidebar.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            artifacts = collect_all_artifacts(total_pages=3)
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            load_to_database(df_meta, df_media, df_colors)
            st.success("ETL completed successfully!")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the last loaded artifact ID to enable incremental loads"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        database=os.getenv('DB_NAME')
    )
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    last_id = cursor.fetchone()[0]
    connection.close()
    return last_id or 0

def incremental_fetch(last_id):
    """Fetch only new artifacts since last ETL run"""
    # Implementation to filter by ID or date
    pass
```

### Error Handling and Retry Logic

```python
import time
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_session_with_retries():
    """Create requests session with automatic retry logic"""
    session = requests.Session()
    retries = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    session.mount('https://', HTTPAdapter(max_retries=retries))
    return session
```

## Troubleshooting

**API Rate Limiting:**
- Add delays between requests: `time.sleep(0.5)`
- Reduce page size: `size=50` instead of `size=100`

**Database Connection Issues:**
- Verify firewall rules allow connections to DB_HOST
- Check database credentials in `.env`
- Ensure database exists: `CREATE DATABASE IF NOT EXISTS harvard_artifacts`

**Missing Data:**
- API returns NULL for many fields; handle with `.get()` and default values
- Use `WHERE column IS NOT NULL` in queries

**Memory Issues with Large Datasets:**
- Process in smaller batches
- Use database cursors for large result sets
- Clear DataFrames after insertion: `del df_metadata`

**Streamlit Performance:**
- Cache database connections: `@st.cache_resource`
- Cache query results: `@st.cache_data`

```python
@st.cache_data(ttl=3600)
def cached_query(query_name):
    return execute_query(query_name)
```
