---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, MySQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and museum data
  - query Harvard artifacts database with SQL
  - visualize museum collection data with Plotly
  - set up data engineering pipeline for art museum collections
  - analyze artifact metadata and media information
  - transform Harvard API JSON to relational database
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL analytics, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Execute analytical queries on structured museum data
- **Interactive Dashboards**: Visualize query results with Streamlit and Plotly

**Architecture**: API → ETL → SQL → Analytics → Visualization

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

### API Key Setup

Get your API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api):

```python
import os

# Store API key in environment variable
# export HARVARD_API_KEY="your_api_key_here"

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

# Database connection (use environment variables)
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

## Database Schema

Create the required tables:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    datebegin INT,
    dateend INT,
    accessionyear INT,
    technique VARCHAR(500),
    primaryimageurl TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    description TEXT,
    technique VARCHAR(255),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Examples

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    artifacts = []
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': page_size,
        'page': 1
    }
    
    while len(artifacts) < num_records:
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            artifacts.extend(records)
            
            # Check if more pages exist
            if len(records) < page_size:
                break
            
            params['page'] += 1
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into structured DataFrames"""
    
    # Metadata DataFrame
    metadata_list = []
    for artifact in artifacts:
        metadata_list.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'datebegin': artifact.get('datebegin'),
            'dateend': artifact.get('dateend'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'primaryimageurl': artifact.get('primaryimageurl')
        })
    
    metadata_df = pd.DataFrame(metadata_list)
    
    # Media DataFrame
    media_list = []
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        for img in images:
            media_list.append({
                'objectid': objectid,
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'description': img.get('description'),
                'technique': img.get('technique')
            })
    
    media_df = pd.DataFrame(media_list)
    
    # Colors DataFrame
    colors_list = []
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        for color_data in colors:
            colors_list.append({
                'objectid': objectid,
                'color': color_data.get('color'),
                'spectrum': color_data.get('spectrum'),
                'hue': color_data.get('hue'),
                'percent': color_data.get('percent')
            })
    
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Load DataFrames into MySQL database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        insert_query = """
        INSERT IGNORE INTO artifactmetadata 
        (objectid, title, culture, period, century, classification, 
         medium, department, division, dated, datebegin, dateend, 
         accessionyear, technique, primaryimageurl)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        cursor.execute(insert_query, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        insert_query = """
        INSERT INTO artifactmedia 
        (objectid, baseimageurl, format, description, technique)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(insert_query, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        insert_query = """
        INSERT INTO artifactcolors 
        (objectid, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(insert_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

## Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by Culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by Century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Media Availability
query_media_availability = """
SELECT 
    CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'No Image' END as image_status,
    COUNT(*) as count
FROM artifactmetadata
GROUP BY image_status;
"""

# Query 4: Top Colors Used
query_top_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percentage
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15;
"""

# Query 5: Department Distribution
query_departments = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC;
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        num_records = st.slider("Number of Records", 10, 500, 100)
        
        if st.button("Run ETL Pipeline"):
            run_etl_pipeline(api_key, num_records)
    
    # Main analytics section
    st.header("📊 Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Media Availability": query_media_availability,
        "Top Colors": query_top_colors,
        "Department Distribution": query_departments
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        df = execute_query(queries[selected_query])
        
        # Display results
        st.dataframe(df)
        
        # Visualize
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

def run_etl_pipeline(api_key, num_records):
    """Run complete ETL pipeline"""
    with st.spinner("Fetching data from API..."):
        artifacts = fetch_artifacts(api_key, num_records)
    
    with st.spinner("Transforming data..."):
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    with st.spinner("Loading to database..."):
        load_to_database(metadata_df, media_df, colors_df, db_config)
    
    st.success(f"Successfully loaded {num_records} artifacts!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(df, table_name, batch_size=1000):
    """Insert data in batches for better performance"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    for start in range(0, len(df), batch_size):
        batch = df.iloc[start:start + batch_size]
        # Insert batch logic here
        conn.commit()
    
    cursor.close()
    conn.close()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_call(url, params, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.error(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(2 ** attempt)
    
    return None
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests using `time.sleep(0.5)`

**Database Connection Issues**: Verify credentials and ensure MySQL service is running

**Missing Data**: Check for NULL values and use `COALESCE` or filtering in queries

**Memory Issues**: Process data in batches rather than loading entire dataset at once

**Streamlit Caching**: Use `@st.cache_data` for expensive operations to improve performance
