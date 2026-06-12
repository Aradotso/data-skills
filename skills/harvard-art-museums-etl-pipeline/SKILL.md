---
name: harvard-art-museums-etl-pipeline
description: Build end-to-end ETL pipelines using the Harvard Art Museums API with SQL analytics and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - fetch and analyze Harvard Art Museums artifacts
  - create a data engineering project with museum API
  - set up SQL analytics for art collection data
  - visualize Harvard museum data with Streamlit
  - implement artifact metadata extraction pipeline
  - query Harvard Art Museums API with pagination
  - design relational database for museum artifacts
---

# Harvard Art Museums ETL Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data pipeline that demonstrates real-world ETL processes using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases, and provides interactive analytics dashboards using Streamlit.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```python
# requirements.txt typically includes:
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

1. Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Configure environment variables:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the required tables in MySQL/TiDB:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    url TEXT,
    accessionyear INT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Usage Patterns

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
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
        raise Exception(f"API request failed: {response.status_code}")

# Usage with pagination handling
def collect_all_artifacts(max_pages=10):
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        
        print(f"Collected page {page}/{info['pages']}")
        
        if page >= info['pages']:
            break
    
    return all_artifacts
```

### 2. Data Transformation (ETL)

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational DataFrames
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
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if 'primaryimageurl' in artifact and artifact['primaryimageurl']:
            media = {
                'artifact_id': artifact.get('id'),
                'iiifbaseuri': artifact.get('primaryimageurl'),
                'height': artifact.get('height', 0),
                'width': artifact.get('width', 0)
            }
            media_list.append(media)
        
        # Extract color data
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percentage': color.get('percent')
                }
                colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(df_metadata, df_media, df_colors):
    """
    Batch insert DataFrames into SQL tables
    """
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        for _, row in df_metadata.iterrows():
            query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, technique, dated, url, accessionyear)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            query = """
            INSERT INTO artifactmedia (artifact_id, iiifbaseuri, height, width)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            query = """
            INSERT INTO artifactcolors (artifact_id, color, spectrum, percentage)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        return True
        
    except Error as e:
        print(f"Database insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. SQL Analytics Queries

```python
def execute_analytics_query(query_name):
    """
    Execute predefined analytical queries
    """
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'color_distribution': """
            SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 15
        """,
        
        'media_availability': """
            SELECT 
                COUNT(DISTINCT m.artifact_id) as with_media,
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
                ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
            FROM artifactmedia m
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    connection = get_db_connection()
    cursor = connection.cursor(dictionary=True)
    
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    
    cursor.close()
    connection.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Module",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("📥 Data Collection")
        
        num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
        
        if st.button("Fetch Artifacts"):
            with st.spinner("Fetching data from API..."):
                artifacts = collect_all_artifacts(max_pages=num_pages)
                df_meta, df_media, df_colors = transform_artifacts(artifacts)
                
                if load_to_database(df_meta, df_media, df_colors):
                    st.success(f"✅ Loaded {len(df_meta)} artifacts successfully!")
                    st.dataframe(df_meta.head(10))
    
    elif page == "SQL Analytics":
        st.header("📊 SQL Analytics")
        
        query_option = st.selectbox(
            "Select Analysis",
            ["artifacts_by_culture", "artifacts_by_century", 
             "color_distribution", "media_availability", "department_distribution"]
        )
        
        if st.button("Run Query"):
            results = execute_analytics_query(query_option)
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results) > 0 and len(results.columns) >= 2:
                fig = px.bar(results, x=results.columns[0], y=results.columns[1],
                            title=f"Analysis: {query_option}")
                st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Visualizations":
        st.header("📈 Interactive Visualizations")
        
        # Example: Color spectrum analysis
        df_colors = execute_analytics_query('color_distribution')
        
        fig = px.pie(df_colors, values='usage_count', names='color',
                     title='Color Distribution Across Artifacts')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Add delay to respect API rate limits"""
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Error Handling for ETL Pipeline

```python
def safe_etl_pipeline(max_retries=3):
    """Robust ETL with retry logic"""
    for attempt in range(max_retries):
        try:
            artifacts = collect_all_artifacts()
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            success = load_to_database(df_meta, df_media, df_colors)
            
            if success:
                return True
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(5)
    
    return False
```

## Troubleshooting

### API Key Issues
- Verify `HARVARD_API_KEY` is set in `.env`
- Check API key validity at Harvard Art Museums developer portal
- Ensure no rate limits exceeded (max 2500 requests/day)

### Database Connection Errors
- Confirm MySQL/TiDB is running
- Verify credentials in environment variables
- Check firewall rules for database port (default 3306)

### Empty Results
- Some artifacts may not have all fields (culture, colors, etc.)
- Use `hasimage=1` parameter to filter artifacts with media
- Check API response structure for nested data

### Streamlit Performance
- Use `@st.cache_data` for expensive queries
- Implement pagination for large result sets
- Consider async data loading for better UX

```python
@st.cache_data(ttl=3600)
def cached_query(query_name):
    return execute_analytics_query(query_name)
```
