---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, Streamlit, and SQL
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering pipeline with museum artifacts
  - fetch and analyze Harvard Art Museums API data
  - set up a Streamlit analytics dashboard for art collections
  - extract transform load museum artifact data
  - query and visualize Harvard artifacts with SQL
  - build a data pipeline with the Harvard museums API
  - create an art museums data analytics application
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution that extracts artifact data from the Harvard Art Museums API, transforms it into relational structures, loads it into SQL databases, and visualizes analytics through an interactive Streamlit dashboard. The architecture follows the pattern: **API → ETL → SQL → Analytics → Visualization**.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### API Key Setup

1. Register at [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Obtain your API key
3. Store it in the `.env` file

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    technique VARCHAR(500),
    dimensions VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key (str): Harvard API key
        page (int): Page number for pagination
        size (int): Number of records per page (max 100)
    
    Returns:
        dict: JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data['records']
print(f"Total artifacts: {data['info']['totalrecords']}")
```

### 2. ETL Pipeline

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured DataFrames for each table.
    
    Args:
        raw_data (list): List of artifact dictionaries from API
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
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
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'technique': artifact.get('technique'),
            'dimensions': artifact.get('dimensions')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if 'primaryimageurl' in artifact and artifact['primaryimageurl']:
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': artifact.get('primaryimageurl')
            }
            media_list.append(media)
        
        # Extract color information
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_entry = {
                    'artifact_id': artifact.get('id'),
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

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection using environment variables."""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        port=int(os.getenv('DB_PORT', 3306))
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed DataFrames into SQL database.
    
    Args:
        metadata_df (pd.DataFrame): Artifact metadata
        media_df (pd.DataFrame): Media information
        colors_df (pd.DataFrame): Color data
    """
    connection = get_db_connection()
    cursor = connection.cursor()
    
    try:
        # Insert metadata (using IGNORE to skip duplicates)
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT IGNORE INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, dated, description, technique, dimensions)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, media_type, baseimageurl)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        connection.rollback()
    finally:
        cursor.close()
        connection.close()
```

### 4. Analytics Queries

```python
def execute_analytics_query(query):
    """
    Execute analytical SQL query and return results as DataFrame.
    
    Args:
        query (str): SQL query string
    
    Returns:
        pd.DataFrame: Query results
    """
    connection = get_db_connection()
    try:
        df = pd.read_sql(query, connection)
        return df
    finally:
        connection.close()

# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        WHERE culture IS NOT NULL 
        GROUP BY culture 
        ORDER BY artifact_count DESC 
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
        SELECT department, COUNT(*) as total_artifacts 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency 
        FROM artifactcolors 
        GROUP BY color 
        ORDER BY frequency DESC 
        LIMIT 15
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
        FROM artifactmedia m
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE classification IS NOT NULL 
        GROUP BY classification 
        ORDER BY count DESC 
        LIMIT 10
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = ANALYTICS_QUERIES[query_name]
            results_df = execute_analytics_query(query)
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(results_df, use_container_width=True)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Section
    st.sidebar.markdown("---")
    st.sidebar.header("ETL Operations")
    
    num_pages = st.sidebar.number_input(
        "Number of pages to fetch",
        min_value=1,
        max_value=10,
        value=1
    )
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching and loading data..."):
            api_key = os.getenv('HARVARD_API_KEY')
            
            for page in range(1, num_pages + 1):
                raw_data = fetch_artifacts(api_key, page=page, size=100)
                artifacts = raw_data['records']
                
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                load_to_database(metadata_df, media_df, colors_df)
            
            st.success(f"ETL completed! Processed {num_pages} page(s)")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_fetch_artifacts(api_key, total_pages=5, delay=1):
    """Fetch multiple pages with rate limiting."""
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data['records'])
        time.sleep(delay)  # Respect API rate limits
        
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, page):
    """ETL with comprehensive error handling."""
    try:
        raw_data = fetch_artifacts(api_key, page=page)
        metadata_df, media_df, colors_df = transform_artifacts(raw_data['records'])
        load_to_database(metadata_df, media_df, colors_df)
        return True
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return False
    except Error as e:
        print(f"Database Error: {e}")
        return False
    except Exception as e:
        print(f"Unexpected Error: {e}")
        return False
```

## Troubleshooting

### API Key Issues
- Verify your API key is valid at the Harvard Art Museums portal
- Check that the environment variable is properly loaded
- Ensure no extra whitespace in the `.env` file

### Database Connection Errors
- Verify database credentials in `.env`
- Check that the database and tables exist
- Ensure network connectivity to the database host
- For TiDB Cloud, verify SSL settings if required

### Empty Results
- Check if `hasimage=1` filter is too restrictive
- Verify pagination parameters
- Examine API response structure for changes

### Performance Optimization
- Use batch inserts for large datasets
- Add database indexes on frequently queried columns
- Implement connection pooling for concurrent requests
- Cache API responses for development/testing
