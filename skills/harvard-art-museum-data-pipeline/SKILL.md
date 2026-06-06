---
name: harvard-art-museum-data-pipeline
description: Build ETL pipelines and analytics apps using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create ETL workflow for museum artifact data
  - set up Harvard art collection analytics dashboard
  - build Streamlit app for art museum data
  - query Harvard Art Museums API and store in database
  - create data engineering project with museum API
  - analyze Harvard art artifacts with SQL
  - visualize museum collection data with Plotly
---

# Harvard Art Museum Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational database tables
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Interactive Dashboard**: Streamlit-based visualization with Plotly charts
- **Database Design**: Normalized schema with artifact metadata, media, and color tables

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
# Create .env file with:
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_database_host
# DB_USER=your_database_user
# DB_PASSWORD=your_database_password
# DB_NAME=harvard_artifacts
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

### Database Configuration

The project supports MySQL and TiDB Cloud:

```python
import mysql.connector
import os

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    period VARCHAR(200),
    provenance TEXT,
    description TEXT,
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagecount INT,
    videocount INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
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
import pandas as pd
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': min(page_size, num_records - len(all_records)),
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_records.extend(records)
            
            if len(records) == 0:
                break
                
            page += 1
            time.sleep(0.5)  # Rate limiting
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_records[:num_records]
```

### Transform: Process Nested JSON

```python
def transform_artifacts(raw_data):
    """
    Transform raw API data into relational table structure
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'period': artifact.get('period'),
            'provenance': artifact.get('provenance'),
            'description': artifact.get('description'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'imagecount': artifact.get('imagecount', 0),
            'videocount': artifact.get('videocount', 0)
        }
        media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed data into SQL database
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, classification, department, 
         division, technique, medium, period, provenance, description, 
         url, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    metadata_values = metadata_df.values.tolist()
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, imagecount, videocount)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    media_values = media_df.values.tolist()
    cursor.executemany(media_query, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
    
    conn.commit()
    cursor.close()
    conn.close()
```

## Analytics Queries

### Example SQL Queries

```python
# Top 10 cultures by artifact count
query_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Media availability analysis
query_media = """
    SELECT 
        SUM(CASE WHEN imagecount > 0 THEN 1 ELSE 0 END) as with_images,
        SUM(CASE WHEN imagecount = 0 THEN 1 ELSE 0 END) as without_images,
        AVG(imagecount) as avg_images_per_artifact
    FROM artifactmedia
"""

# Top colors in collection
query_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Department distribution
query_department = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
import os

def run_query(query, db_config):
    """Execute SQL query and return DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

def create_visualization(df, x_col, y_col, title):
    """Create Plotly bar chart"""
    fig = px.bar(df, x=x_col, y=y_col, title=title)
    return fig

# Streamlit app
st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")
query_options = {
    "Top Cultures": query_culture,
    "Century Distribution": query_century,
    "Media Analysis": query_media,
    "Color Usage": query_colors,
    "Department Stats": query_department
}

selected_query = st.sidebar.selectbox("Select Analysis", list(query_options.keys()))

# Execute and display results
if st.button("Run Analysis"):
    with st.spinner("Fetching data..."):
        db_config = {
            'host': os.getenv('DB_HOST'),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
        
        df = run_query(query_options[selected_query], db_config)
        
        # Display table
        st.subheader("Results")
        st.dataframe(df)
        
        # Display visualization
        if len(df.columns) >= 2:
            fig = create_visualization(
                df, 
                df.columns[0], 
                df.columns[1],
                f"{selected_query} Analysis"
            )
            st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
from dotenv import load_dotenv
import os

# Load configuration
load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Execute ETL
raw_data = fetch_artifacts(API_KEY, num_records=500)
metadata_df, media_df, colors_df = transform_artifacts(raw_data)
load_to_database(metadata_df, media_df, colors_df, db_config)

print(f"Loaded {len(metadata_df)} artifacts successfully!")
```

## Troubleshooting

### API Rate Limiting
```python
# Add delay between requests
import time
time.sleep(0.5)  # 500ms delay

# Use exponential backoff for retries
def fetch_with_retry(url, params, max_retries=3):
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        if response.status_code == 200:
            return response
        time.sleep(2 ** attempt)
    return None
```

### Database Connection Issues
```python
# Add connection pooling
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

conn = connection_pool.get_connection()
```

### Handling Missing Data
```python
# Safe data extraction
def safe_get(data, key, default=None):
    return data.get(key, default) if data else default

metadata = {
    'id': safe_get(artifact, 'id'),
    'title': safe_get(artifact, 'title', 'Untitled'),
    'culture': safe_get(artifact, 'culture', 'Unknown')
}
```
