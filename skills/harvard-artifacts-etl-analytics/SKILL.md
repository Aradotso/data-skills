---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - set up analytics dashboard for museum collection data
  - extract and transform Harvard art museum data
  - build streamlit app for artifact analytics
  - query Harvard museum API and load to SQL database
  - visualize museum collection data with plotly
  - create SQL analytics for art artifacts
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. The project extracts artifact data, transforms it into relational structures, loads it into SQL databases, and provides interactive analytics dashboards using Streamlit.

**Architecture:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    technique VARCHAR(500),
    period VARCHAR(255)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    primaryimageurl VARCHAR(1000),
    baseimageurl VARCHAR(1000),
    imagepermissionlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(100),
    percent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will open at `http://localhost:8501`

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

def fetch_all_artifacts(max_pages=10):
    """
    Fetch multiple pages of artifacts
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        artifacts = data.get('records', [])
        all_artifacts.extend(artifacts)
        
        # Check if there are more pages
        if page >= data.get('info', {}).get('pages', 0):
            break
    
    return all_artifacts
```

### Transform: Processing JSON to Relational Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into structured DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        
        # Metadata table
        metadata = {
            'objectid': objectid,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period')
        }
        metadata_list.append(metadata)
        
        # Media table
        media = {
            'objectid': objectid,
            'primaryimageurl': artifact.get('primaryimageurl'),
            'baseimageurl': artifact.get('baseimageurl'),
            'imagepermissionlevel': artifact.get('imagepermissionlevel'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        }
        media_list.append(media)
        
        # Colors table (nested array)
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_entry = {
                    'objectid': objectid,
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

### Load: Inserting into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """
    Create database connection
    """
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

def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert DataFrames into SQL tables
    """
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (objectid, title, culture, century, classification, department, dated, description, technique, period)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (objectid, primaryimageurl, baseimageurl, imagepermissionlevel, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        return True
        
    except Error as e:
        print(f"Database insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## Analytics Queries

### Sample SQL Queries for Insights

```python
def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    connection = create_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        connection.close()

# Example analytics queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as total_artifacts,
               AVG(totalpageviews) as avg_pageviews
        FROM artifactmetadata m
        LEFT JOIN artifactmedia a ON m.objectid = a.objectid
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
        LIMIT 15
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency,
               AVG(percent) as avg_percentage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Pageviews": """
        SELECT m.title, m.culture, m.century, a.totalpageviews
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.objectid = a.objectid
        WHERE a.totalpageviews IS NOT NULL
        ORDER BY a.totalpageviews DESC
        LIMIT 20
    """
}
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose Function",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        st.header("📥 Data Collection & ETL")
        
        num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching artifacts..."):
                artifacts = fetch_all_artifacts(max_pages=num_pages)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                success = load_to_database(metadata_df, media_df, colors_df)
                if success:
                    st.success("✅ ETL pipeline completed!")
                else:
                    st.error("❌ Database load failed")
    
    elif page == "SQL Analytics":
        st.header("📊 SQL Query Analytics")
        
        query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Execute Query"):
            query = ANALYTICS_QUERIES[query_name]
            st.code(query, language='sql')
            
            df = execute_query(query)
            if df is not None and not df.empty:
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df.head(20),
                        x=df.columns[0],
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Visualizations":
        st.header("📈 Data Visualizations")
        
        # Culture distribution
        df_culture = execute_query(ANALYTICS_QUERIES["Artifacts by Culture"])
        if df_culture is not None:
            fig1 = px.pie(
                df_culture,
                values='artifact_count',
                names='culture',
                title='Artifact Distribution by Culture'
            )
            st.plotly_chart(fig1, use_container_width=True)
        
        # Color analysis
        df_colors = execute_query(ANALYTICS_QUERIES["Most Common Colors"])
        if df_colors is not None:
            fig2 = px.bar(
                df_colors,
                x='color',
                y='frequency',
                color='avg_percentage',
                title='Most Common Colors in Artifacts'
            )
            st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """
    Fetch data with rate limiting to respect API limits
    """
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Error Handling in ETL

```python
def safe_etl_pipeline(max_pages=10):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        artifacts = fetch_all_artifacts(max_pages)
        if not artifacts:
            return False, "No artifacts fetched"
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        success = load_to_database(metadata_df, media_df, colors_df)
        return success, f"Processed {len(artifacts)} artifacts"
        
    except Exception as e:
        return False, f"ETL Error: {str(e)}"
```

### Incremental Data Loading

```python
def get_latest_objectid():
    """
    Get the latest objectid from database to avoid duplicates
    """
    query = "SELECT MAX(objectid) as max_id FROM artifactmetadata"
    df = execute_query(query)
    return df['max_id'].iloc[0] if df is not None else 0

def fetch_new_artifacts_only():
    """
    Fetch only artifacts newer than what's in database
    """
    latest_id = get_latest_objectid()
    # Implement filtering logic based on objectid
    pass
```

## Troubleshooting

**API Key Issues:**
```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

**Database Connection Errors:**
```python
# Test database connection
def test_db_connection():
    connection = create_db_connection()
    if connection and connection.is_connected():
        print("✅ Database connected successfully")
        connection.close()
        return True
    else:
        print("❌ Database connection failed")
        return False
```

**Empty Results:**
- Check if API key has proper permissions
- Verify `hasimage=1` parameter isn't over-filtering
- Ensure database tables exist before loading

**Memory Issues with Large Datasets:**
```python
def batch_etl_pipeline(total_pages, batch_size=5):
    """
    Process data in batches to manage memory
    """
    for i in range(0, total_pages, batch_size):
        batch_pages = min(batch_size, total_pages - i)
        artifacts = fetch_all_artifacts(max_pages=batch_pages)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Batch {i//batch_size + 1} completed")
```
