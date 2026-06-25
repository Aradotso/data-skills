---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit for art collections
  - query Harvard museum artifacts database
  - set up SQL database for art museum data
  - visualize artifact analytics with Plotly
  - extract and transform Harvard API JSON data
  - implement batch inserts for artifact metadata
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms nested JSON into relational structures, loads data into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics through Streamlit dashboards with Plotly visualizations.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements typically include:**
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

Create a `.env` file or set environment variables:

```bash
# Harvard API Key (get from https://www.harvardartmuseums.org/collections/api)
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required tables in your SQL database:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(255),
    objectnumber VARCHAR(100),
    url VARCHAR(500),
    lastupdate DATETIME
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT PRIMARY KEY,
    media_type VARCHAR(50),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Key Components

### 1. API Integration

**Fetching artifacts with pagination:**

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=100, page=1)
artifacts = data.get('records', [])
```

**Handling rate limits and pagination:**

```python
import time

def fetch_all_artifacts(api_key, total_artifacts=1000):
    """Fetch multiple pages with rate limiting"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_artifacts:
        try:
            data = fetch_artifacts(api_key, size=size, page=page)
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
            
            # Rate limiting - be respectful to API
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts[:total_artifacts]
```

### 2. ETL Pipeline

**Extract and Transform:**

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract artifact metadata into dataframe"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'objectnumber': artifact.get('objectnumber'),
            'url': artifact.get('url'),
            'lastupdate': artifact.get('lastupdate')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """Extract media data from nested structure"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'media_id': image.get('imageid'),
                'media_type': 'image',
                'baseimageurl': image.get('baseimageurl'),
                'iiifbaseuri': image.get('iiifbaseuri'),
                'height': image.get('height'),
                'width': image.get('width')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color data"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'color_percent': color.get('percent')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

**Load to Database:**

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

def load_metadata(df_metadata):
    """Batch insert artifact metadata"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, classification, department, 
         technique, medium, dimensions, creditline, division, objectnumber, url, lastupdate)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), lastupdate=VALUES(lastupdate)
    """
    
    # Convert DataFrame to list of tuples
    data = df_metadata.values.tolist()
    
    try:
        cursor.executemany(insert_query, data)
        conn.commit()
        print(f"Inserted {cursor.rowcount} records into artifactmetadata")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def load_media(df_media):
    """Batch insert media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_id, media_type, baseimageurl, iiifbaseuri, height, width)
        VALUES (%s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        baseimageurl=VALUES(baseimageurl)
    """
    
    data = df_media.values.tolist()
    
    try:
        cursor.executemany(insert_query, data)
        conn.commit()
        print(f"Inserted {cursor.rowcount} records into artifactmedia")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 3. Analytics Queries

**Common analytical queries:**

```python
# Query 1: Artifact count by culture
query_by_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts with images
query_with_images = """
    SELECT 
        am.department,
        COUNT(DISTINCT am.id) as total_artifacts,
        COUNT(DISTINCT m.media_id) as artifacts_with_images
    FROM artifactmetadata am
    LEFT JOIN artifactmedia m ON am.id = m.artifact_id
    GROUP BY am.department
    ORDER BY total_artifacts DESC
"""

# Query 3: Most common colors across artifacts
query_top_colors = """
    SELECT 
        color_name,
        COUNT(*) as usage_count,
        AVG(color_percent) as avg_percent
    FROM artifactcolors
    WHERE color_name IS NOT NULL
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 4: Artifacts by century
query_by_century = """
    SELECT 
        century,
        COUNT(*) as count,
        GROUP_CONCAT(DISTINCT classification SEPARATOR ', ') as classifications
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 4. Streamlit Dashboard

**Complete Streamlit app structure:**

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar
    st.sidebar.header("Navigation")
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics":
        show_analytics()
    else:
        show_visualizations()

def show_data_collection():
    """Data collection and ETL interface"""
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    
    num_artifacts = st.slider("Number of artifacts to fetch", 
                              min_value=10, max_value=1000, value=100)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_all_artifacts(api_key, num_artifacts)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = extract_metadata(artifacts)
            df_media = extract_media(artifacts)
            df_colors = extract_colors(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            load_metadata(df_metadata)
            load_media(df_media)
            load_colors(df_colors)
            st.success("Data loaded successfully!")
        
        st.dataframe(df_metadata.head())

def show_analytics():
    """Run predefined analytics queries"""
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Artifacts with Images": query_with_images,
        "Top Colors": query_top_colors,
        "Artifacts by Century": query_by_century
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df_result = execute_query(queries[selected_query])
        
        st.subheader("Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """Advanced visualizations"""
    st.header("📈 Visualizations")
    
    # Color distribution pie chart
    df_colors = execute_query(query_top_colors)
    
    fig = go.Figure(data=[go.Pie(
        labels=df_colors['color_name'],
        values=df_colors['usage_count'],
        hole=0.3
    )])
    fig.update_layout(title="Color Distribution Across Artifacts")
    st.plotly_chart(fig, use_container_width=True)
    
    # Culture distribution bar chart
    df_culture = execute_query(query_by_culture)
    
    fig = px.bar(df_culture, x='culture', y='artifact_count',
                 title="Top 10 Cultures by Artifact Count",
                 color='artifact_count',
                 color_continuous_scale='Viridis')
    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline Function

```python
def run_full_etl(api_key, num_artifacts=500):
    """Complete ETL pipeline execution"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Step 1: Extracting data from API...")
    artifacts = fetch_all_artifacts(api_key, num_artifacts)
    
    # Transform
    print("Step 2: Transforming data...")
    df_metadata = extract_metadata(artifacts)
    df_media = extract_media(artifacts)
    df_colors = extract_colors(artifacts)
    
    # Load
    print("Step 3: Loading data to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL Pipeline completed successfully!")
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'color_count': len(df_colors)
    }
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_execution(api_key, num_artifacts):
    """ETL with comprehensive error handling"""
    try:
        artifacts = fetch_all_artifacts(api_key, num_artifacts)
        logger.info(f"Successfully fetched {len(artifacts)} artifacts")
        
        df_metadata = extract_metadata(artifacts)
        logger.info(f"Transformed {len(df_metadata)} metadata records")
        
        load_metadata(df_metadata)
        logger.info("Data loaded successfully")
        
        return True
    except requests.exceptions.RequestException as e:
        logger.error(f"API request failed: {e}")
        return False
    except Error as e:
        logger.error(f"Database error: {e}")
        return False
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return False
```

## Troubleshooting

**API Key Issues:**
```python
# Verify API key is valid
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': api_key, 'size': 1}
)
if response.status_code == 401:
    print("Invalid API key. Get one from https://www.harvardartmuseums.org/collections/api")
```

**Database Connection Issues:**
```python
# Test database connection
try:
    conn = get_db_connection()
    print("Database connection successful")
    conn.close()
except Error as e:
    print(f"Connection failed: {e}")
    print("Check DB_HOST, DB_USER, DB_PASSWORD environment variables")
```

**Empty Results:**
```python
# Check if data exists
conn = get_db_connection()
cursor = conn.cursor()
cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
count = cursor.fetchone()[0]
print(f"Total artifacts in database: {count}")
cursor.close()
conn.close()
```

**Rate Limiting:**
- Add delays between API requests: `time.sleep(0.5)`
- Implement exponential backoff for retries
- Cache API responses locally to avoid repeated calls
