---
name: harvard-art-museums-data-pipeline
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for Harvard Art Museums
  - create ETL pipeline with museum artifact data
  - set up Harvard API data collection and analytics
  - implement art museum data engineering workflow
  - build Streamlit dashboard for museum collections
  - extract and analyze Harvard Art Museums data
  - create SQL analytics for museum artifacts
  - develop museum data visualization app
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build and work with end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates real-world ETL patterns, relational database design, SQL analytics, and interactive Streamlit dashboards for museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides a complete data pipeline that:

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with batch optimization
- **Analyzes** using 20+ predefined SQL queries for insights
- **Visualizes** results through interactive Plotly charts in Streamlit

## Installation

### Prerequisites

```bash
# Install required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
cp .env.example .env
# Edit .env with your credentials
```

### Environment Configuration

Create a `.env` file with the following variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key from: https://www.harvardartmuseums.org/collections/api

## Core Architecture

```
API Source (Harvard) → ETL Pipeline → SQL Database → Analytics Queries → Streamlit Dashboard
```

### Database Schema

The pipeline creates three main tables:

1. **artifactmetadata** - Core artifact information (id, title, culture, century, department)
2. **artifactmedia** - Media URLs and image availability
3. **artifactcolors** - Color palette data with percentages

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
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

# Collect multiple pages
def collect_artifacts(num_pages=5):
    all_artifacts = []
    for page in range(1, num_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        print(f"Collected page {page}: {len(records)} artifacts")
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata table
    metadata = []
    media = []
    colors = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata.append({
            'artifact_id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
        
        # Extract media
        if artifact.get('primaryimageurl'):
            media.append({
                'artifact_id': artifact_id,
                'primary_image_url': artifact.get('primaryimageurl'),
                'has_image': 1 if artifact.get('primaryimageurl') else 0
            })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors.append({
                    'artifact_id': artifact_id,
                    'color_name': color.get('color'),
                    'hex_code': color.get('hex'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media),
        pd.DataFrame(colors)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Create tables
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                artifact_id INT PRIMARY KEY,
                title TEXT,
                culture VARCHAR(255),
                century VARCHAR(100),
                department VARCHAR(255),
                classification VARCHAR(255),
                dated VARCHAR(100),
                url TEXT
            )
        """)
        
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                primary_image_url TEXT,
                has_image TINYINT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                color_name VARCHAR(100),
                hex_code VARCHAR(10),
                percentage FLOAT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        # Batch insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (artifact_id, title, culture, century, department, classification, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Batch insert media
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia (artifact_id, primary_image_url, has_image)
                VALUES (%s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Batch insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors (artifact_id, color_name, hex_code, percentage)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. SQL Analytics Queries

```python
# Example analytical queries

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
        SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Colors": """
        SELECT m.artifact_id, m.title, COUNT(c.color_name) as color_count
        FROM artifactmetadata m
        JOIN artifactcolors c ON m.artifact_id = c.artifact_id
        GROUP BY m.artifact_id, m.title
        ORDER BY color_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT has_image, COUNT(*) as count
        FROM artifactmedia
        GROUP BY has_image
    """
}

def execute_analytics_query(query_name):
    """Execute a predefined analytics query"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            results = execute_analytics_query(query_name)
            
            st.subheader(f"Results: {query_name}")
            st.dataframe(results)
            
            # Auto-generate visualization
            if len(results.columns) >= 2:
                fig = px.bar(
                    results,
                    x=results.columns[0],
                    y=results.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig)
    
    # ETL Pipeline Section
    st.sidebar.header("Data Collection")
    num_pages = st.sidebar.number_input("Pages to collect", 1, 20, 5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Collecting data from API..."):
            raw_data = collect_artifacts(num_pages)
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            
        st.success(f"✅ Loaded {len(metadata_df)} artifacts successfully!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, delay=1):
    """Fetch data with rate limiting"""
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Error Handling in ETL

```python
def safe_etl_pipeline(raw_data):
    """ETL pipeline with error handling"""
    try:
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        load_to_database(metadata_df, media_df, colors_df)
        return True, "Success"
    except Exception as e:
        return False, f"ETL failed: {str(e)}"
```

### Incremental Data Loading

```python
def get_existing_artifact_ids():
    """Get IDs of artifacts already in database"""
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    cursor.execute("SELECT artifact_id FROM artifactmetadata")
    existing_ids = {row[0] for row in cursor.fetchall()}
    connection.close()
    return existing_ids

def incremental_load(raw_data):
    """Only load new artifacts"""
    existing_ids = get_existing_artifact_ids()
    new_artifacts = [a for a in raw_data if a['id'] not in existing_ids]
    return transform_artifacts(new_artifacts)
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in `.env`
- Verify key is active at https://www.harvardartmuseums.org/collections/api
- Check for rate limit errors (429 status code)

### Database Connection Errors
- Verify all DB credentials in `.env`
- Check firewall rules for remote databases (TiDB Cloud)
- Ensure database exists before running ETL

### Empty Results
- Harvard API returns empty `records` if page exceeds total pages
- Check `info.pages` to validate page count
- Use `hasimage=1` parameter to filter artifacts with images

### Performance Optimization
- Use batch inserts (executemany) instead of single inserts
- Create indexes on foreign keys and frequently queried columns
- Limit initial data collection to 5-10 pages for testing

### Streamlit Caching

```python
@st.cache_data(ttl=3600)
def cached_query(query_name):
    """Cache query results for 1 hour"""
    return execute_analytics_query(query_name)
```
