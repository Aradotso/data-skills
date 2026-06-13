---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for museum data
  - show me how to use the Harvard Art Museums API
  - create a data engineering project with Streamlit
  - build analytics dashboard from API data
  - extract and load art museum data into SQL
  - visualize Harvard museum artifacts with Python
  - set up end-to-end data pipeline for museum collection
  - query and analyze art museum database
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates an end-to-end data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading it into SQL databases, and visualizing insights through interactive Streamlit dashboards.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

Obtain an API key from [Harvard Art Museums](https://harvardartmuseums.org/collections/api):

```python
# In your code or Streamlit app
import os

# Set via environment variable
API_KEY = os.getenv('HARVARD_API_KEY')

# Or configure in Streamlit secrets (.streamlit/secrets.toml)
# harvard_api_key = "your_key_here"
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}

connection = mysql.connector.connect(**db_config)
```

## Core Components

### 1. API Data Extraction

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Collect multiple pages
def collect_artifacts(api_key, num_pages=5):
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        records, info = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(records)
        print(f"Fetched page {page}/{info['pages']}")
    
    return all_artifacts
```

### 2. ETL Transform Functions

```python
def transform_artifact_metadata(artifacts):
    """
    Transform nested JSON into flat metadata structure
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_year': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image information
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color palette data
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent'),
                'hue': color.get('hue')
            }
            color_list.append(color_record)
    
    return pd.DataFrame(color_list)
```

### 3. Database Schema Setup

```sql
-- Create artifacts metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium TEXT,
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_year INT
);

-- Create media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_id INT,
    base_url VARCHAR(500),
    width INT,
    height INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Create colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(100),
    spectrum VARCHAR(100),
    percent FLOAT,
    hue VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

### 4. Load Data to SQL

```python
def load_to_sql(df, table_name, connection):
    """
    Batch insert DataFrame into SQL table
    """
    cursor = connection.cursor()
    
    # Prepare columns and placeholders
    cols = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    
    # Create INSERT query
    query = f"INSERT INTO {table_name} ({cols}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(query, data_tuples)
    connection.commit()
    
    print(f"Loaded {len(df)} records into {table_name}")

# Usage
artifacts = collect_artifacts(api_key, num_pages=3)

metadata_df = transform_artifact_metadata(artifacts)
media_df = transform_artifact_media(artifacts)
colors_df = transform_artifact_colors(artifacts)

load_to_sql(metadata_df, 'artifactmetadata', connection)
load_to_sql(media_df, 'artifactmedia', connection)
load_to_sql(colors_df, 'artifactcolors', connection)
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Navigation",
    ["Data Collection", "SQL Analytics", "Visualizations"]
)

if page == "Data Collection":
    st.header("📥 Collect Artifact Data")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_pages = st.slider("Number of pages to fetch", 1, 10, 3)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_artifacts(api_key, num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            # Transform and load
            metadata_df = transform_artifact_metadata(artifacts)
            st.dataframe(metadata_df.head())

elif page == "SQL Analytics":
    st.header("📊 SQL Query Analytics")
    
    # Predefined queries
    queries = {
        "Top 10 Cultures": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
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
        "Color Analysis": """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 15
        """,
        "Image Availability": """
            SELECT 
                CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.artifact_id = m.artifact_id
            GROUP BY status
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        cursor = connection.cursor()
        cursor.execute(queries[selected_query])
        results = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        
        df_result = pd.DataFrame(results, columns=columns)
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig, use_container_width=True)

elif page == "Visualizations":
    st.header("📈 Interactive Visualizations")
    
    # Example: Color distribution
    query = "SELECT color, SUM(percent) as total_percent FROM artifactcolors GROUP BY color ORDER BY total_percent DESC LIMIT 10"
    df = pd.read_sql(query, connection)
    
    fig = px.pie(df, names='color', values='total_percent', title='Color Distribution')
    st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """
    Fetch data with rate limiting
    """
    time.sleep(delay)
    return fetch_artifacts(api_key, page)
```

### Error Handling

```python
def safe_extract(artifact, key, default=None):
    """
    Safely extract nested values
    """
    try:
        return artifact.get(key, default)
    except (KeyError, TypeError):
        return default
```

### Incremental Loading

```python
def get_max_artifact_id(connection):
    """
    Get last loaded artifact ID for incremental updates
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def fetch_new_artifacts(api_key, last_id):
    """
    Fetch only artifacts newer than last_id
    """
    # Implement pagination with filtering logic
    pass
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

**API Rate Limits:**
- Add delays between requests (0.5-1 second)
- Use pagination responsibly

**Database Connection Issues:**
- Verify credentials in environment variables
- Check firewall rules for remote databases
- Use connection pooling for performance

**Memory Issues with Large Datasets:**
- Process data in batches
- Use chunked DataFrame operations
- Clear variables after loading

**Missing Data Fields:**
- Use safe extraction functions
- Set appropriate defaults for NULL values
- Validate data types before SQL insert
