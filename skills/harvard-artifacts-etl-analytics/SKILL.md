---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - connect to Harvard Art Museums API and store in database
  - create analytics dashboard for art collection data
  - set up artifact data engineering pipeline with Streamlit
  - query and visualize Harvard museum artifacts
  - build SQL analytics for art museum collections
  - extract and transform art museum API data
  - create interactive dashboard for artifact metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering solution for collecting, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates production-grade ETL pipelines with:

- **API Integration**: Paginated data extraction from Harvard Art Museums API
- **ETL Pipeline**: Transform nested JSON into normalized relational tables
- **SQL Storage**: MySQL/TiDB Cloud database with proper schema design
- **Analytics**: 20+ predefined SQL queries for insights
- **Visualization**: Streamlit dashboard with Plotly charts

## Architecture Flow

```
Harvard API → Extract → Transform → Load → MySQL/TiDB → Query → Streamlit Dashboard
```

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
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=artifacts_db
```

### Database Schema

The project uses three normalized tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
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

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## API Integration Pattern

### Basic API Request with Pagination

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
        JSON response with artifact data
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
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Total pages: {data['info']['pages']}")
```

### Collecting Multiple Pages

```python
import time

def collect_all_artifacts(api_key, max_records=1000):
    """
    Collect artifacts with pagination and rate limiting
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            data = fetch_artifacts(api_key, page=page, size=size)
            artifacts = data.get('records', [])
            
            if not artifacts:
                break
                
            all_artifacts.extend(artifacts)
            print(f"Collected {len(all_artifacts)} artifacts...")
            
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts[:max_records]
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifacts_to_dataframes(artifacts):
    """
    Transform nested JSON artifacts into normalized dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_records.append(metadata)
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
            }
            media_records.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_record)
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL/TiDB connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def batch_insert_metadata(connection, metadata_df):
    """
    Batch insert artifact metadata with conflict handling
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         dated, description, technique, medium, dimensions, 
         totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title),
        culture=VALUES(culture),
        century=VALUES(century)
    """
    
    # Convert DataFrame to list of tuples
    records = metadata_df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(connection, media_df):
    """
    Batch insert media records
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
        VALUES (%s, %s, %s, %s)
    """
    
    records = media_df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century distribution
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Most common color spectrums
query_colors = """
    SELECT spectrum, COUNT(*) as usage_count, 
           AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE spectrum IS NOT NULL
    GROUP BY spectrum
    ORDER BY usage_count DESC
"""

# Department distribution
query_departments = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""

# Artifacts with most page views
query_popular = """
    SELECT title, culture, century, totalpageviews
    FROM artifactmetadata
    WHERE totalpageviews > 0
    ORDER BY totalpageviews DESC
    LIMIT 20
"""
```

### Query Execution Pattern

```python
def execute_analytics_query(connection, query):
    """
    Execute SQL query and return results as DataFrame
    """
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Error executing query: {e}")
        return pd.DataFrame()

# Usage
connection = create_database_connection()
results = execute_analytics_query(connection, query_cultures)
print(results)
```

## Streamlit Dashboard Pattern

```python
import streamlit as st
import plotly.express as px

def create_streamlit_dashboard():
    """
    Main Streamlit dashboard application
    """
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection
    connection = create_database_connection()
    
    if connection:
        st.success("✅ Database connected")
        
        # Query selector
        query_options = {
            "Top Cultures": query_cultures,
            "Century Distribution": query_century,
            "Color Analysis": query_colors,
            "Department Breakdown": query_departments,
            "Popular Artifacts": query_popular
        }
        
        selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
        
        if st.button("Run Analysis"):
            with st.spinner("Executing query..."):
                results = execute_analytics_query(
                    connection, 
                    query_options[selected_query]
                )
                
                if not results.empty:
                    st.dataframe(results)
                    
                    # Auto-generate visualization
                    if len(results.columns) >= 2:
                        fig = px.bar(
                            results.head(10),
                            x=results.columns[0],
                            y=results.columns[1],
                            title=selected_query
                        )
                        st.plotly_chart(fig, use_container_width=True)
    else:
        st.error("❌ Database connection failed")

if __name__ == "__main__":
    create_streamlit_dashboard()
```

## Complete ETL Workflow

```python
def run_complete_etl_pipeline():
    """
    Execute complete ETL pipeline from API to database
    """
    # 1. Extract
    api_key = os.getenv('HARVARD_API_KEY')
    print("Step 1: Extracting data from API...")
    artifacts = collect_all_artifacts(api_key, max_records=1000)
    
    # 2. Transform
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts_to_dataframes(artifacts)
    
    # 3. Load
    print("Step 3: Loading data to database...")
    connection = create_database_connection()
    
    if connection:
        batch_insert_metadata(connection, metadata_df)
        batch_insert_media(connection, media_df)
        batch_insert_media(connection, colors_df)  # Similar function for colors
        
        connection.close()
        print("✅ ETL pipeline completed successfully")
    else:
        print("❌ Database connection failed")

# Run pipeline
if __name__ == "__main__":
    run_complete_etl_pipeline()
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, max_retries=3):
    """
    Fetch with exponential backoff retry logic
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_database_connection():
    """
    Test database connectivity and permissions
    """
    try:
        connection = create_database_connection()
        cursor = connection.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        print(f"✅ Database connection successful: {result}")
        cursor.close()
        connection.close()
        return True
    except Error as e:
        print(f"❌ Database error: {e}")
        return False
```

### Data Validation

```python
def validate_dataframe(df, table_name):
    """
    Validate DataFrame before insertion
    """
    print(f"\nValidating {table_name}:")
    print(f"  Rows: {len(df)}")
    print(f"  Columns: {list(df.columns)}")
    print(f"  Nulls: {df.isnull().sum().sum()}")
    print(f"  Duplicates: {df.duplicated().sum()}")
    
    # Check for required fields
    if table_name == "metadata" and 'id' not in df.columns:
        raise ValueError("Missing required 'id' column")
    
    return True
```

## Common Use Cases

### 1. Daily Data Refresh

```python
import schedule

def daily_etl_job():
    """
    Scheduled daily ETL refresh
    """
    print(f"Starting daily ETL at {pd.Timestamp.now()}")
    run_complete_etl_pipeline()

schedule.every().day.at("02:00").do(daily_etl_job)
```

### 2. Incremental Updates

```python
def get_latest_artifact_id(connection):
    """
    Get the most recent artifact ID in database
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key):
    """
    Fetch only artifacts newer than what's in database
    """
    connection = create_database_connection()
    latest_id = get_latest_artifact_id(connection)
    
    # Fetch artifacts with ID > latest_id
    # Implementation depends on API filtering capabilities
```

### 3. Export Analytics Results

```python
def export_query_results(query, filename):
    """
    Export query results to CSV
    """
    connection = create_database_connection()
    df = execute_analytics_query(connection, query)
    df.to_csv(filename, index=False)
    print(f"Exported {len(df)} rows to {filename}")
```
