---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline and analytics app using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with museum artifacts
  - set up Harvard Art Museums API integration
  - implement artifact collection data pipeline
  - build analytics dashboard for museum data
  - create Streamlit app for art collection analytics
  - extract and analyze Harvard museum artifacts
  - design SQL schema for art collection data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application built on the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What It Does

The application:
- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into relational database schema
- **Loads** structured data into MySQL/TiDB Cloud
- **Analyzes** data through 20+ predefined SQL queries
- **Visualizes** insights using Plotly in an interactive Streamlit dashboard

## Architecture

```
Harvard API → ETL Pipeline → SQL Database → Analytics Queries → Streamlit Dashboard
```

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

### Dependencies

```python
# requirements.txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("HARVARD_API_KEY")
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

connection = mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_year INT,
    dimensions VARCHAR(500),
    url TEXT
);

-- Artifact Media Table
CREATE TABLE IF NOT EXISTS artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl TEXT,
    format VARCHAR(100),
    description TEXT,
    technique VARCHAR(200),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors Table
CREATE TABLE IF NOT EXISTS artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """Extract artifacts from Harvard Art Museums API with pagination."""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(BASE_URL, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Normalize Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data into metadata dataframe."""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accession_year': artifact.get('accessionyear'),
            'dimensions': artifact.get('dimensions', 'Unknown'),
            'url': artifact.get('url', '')
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Transform artifact media into dataframe."""
    media_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for image in images:
            media_records.append({
                'objectid': objectid,
                'baseimageurl': image.get('baseimageurl', ''),
                'format': image.get('format', ''),
                'description': image.get('description', ''),
                'technique': image.get('technique', '')
            })
    
    return pd.DataFrame(media_records)

def transform_colors(artifacts):
    """Transform artifact color data into dataframe."""
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'objectid': objectid,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_records)
```

### Load: Insert into Database

```python
def load_metadata(df, connection):
    """Load metadata into database with batch insert."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, period, century, classification, 
     department, dated, accession_year, dimensions, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_media(df, connection):
    """Load media data into database."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (objectid, baseimageurl, format, description, technique)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_colors(df, connection):
    """Load color data into database."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
```

## Analytical SQL Queries

### Example Queries

```python
analytical_queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE culture != 'Unknown'
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 15;
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL
        GROUP BY century 
        ORDER BY count DESC;
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY artifact_count DESC;
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.objectid) as with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
            ROUND(COUNT(DISTINCT m.objectid) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as percent_with_media
        FROM artifactmedia m;
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10;
    """,
    
    "Artifacts with Most Images": """
        SELECT a.objectid, a.title, COUNT(m.mediaid) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY a.objectid, a.title
        ORDER BY image_count DESC
        LIMIT 10;
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_coverage
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY count DESC;
    """
}
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Sidebar for ETL controls
    with st.sidebar:
        st.header("ETL Pipeline")
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_artifacts(API_KEY, num_pages=5)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                df_metadata = transform_metadata(artifacts)
                df_media = transform_media(artifacts)
                df_colors = transform_colors(artifacts)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                connection = mysql.connector.connect(**db_config)
                load_metadata(df_metadata, connection)
                load_media(df_media, connection)
                load_colors(df_colors, connection)
                connection.close()
                st.success("Data loaded to database")
    
    # Main analytics section
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(analytical_queries.keys()))
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        df_result = pd.read_sql(analytical_queries[query_name], connection)
        connection.close()
        
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Visualization Components

```python
def create_bar_chart(df, x_col, y_col, title):
    """Create interactive bar chart with Plotly."""
    fig = px.bar(df, x=x_col, y=y_col, title=title,
                 color=y_col, color_continuous_scale='Viridis')
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_pie_chart(df, names_col, values_col, title):
    """Create pie chart for distribution analysis."""
    fig = px.pie(df, names=names_col, values=values_col, title=title)
    return fig

def create_treemap(df, path_col, values_col, title):
    """Create treemap for hierarchical data."""
    fig = px.treemap(df, path=[path_col], values=values_col, title=title)
    return fig
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(api_key, db_config, num_pages=10):
    """Execute complete ETL pipeline."""
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(api_key, num_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_metadata(artifacts)
    df_media = transform_media(artifacts)
    df_colors = transform_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    connection = mysql.connector.connect(**db_config)
    
    load_metadata(df_metadata, connection)
    load_media(df_media, connection)
    load_colors(df_colors, connection)
    
    connection.close()
    print(f"ETL complete. Processed {len(artifacts)} artifacts.")
    
    return {
        'artifacts': len(artifacts),
        'media_records': len(df_media),
        'color_records': len(df_colors)
    }
```

### Query Execution Helper

```python
def execute_analytical_query(query_name, db_config):
    """Execute an analytical query and return results."""
    if query_name not in analytical_queries:
        raise ValueError(f"Unknown query: {query_name}")
    
    connection = mysql.connector.connect(**db_config)
    df = pd.read_sql(analytical_queries[query_name], connection)
    connection.close()
    
    return df
```

## Troubleshooting

### API Rate Limiting

```python
# Add exponential backoff for API calls
import time
from requests.exceptions import HTTPError

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with retry logic."""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Too many requests
                wait_time = (2 ** attempt)
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def get_db_connection(db_config, max_retries=3):
    """Get database connection with retry logic."""
    for attempt in range(max_retries):
        try:
            connection = mysql.connector.connect(**db_config)
            return connection
        except mysql.connector.Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2)
            else:
                raise
```

### Handling Missing Data

```python
def safe_get(data, key, default='Unknown'):
    """Safely get value from dictionary with fallback."""
    value = data.get(key, default)
    return value if value is not None else default

def clean_dataframe(df):
    """Clean dataframe by handling nulls and duplicates."""
    df = df.fillna('Unknown')
    df = df.drop_duplicates()
    return df
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(artifacts, batch_size=1000):
    """Process artifacts in batches to manage memory."""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        
        df_metadata = transform_metadata(batch)
        df_media = transform_media(batch)
        df_colors = transform_colors(batch)
        
        yield df_metadata, df_media, df_colors
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Execute specific analytical query
python run_query.py --query "Artifacts by Culture"
```

This skill enables AI agents to implement complete data engineering workflows using the Harvard Art Museums API, from extraction through analytics and visualization.
