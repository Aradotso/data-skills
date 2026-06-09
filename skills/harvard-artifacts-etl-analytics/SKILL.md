---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - set up data engineering pipeline with Harvard API
  - query and visualize art museum collection data
  - implement artifact data collection and analysis
  - build Streamlit app for museum data analytics
  - create SQL database for art collection metadata
  - analyze Harvard museum artifacts with Python
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates a complete ETL pipeline that extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through a Streamlit dashboard.

**Architecture:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
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

```env
HARVARD_API_KEY=your_api_key_here
MYSQL_HOST=your_database_host
MYSQL_USER=your_database_user
MYSQL_PASSWORD=your_database_password
MYSQL_DATABASE=harvard_artifacts
```

### Database Setup

The application uses three main tables with the following structure:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    period VARCHAR(200),
    technique VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Collection

**Fetching artifacts with pagination:**

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API"""
    artifacts = []
    page = 1
    size = 100  # Max per request
    
    while len(artifacts) < num_records:
        params = {
            'apikey': API_KEY,
            'size': min(size, num_records - len(artifacts)),
            'page': page
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if not data.get('records'):
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. ETL Pipeline

**Extract and transform artifact data:**

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into structured DataFrames"""
    
    # Metadata extraction
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
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri'),
                'primaryimageurl': artifact.get('primaryimageurl')
            }
            media_records.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

**Load data into SQL database:**

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert DataFrames into SQL database"""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, accessionyear, period, technique)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, percent)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 3. Analytics Queries

**Common analytical queries:**

```python
ANALYTICS_QUERIES = {
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
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'Has Image' 
                 ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia med ON m.id = med.artifact_id
        GROUP BY image_status
    """,
    
    "Top Colors in Collection": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        cursor.execute(ANALYTICS_QUERIES[query_name])
        columns = [desc[0] for desc in cursor.description]
        results = cursor.fetchall()
        return pd.DataFrame(results, columns=columns)
    finally:
        cursor.close()
        conn.close()
```

### 4. Streamlit Dashboard

**Main application structure:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums - Data Analytics")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "Analytics Dashboard", "Raw Data Viewer"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_raw_data_page()

def show_data_collection_page():
    """ETL page for data collection"""
    st.header("📥 Collect Artifact Data")
    
    num_records = st.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=1000,
        value=100,
        step=10
    )
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded to database!")

def show_analytics_page():
    """Analytics dashboard with visualizations"""
    st.header("📊 Analytics Dashboard")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Analysis"):
        results_df = execute_query(query_choice)
        
        # Display table
        st.dataframe(results_df)
        
        # Auto-generate visualization
        if len(results_df.columns) >= 2:
            fig = px.bar(
                results_df,
                x=results_df.columns[0],
                y=results_df.columns[1],
                title=query_choice
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(num_records=100):
    """Complete ETL workflow"""
    # Extract
    print("Extracting data from API...")
    raw_data = fetch_artifacts(num_records)
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("ETL pipeline completed successfully!")
    return metadata_df, media_df, colors_df
```

### Incremental Data Loading

```python
def get_max_artifact_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def fetch_new_artifacts_only():
    """Fetch only artifacts newer than what's in database"""
    max_id = get_max_artifact_id()
    # Implement pagination with filter for id > max_id
    # ...
```

## Troubleshooting

**API Rate Limiting:**
- Harvard API has rate limits; add delays between requests
- Use `time.sleep(0.5)` between pagination calls

**Database Connection Issues:**
- Verify environment variables are loaded: `print(os.getenv('MYSQL_HOST'))`
- Check firewall rules for TiDB Cloud or remote MySQL
- Test connection: `mysql -h HOST -u USER -p`

**Missing Data:**
- Not all artifacts have images or colors; handle null values
- Use `LEFT JOIN` for optional relationships
- Check API response structure for nested fields

**Memory Issues with Large Datasets:**
- Process data in batches (e.g., 100 records at a time)
- Use `cursor.executemany()` for bulk inserts
- Clear DataFrames after loading: `del metadata_df`

**Streamlit Caching:**
```python
@st.cache_data(ttl=3600)
def fetch_artifacts_cached(num_records):
    return fetch_artifacts(num_records)
```
