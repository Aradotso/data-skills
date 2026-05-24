---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline for Harvard Art Museums API with SQL analytics and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts
  - fetch Harvard Art Museums API data
  - create analytics dashboard with artifact data
  - set up museum data warehouse with SQL
  - visualize art collection metadata
  - process Harvard artifacts into relational database
  - query and analyze museum collections data
  - build Streamlit app for art analytics
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering pipeline that:
- Extracts artifact data from Harvard Art Museums API
- Transforms nested JSON into normalized relational schema
- Loads data into MySQL/TiDB Cloud
- Runs analytical SQL queries
- Visualizes insights via Streamlit dashboards

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements (typical setup):**
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

Create a `.env` file or configure via Streamlit secrets:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Connection
DB_HOST=your_mysql_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Streamlit Secrets (`.streamlit/secrets.toml`)

```toml
HARVARD_API_KEY = "your_api_key_here"
DB_HOST = "your_mysql_host"
DB_PORT = 3306
DB_USER = "your_db_user"
DB_PASSWORD = "your_db_password"
DB_NAME = "harvard_artifacts"
```

### Get Harvard API Key

Register at: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform

## Database Schema

The ETL pipeline creates three main tables:

### artifactmetadata
```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    description TEXT,
    url VARCHAR(500)
);
```

### artifactmedia
```sql
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    base_url VARCHAR(500),
    thumbnail_url VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### artifactcolors
```sql
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    color_hex VARCHAR(10),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Functions

### 1. Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data.get("records", [])


def collect_multiple_pages(api_key, num_pages=5):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        artifacts = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(artifacts)
        
    return all_artifacts
```

### 2. Transform JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform raw artifact JSON into normalized dataframes
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            "id": artifact.get("id"),
            "title": artifact.get("title", "")[:500],
            "culture": artifact.get("culture", "")[:200],
            "period": artifact.get("period", "")[:200],
            "century": artifact.get("century", "")[:100],
            "classification": artifact.get("classification", "")[:200],
            "department": artifact.get("department", "")[:200],
            "dated": artifact.get("dated", "")[:200],
            "accession_number": artifact.get("accessionNumber", "")[:100],
            "description": artifact.get("description", ""),
            "url": artifact.get("url", "")[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media
        for img in artifact.get("images", []):
            media = {
                "artifact_id": artifact.get("id"),
                "media_type": "image",
                "base_url": img.get("baseimageurl", ""),
                "thumbnail_url": img.get("thumbnailurl", "")
            }
            media_list.append(media)
        
        # Extract colors
        for color in artifact.get("colors", []):
            color_data = {
                "artifact_id": artifact.get("id"),
                "color": color.get("color", ""),
                "color_hex": color.get("hex", ""),
                "percentage": color.get("percent", 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection(host, port, user, password, database):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=host,
            port=port,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None


def load_metadata(connection, df):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, 
     department, dated, accession_number, description, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
        title=VALUES(title),
        culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")


def load_media(connection, df):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, media_type, base_url, thumbnail_url)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")


def load_colors(connection, df):
    """Batch insert color data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, color_hex, percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    print(f"Inserted {cursor.rowcount} color records")
```

## Complete ETL Pipeline

```python
def run_etl_pipeline():
    """Execute full ETL pipeline"""
    
    # 1. Extract
    print("Starting extraction...")
    api_key = os.getenv("HARVARD_API_KEY")
    artifacts = collect_multiple_pages(api_key, num_pages=10)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # 2. Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    
    # 3. Load
    print("Loading to database...")
    connection = get_db_connection(
        host=os.getenv("DB_HOST"),
        port=int(os.getenv("DB_PORT", 3306)),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )
    
    if connection:
        load_metadata(connection, metadata_df)
        load_media(connection, media_df)
        load_colors(connection, colors_df)
        connection.close()
        print("ETL pipeline completed successfully")
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Top 10 cultures by artifact count
query_top_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century distribution
query_century_distribution = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Most common colors in collection
query_color_analysis = """
SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 15
"""

# Artifacts with most images
query_most_images = """
SELECT a.id, a.title, COUNT(m.id) as image_count
FROM artifactmetadata a
JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.id, a.title
ORDER BY image_count DESC
LIMIT 10
"""

# Department breakdown
query_departments = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY count DESC
"""
```

### Execute Query Function

```python
def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password")
    
    # Main tabs
    tab1, tab2, tab3 = st.tabs(["📥 ETL Pipeline", "📊 Analytics", "🔍 Data Explorer"])
    
    with tab1:
        show_etl_interface()
    
    with tab2:
        show_analytics_dashboard()
    
    with tab3:
        show_data_explorer()


def show_etl_interface():
    """ETL pipeline interface"""
    st.header("ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Pages to fetch", min_value=1, max_value=50, value=5)
    
    with col2:
        if st.button("🚀 Run ETL Pipeline", type="primary"):
            with st.spinner("Running ETL pipeline..."):
                run_etl_pipeline()
                st.success("ETL completed successfully!")


def show_analytics_dashboard():
    """Analytics dashboard with visualizations"""
    st.header("Analytics Dashboard")
    
    connection = get_db_connection(
        host=st.secrets["DB_HOST"],
        port=st.secrets["DB_PORT"],
        user=st.secrets["DB_USER"],
        password=st.secrets["DB_PASSWORD"],
        database=st.secrets["DB_NAME"]
    )
    
    if connection:
        # Culture analysis
        st.subheader("Top Cultures")
        df_cultures = execute_query(connection, query_top_cultures)
        
        fig = px.bar(df_cultures, x='culture', y='artifact_count',
                     title='Artifacts by Culture')
        st.plotly_chart(fig, use_container_width=True)
        
        # Color analysis
        st.subheader("Color Distribution")
        df_colors = execute_query(connection, query_color_analysis)
        
        fig = px.pie(df_colors, names='color', values='frequency',
                     title='Most Common Colors')
        st.plotly_chart(fig, use_container_width=True)
        
        connection.close()


if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0


def incremental_etl(last_id):
    """Load only new artifacts since last_id"""
    artifacts = fetch_artifacts(api_key)
    new_artifacts = [a for a in artifacts if a["id"] > last_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        # Load to database...
```

### Error Handling & Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_run():
    """ETL with comprehensive error handling"""
    try:
        artifacts = fetch_artifacts(api_key)
        logger.info(f"Fetched {len(artifacts)} artifacts")
        
    except requests.RequestException as e:
        logger.error(f"API request failed: {e}")
        return False
        
    except Error as e:
        logger.error(f"Database error: {e}")
        return False
        
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        return False
    
    return True
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def ensure_connection(connection):
    """Check and reconnect if needed"""
    try:
        connection.ping(reconnect=True, attempts=3, delay=2)
    except Error:
        connection = get_db_connection(...)
    return connection
```

### Missing Data Handling

```python
def safe_get(data, key, default=""):
    """Safely extract nested JSON values"""
    value = data.get(key, default)
    return value if value is not None else default


# Use in transformation
metadata = {
    "culture": safe_get(artifact, "culture", "Unknown"),
    "period": safe_get(artifact, "period", "Unspecified")
}
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run ETL pipeline only
python etl_pipeline.py

# Run specific analytics
python analytics.py --query top_cultures
```
