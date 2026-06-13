---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - set up SQL analytics for museum collection data
  - build a Streamlit dashboard for art museum data
  - extract and transform Harvard API artifact data
  - create analytics queries for museum artifacts
  - visualize Harvard Art Museums collection data
  - implement ETL workflow for cultural heritage data
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What This Project Does

The Harvard Artifacts Collection application:
- Fetches artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Provides 20+ predefined analytical SQL queries
- Visualizes results using Plotly charts in a Streamlit interface

**Architecture Flow**: API → ETL (Extract/Transform/Load) → SQL Database → Analytics Queries → Streamlit Dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key from: https://www.harvardartmuseums.org/collections/api

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Create database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Initialize database schema
def create_tables():
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dated VARCHAR(255),
            url TEXT,
            creditline TEXT,
            division VARCHAR(255),
            contact VARCHAR(255),
            totalpageviews INT,
            totaluniquepageviews INT
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl TEXT,
            primaryimageurl TEXT,
            imagepermissionlevel INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_records = []
    page = 1
    
    while len(all_records) < num_records:
        params = {
            'apikey': api_key,
            'size': min(page_size, num_records - len(all_records)),
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            if not records:
                break
                
            all_records.extend(records)
            page += 1
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching data: {e}")
            break
    
    return all_records[:num_records]
```

### Transform: Normalize JSON Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform nested JSON into normalized dataframes
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
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'contact': artifact.get('contact'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if 'primaryimageurl' in artifact or 'baseimageurl' in artifact:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
            }
            media_list.append(media)
        
        # Extract color information
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

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert dataframes into SQL database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, 
             technique, medium, dated, url, creditline, division, contact, 
             totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Exception as e:
        conn.rollback()
        print(f"Error loading data: {e}")
    finally:
        cursor.close()
        conn.close()
```

## Analytics Queries

### Sample SQL Analytics

```python
# Analytics query examples
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "top_departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'Has Image' 
                 ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "most_viewed_artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        WHERE totalpageviews > 0
        ORDER BY totalpageviews DESC
        LIMIT 10
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    try:
        cursor.execute(ANALYTICS_QUERIES[query_name])
        results = cursor.fetchall()
        return pd.DataFrame(results)
    finally:
        cursor.close()
        conn.close()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
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
    st.header("📥 Data Collection & ETL")
    
    api_key = st.text_input("Harvard API Key", type="password")
    num_records = st.number_input("Number of Records", min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            raw_data = fetch_artifacts(api_key, num_records)
            st.success(f"Fetched {len(raw_data)} records")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded to database!")

def show_analytics():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        results_df = execute_query(query_name)
        
        st.subheader("Query Results")
        st.dataframe(results_df)
        
        # Auto-generate visualization
        if len(results_df.columns) == 2:
            fig = px.bar(
                results_df,
                x=results_df.columns[0],
                y=results_df.columns[1],
                title=f"Analysis: {query_name}"
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline Execution

```python
from dotenv import load_dotenv
import os

load_dotenv()

def run_etl_pipeline():
    """Execute complete ETL workflow"""
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    raw_data = fetch_artifacts(api_key, num_records=500)
    print(f"Extracted {len(raw_data)} artifacts")
    
    # Transform
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    print(f"Transformed: {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} colors")
    
    # Load
    load_to_database(metadata_df, media_df, colors_df)
    print("ETL pipeline completed successfully")

# Run the pipeline
run_etl_pipeline()
```

### Custom Analytics Query

```python
def custom_query(sql):
    """Execute custom SQL query"""
    conn = get_db_connection()
    df = pd.read_sql(sql, conn)
    conn.close()
    return df

# Example usage
query = """
    SELECT 
        m.culture,
        COUNT(DISTINCT m.id) as artifacts,
        AVG(m.totalpageviews) as avg_views,
        COUNT(DISTINCT c.color) as unique_colors
    FROM artifactmetadata m
    LEFT JOIN artifactcolors c ON m.id = c.artifact_id
    WHERE m.culture IS NOT NULL
    GROUP BY m.culture
    HAVING artifacts > 5
    ORDER BY artifacts DESC
"""
results = custom_query(query)
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for API calls
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retries():
    session = requests.Session()
    retry = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues
```python
# Test database connection
def test_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        print("Database connection successful")
        cursor.close()
        conn.close()
        return True
    except Exception as e:
        print(f"Connection failed: {e}")
        return False
```

### Missing Data Handling
```python
# Handle null values during transformation
def safe_get(artifact, key, default=None):
    """Safely extract values with defaults"""
    value = artifact.get(key, default)
    return value if value not in [None, '', 'null'] else default

# Use in transformation
metadata = {
    'culture': safe_get(artifact, 'culture', 'Unknown'),
    'period': safe_get(artifact, 'period', 'Unspecified')
}
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline only
python etl_pipeline.py

# Execute specific analytics
python analytics.py --query artifacts_by_culture
```
