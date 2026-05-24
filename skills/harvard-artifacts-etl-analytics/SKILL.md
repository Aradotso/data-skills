---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics application for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - set up analytics dashboard for museum artifacts
  - integrate Harvard Art Museums API with SQL database
  - create artifact data engineering pipeline
  - visualize museum collection data with Streamlit
  - extract and analyze Harvard museum artifacts
  - build museum data warehouse with Python
  - query and visualize art collection metadata
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ predefined analytical SQL queries
- Visualizes results through interactive Streamlit dashboards with Plotly charts

**Architecture Flow:** `API → ETL → SQL → Analytics → Visualization`

## Installation

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
export DB_NAME="your_database_name"
```

### Required Dependencies

```text
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Obtain an API key from Harvard Art Museums: https://www.harvardartmuseums.org/collections/api

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

### Table Structure

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(200),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    totalimages INT,
    totalpaginatedresults INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5):
    """Extract artifact data from Harvard Art Museums API"""
    all_artifacts = []
    base_url = "https://api.harvardartmuseums.org/object"
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,  # Max per page
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
            time.sleep(1)  # Rate limiting
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### Transform: Normalize Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata transformation
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
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
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'totalimages': artifact.get('totalimages', 0),
            'totalpaginatedresults': artifact.get('totalpaginatedresults', 0)
        }
        media_list.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Load dataframes into MySQL database"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, 
         period, technique, medium, dimensions, creditline, division, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, totalimages, totalpaginatedresults)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print("Data loaded successfully!")
```

## Streamlit Application

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end ETL and Analytics Application")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Home", "ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Home":
        show_home()
    elif page == "ETL Pipeline":
        show_etl_pipeline()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_etl_pipeline():
    st.header("🔄 ETL Pipeline")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_pages = st.number_input("Number of Pages to Fetch", min_value=1, max_value=20, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = fetch_artifacts(api_key, num_pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading data to database..."):
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded to database!")

if __name__ == "__main__":
    main()
```

## Analytical SQL Queries

### Common Query Patterns

```python
# Query 1: Artifact distribution by culture
query_culture = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Query 2: Artifacts by century and classification
query_century_classification = """
SELECT century, classification, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND classification IS NOT NULL
GROUP BY century, classification
ORDER BY count DESC
LIMIT 20
"""

# Query 3: Media availability analysis
query_media = """
SELECT 
    CASE 
        WHEN totalimages > 0 THEN 'Has Images'
        ELSE 'No Images'
    END as image_status,
    COUNT(*) as count
FROM artifactmedia
GROUP BY image_status
"""

# Query 4: Color distribution across artifacts
query_colors = """
SELECT spectrum, COUNT(DISTINCT artifact_id) as artifact_count
FROM artifactcolors
GROUP BY spectrum
ORDER BY artifact_count DESC
"""

# Query 5: Department-wise artifact count
query_department = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY count DESC
"""
```

### Execute and Visualize Queries

```python
def execute_query_and_visualize(query, chart_type='bar'):
    """Execute SQL query and create visualization"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    
    st.dataframe(df)
    
    if len(df) > 0 and len(df.columns) >= 2:
        x_col = df.columns[0]
        y_col = df.columns[1]
        
        if chart_type == 'bar':
            fig = px.bar(df, x=x_col, y=y_col, title="Query Results")
        elif chart_type == 'pie':
            fig = px.pie(df, names=x_col, values=y_col, title="Query Results")
        
        st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(data_list, query, batch_size=1000):
    """Insert data in batches for performance"""
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    for i in range(0, len(data_list), batch_size):
        batch = data_list[i:i+batch_size]
        cursor.executemany(query, batch)
        conn.commit()
        print(f"Inserted batch {i//batch_size + 1}")
    
    cursor.close()
    conn.close()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_fetch(url, params, max_retries=3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.error(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

## Troubleshooting

### API Rate Limiting
- Add `time.sleep(1)` between requests
- Reduce page size or number of pages
- Monitor API quota usage

### Database Connection Issues
- Verify credentials in environment variables
- Check database server is running
- Ensure firewall rules allow connection
- Test connection with simple query first

### Memory Issues with Large Datasets
- Process data in smaller batches
- Use database pagination
- Clear DataFrames after loading: `del df; gc.collect()`

### Streamlit Caching
```python
@st.cache_data
def load_cached_data(query):
    """Cache query results to improve performance"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

This skill provides everything needed to build, run, and extend the Harvard Artifacts ETL and analytics pipeline.
