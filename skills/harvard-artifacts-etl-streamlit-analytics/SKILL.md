---
name: harvard-artifacts-etl-streamlit-analytics
description: Build end-to-end data engineering pipelines using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - show me how to create a data analytics app with Streamlit
  - help me extract and transform Harvard museum artifact data
  - build a SQL analytics dashboard for art museum data
  - create a data engineering project with API and database
  - visualize Harvard artifacts collection with Plotly
  - set up ETL workflow for museum API data
  - implement pagination and rate limiting for Harvard API
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases (MySQL/TiDB Cloud), and visualizing analytics through an interactive Streamlit dashboard. It includes 20+ predefined SQL analytics queries with auto-generated visualizations.

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

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

## Database Schema

The project uses three relational tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dated VARCHAR(255),
    period VARCHAR(255),
    objectnumber VARCHAR(100),
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    primaryimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
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

## Core ETL Pipeline

### Extract from API

```python
import requests
import pandas as pd
import time

def fetch_artifacts(api_key, total_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    pages = (total_records + page_size - 1) // page_size
    
    for page in range(1, pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
            
            # Rate limiting - be respectful to the API
            time.sleep(1)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts[:total_records]
```

### Transform Data

```python
def transform_artifacts(raw_data):
    """
    Transform raw API data into structured DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:255],
            'century': artifact.get('century', 'Unknown')[:255],
            'classification': artifact.get('classification', 'Unknown')[:255],
            'department': artifact.get('department', 'Unknown')[:255],
            'medium': artifact.get('medium', 'Unknown')[:500],
            'technique': artifact.get('technique', 'Unknown')[:500],
            'dated': artifact.get('dated', 'Unknown')[:255],
            'period': artifact.get('period', 'Unknown')[:255],
            'objectnumber': artifact.get('objectnumber', 'Unknown')[:100],
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', ''),
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'iiifbaseuri': artifact.get('iiifbaseuri', '')
        }
        media_list.append(media)
        
        # Extract color information
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_entry = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', 'Unknown'),
                    'spectrum': color.get('spectrum', 'Unknown'),
                    'hue': color.get('hue', 'Unknown'),
                    'percent': color.get('percent', 0.0)
                }
                colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load into Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
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
    Batch insert DataFrames into SQL database
    """
    connection = create_database_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         medium, technique, dated, period, objectnumber, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Load colors
        if not colors_df.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
        
    finally:
        cursor.close()
        connection.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a section",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection from API")
    
    api_key = st.text_input("Enter Harvard API Key", type="password")
    num_records = st.slider("Number of records to fetch", 10, 500, 100)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Fetching artifacts..."):
            raw_data = fetch_artifacts(api_key, num_records)
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            
            success = load_to_database(metadata_df, media_df, colors_df)
            
            if success:
                st.success(f"✅ Loaded {len(metadata_df)} artifacts successfully!")
                st.dataframe(metadata_df.head())
            else:
                st.error("❌ Failed to load data")

if __name__ == "__main__":
    main()
```

## SQL Analytics Queries

### Sample Analytics

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, 
               AVG(percent) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE 
                WHEN primaryimageurl != '' THEN 'Has Image'
                ELSE 'No Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as total,
               AVG(CHAR_LENGTH(medium)) as avg_medium_length
        FROM artifactmetadata
        WHERE classification != 'Unknown'
        GROUP BY classification
        ORDER BY total DESC
        LIMIT 20
    """
}

def execute_query(query):
    """
    Execute SQL query and return DataFrame
    """
    connection = create_database_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return None
    finally:
        connection.close()

def show_sql_analytics():
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        st.code(query, language="sql")
        
        df = execute_query(query)
        
        if df is not None and not df.empty:
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                import plotly.express as px
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
```

## Visualization Patterns

```python
import plotly.express as px
import plotly.graph_objects as go

def create_culture_distribution_chart(df):
    """
    Create interactive bar chart for culture distribution
    """
    fig = px.bar(
        df,
        x='culture',
        y='count',
        title='Artifact Distribution by Culture',
        labels={'culture': 'Culture', 'count': 'Number of Artifacts'},
        color='count',
        color_continuous_scale='Viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_pie_chart(df):
    """
    Create pie chart for color distribution
    """
    fig = px.pie(
        df,
        values='frequency',
        names='color',
        title='Color Distribution in Artifacts',
        hole=0.3
    )
    return fig

def create_timeline_chart(df):
    """
    Create timeline visualization by century
    """
    fig = px.line(
        df,
        x='century',
        y='count',
        title='Artifacts Timeline by Century',
        markers=True
    )
    return fig
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl_pipeline(api_key, num_records=100):
    """
    Execute full ETL pipeline
    """
    # Extract
    st.info("Step 1: Extracting data from API...")
    raw_data = fetch_artifacts(api_key, num_records)
    
    # Transform
    st.info("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    st.info("Step 3: Loading to database...")
    success = load_to_database(metadata_df, media_df, colors_df)
    
    return success, metadata_df, media_df, colors_df
```

### Error Handling

```python
def safe_api_call(api_key, endpoint, params):
    """
    API call with retry logic and error handling
    """
    max_retries = 3
    retry_delay = 2
    
    for attempt in range(max_retries):
        try:
            response = requests.get(endpoint, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.Timeout:
            if attempt < max_retries - 1:
                time.sleep(retry_delay)
                continue
            else:
                raise
        except requests.exceptions.RequestException as e:
            st.error(f"API Error: {e}")
            return None
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Run on specific port
streamlit run app.py --server.port 8501

# Run with custom config
streamlit run app.py --server.headless true
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
time.sleep(1)  # 1 second between API calls

# Implement exponential backoff
def exponential_backoff(attempt):
    return min(60, 2 ** attempt)
```

### Database Connection Issues
```python
# Test connection
def test_db_connection():
    try:
        connection = create_database_connection()
        if connection and connection.is_connected():
            st.success("✅ Database connected")
            connection.close()
            return True
    except Error as e:
        st.error(f"❌ Connection failed: {e}")
        return False
```

### Memory Management for Large Datasets
```python
# Process in chunks
def process_in_chunks(data, chunk_size=100):
    for i in range(0, len(data), chunk_size):
        chunk = data[i:i + chunk_size]
        yield transform_artifacts(chunk)
```

### Empty Results Handling
```python
# Validate data before processing
if not raw_data or len(raw_data) == 0:
    st.warning("No artifacts found. Try adjusting your search parameters.")
    return

# Handle missing values
df.fillna({'culture': 'Unknown', 'century': 'Unknown'}, inplace=True)
```
