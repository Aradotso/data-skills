---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with Harvard API
  - set up artifact collection analytics with Streamlit
  - extract and transform Harvard museum data to SQL
  - build a museum artifacts dashboard with SQL analytics
  - integrate Harvard Art Museums API with database
  - create visualizations for art collection data
  - develop end-to-end data pipeline for museum artifacts
---

# Harvard Artifacts Collection ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It implements a complete ETL pipeline that extracts artifact data, transforms it into relational tables, loads it into MySQL/TiDB, and provides interactive analytics through a Streamlit dashboard with 20+ predefined SQL queries and Plotly visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Project Structure

```
├── app.py                 # Main Streamlit application
├── etl_pipeline.py        # ETL logic for data extraction and transformation
├── db_setup.py            # Database connection and table creation
├── queries.py             # Predefined SQL analytics queries
├── requirements.txt       # Python dependencies
└── .env                   # Environment variables (not committed)
```

## Database Schema

The application creates three relational tables:

**artifactmetadata**
- `id` (PRIMARY KEY)
- `title`
- `culture`
- `century`
- `classification`
- `department`
- `dated`
- `objectnumber`

**artifactmedia**
- `id` (AUTO_INCREMENT PRIMARY KEY)
- `artifact_id` (FOREIGN KEY)
- `media_type`
- `media_url`
- `media_description`

**artifactcolors**
- `id` (AUTO_INCREMENT PRIMARY KEY)
- `artifact_id` (FOREIGN KEY)
- `color_hex`
- `color_percent`
- `color_spectrum`

## Configuration

### Environment Variables

Create a `.env` file:

```bash
HARVARD_API_KEY=your_harvard_api_key
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Connection

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

def get_db_connection():
    """Establish database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Core ETL Pipeline

### Data Extraction from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Extract artifact data from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

def extract_all_artifacts(api_key, max_pages=10):
    """Extract multiple pages of artifacts"""
    all_records = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        records = data.get('records', [])
        all_records.extend(records)
        
        if not records:
            break
    
    return all_records
```

### Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(raw_data):
    """Transform raw API data into structured metadata"""
    metadata_list = []
    
    for artifact in raw_data:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'objectnumber': artifact.get('objectnumber', 'Unknown')
        })
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(raw_data):
    """Extract media information"""
    media_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'media_url': img.get('baseimageurl'),
                'media_description': img.get('caption', '')
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(raw_data):
    """Extract color data"""
    color_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent'),
                'color_spectrum': color.get('spectrum')
            })
    
    return pd.DataFrame(color_list)
```

### Data Loading

```python
def load_to_database(df, table_name, connection):
    """Batch insert DataFrame into SQL table"""
    cursor = connection.cursor()
    
    if table_name == 'artifactmetadata':
        insert_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, dated, objectnumber)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE 
            title=VALUES(title), culture=VALUES(culture)
        """
        data_tuples = [tuple(row) for row in df.values]
    
    elif table_name == 'artifactmedia':
        insert_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, media_url, media_description)
            VALUES (%s, %s, %s, %s)
        """
        data_tuples = [tuple(row) for row in df.values]
    
    elif table_name == 'artifactcolors':
        insert_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percent, color_spectrum)
            VALUES (%s, %s, %s, %s)
        """
        data_tuples = [tuple(row) for row in df.values]
    
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    cursor.close()
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# queries.py - Sample analytics queries

ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        WHERE culture != 'Unknown'
        GROUP BY culture 
        ORDER BY artifact_count DESC 
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century != 'Unknown'
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY total_artifacts DESC
    """,
    
    "Color Analysis": """
        SELECT color_spectrum, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percent 
        FROM artifactcolors 
        GROUP BY color_spectrum 
        ORDER BY usage_count DESC
    """,
    
    "Media Availability": """
        SELECT media_type, COUNT(*) as media_count 
        FROM artifactmedia 
        GROUP BY media_type
    """,
    
    "Artifacts with Most Images": """
        SELECT am.artifact_id, am.title, COUNT(*) as image_count 
        FROM artifactmedia media
        JOIN artifactmetadata am ON media.artifact_id = am.id
        GROUP BY am.artifact_id, am.title 
        ORDER BY image_count DESC 
        LIMIT 10
    """
}
```

## Streamlit Application

### Main Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from db_setup import get_db_connection
from queries import ANALYTICS_QUERIES

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def run_query(query, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

def create_visualization(df, chart_type='bar'):
    """Generate Plotly visualization"""
    if len(df.columns) >= 2:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                     title=f"{df.columns[1]} by {df.columns[0]}")
        return fig
    return None

# Main app
st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar
st.sidebar.header("Analytics Options")
query_name = st.sidebar.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

# Execute query
if st.button("Run Analysis"):
    conn = get_db_connection()
    query = ANALYTICS_QUERIES[query_name]
    
    with st.spinner("Executing query..."):
        result_df = run_query(query, conn)
        
        st.subheader(f"Results: {query_name}")
        st.dataframe(result_df)
        
        # Visualization
        fig = create_visualization(result_df)
        if fig:
            st.plotly_chart(fig, use_container_width=True)
    
    conn.close()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
import os
from dotenv import load_dotenv

load_dotenv()

def run_etl_pipeline():
    """Execute complete ETL pipeline"""
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Extract
    print("Extracting data from API...")
    raw_data = extract_all_artifacts(api_key, max_pages=5)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(raw_data)
    media_df = transform_artifact_media(raw_data)
    colors_df = transform_artifact_colors(raw_data)
    
    # Load
    print("Loading to database...")
    conn = get_db_connection()
    load_to_database(metadata_df, 'artifactmetadata', conn)
    load_to_database(media_df, 'artifactmedia', conn)
    load_to_database(colors_df, 'artifactcolors', conn)
    conn.close()
    
    print(f"ETL Complete: {len(metadata_df)} artifacts processed")

if __name__ == "__main__":
    run_etl_pipeline()
```

## Troubleshooting

**API Rate Limiting**: Implement delays between requests
```python
import time
time.sleep(0.5)  # 500ms delay between requests
```

**Database Connection Errors**: Verify credentials and network access
```python
try:
    conn = get_db_connection()
    print("Connection successful")
except Exception as e:
    print(f"Connection failed: {e}")
```

**Missing API Key**: Ensure environment variable is set
```python
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment")
```

**Empty Results**: Check API response structure
```python
data = fetch_artifacts(api_key)
print(f"Records found: {len(data.get('records', []))}")
```
