---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifact data
  - extract Harvard museum collection data into SQL database
  - visualize Harvard Art Museums API data with Streamlit
  - set up data engineering pipeline for art museum artifacts
  - query and analyze Harvard Art Museums collection data
  - build museum data analytics application
  - create artifact collection data pipeline
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Connects to Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts artifact metadata, transforms nested JSON into relational format, loads into SQL databases
- **SQL Analytics**: Stores data in normalized tables (artifactmetadata, artifactmedia, artifactcolors)
- **Interactive Dashboard**: Streamlit-based UI for data collection, query execution, and visualization
- **Data Visualization**: Plotly charts for analytical insights

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Set up your configuration using environment variables:

```bash
# Harvard Art Museums API Key
export HARVARD_API_KEY="your_api_key_here"

# Database Configuration (MySQL/TiDB Cloud)
export DB_HOST="your_database_host"
export DB_PORT="3306"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
```

### API Key Setup

Get your API key from [Harvard Art Museums API](https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform).

### Database Setup

Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    description TEXT,
    provenance TEXT,
    url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url TEXT,
    media_caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. ETL Pipeline

**Extract**: Fetch data from Harvard Art Museums API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Extract artifacts from Harvard Art Museums API"""
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

**Transform**: Convert nested JSON to relational format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform artifact data into normalized dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Media
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'media_url': image.get('baseimageurl'),
                'media_caption': image.get('caption')
            }
            media_list.append(media)
        
        # Colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percentage': color.get('percent')
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

**Load**: Insert into SQL database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_sql(df_metadata, df_media, df_colors):
    """Load dataframes into SQL tables"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch insert for performance)
        metadata_sql = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, period, technique, medium, description, 
             provenance, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        metadata_values = df_metadata.values.tolist()
        cursor.executemany(metadata_sql, metadata_values)
        
        # Insert media
        media_sql = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, media_url, media_caption)
            VALUES (%s, %s, %s, %s)
        """
        media_values = df_media.values.tolist()
        cursor.executemany(media_sql, media_values)
        
        # Insert colors
        colors_sql = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percentage)
            VALUES (%s, %s, %s)
        """
        colors_values = df_colors.values.tolist()
        cursor.executemany(colors_sql, colors_values)
        
        conn.commit()
        return True
        
    except Error as e:
        print(f"Database Error: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

### 2. SQL Analytics Queries

**Example Analytical Queries**:

```python
# Query: Artifacts by century
query_century = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query: Top cultures by artifact count
query_culture = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 10
"""

# Query: Color distribution
query_colors = """
    SELECT color_hex, COUNT(*) as usage_count, 
           AVG(color_percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_hex
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query: Department distribution
query_dept = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""

# Execute query
def execute_query(query):
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 3. Streamlit Dashboard

**Main Application**:

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for API configuration
with st.sidebar:
    st.header("⚙️ Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    
    st.header("📥 Data Collection")
    num_pages = st.number_input("Pages to fetch", min_value=1, max_value=20, value=5)
    
    if st.button("🔄 Run ETL Pipeline"):
        with st.spinner("Fetching data..."):
            all_artifacts = []
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(api_key, page=page, size=100)
                all_artifacts.extend(data.get('records', []))
            
            st.info(f"Fetched {len(all_artifacts)} artifacts")
            
            # Transform
            df_meta, df_media, df_colors = transform_artifacts(all_artifacts)
            
            # Load
            success = load_to_sql(df_meta, df_media, df_colors)
            
            if success:
                st.success("✅ ETL Pipeline completed successfully!")
            else:
                st.error("❌ ETL Pipeline failed")

# Analytics Section
st.header("📊 Analytics Dashboard")

# Query selector
queries = {
    "Artifacts by Century": query_century,
    "Top Cultures": query_culture,
    "Color Distribution": query_colors,
    "Department Distribution": query_dept
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    df_result = execute_query(queries[selected_query])
    
    # Display table
    st.dataframe(df_result)
    
    # Visualization
    if len(df_result) > 0:
        col_x = df_result.columns[0]
        col_y = df_result.columns[1]
        
        fig = px.bar(df_result, x=col_x, y=col_y, 
                     title=f"{selected_query} Analysis")
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate-Limited API Fetching

```python
import time

def fetch_all_artifacts_with_rate_limit(api_key, max_pages=10, delay=1):
    """Fetch artifacts with rate limiting"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(data.get('records', []))
            time.sleep(delay)  # Rate limiting
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the latest artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def incremental_load(api_key):
    """Load only new artifacts"""
    last_id = get_last_artifact_id()
    # Implement logic to fetch only artifacts with id > last_id
```

## Troubleshooting

**API Rate Limits**: Add delays between requests (1-2 seconds)

**Database Connection Errors**: Verify credentials and network access to database

**Missing Data**: Use `.get()` with defaults when parsing JSON to handle missing fields

**Memory Issues**: Process data in batches rather than loading all at once

**Foreign Key Violations**: Ensure metadata is inserted before media/colors tables

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

The dashboard provides interactive controls for ETL execution, query selection, and visualization generation.
