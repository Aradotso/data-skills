---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and load Harvard Art Museums API data to SQL
  - set up museum data engineering pipeline
  - analyze Harvard Art Museums collection data
  - create artifact analytics with Streamlit
  - implement museum data warehouse with SQL
  - visualize Harvard Art Museums metadata
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- Dynamic data collection from Harvard Art Museums API with pagination and rate limiting
- ETL pipeline that transforms nested JSON into relational database tables
- SQL database schema for artifact metadata, media, and color information
- 20+ predefined analytical SQL queries for insights
- Interactive Streamlit dashboard with Plotly visualizations

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Add to your `.env` file

### Database Setup

The application supports MySQL and TiDB Cloud. Create the database:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;
```

## Database Schema

The ETL pipeline creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    rank INT,
    verificationlevel INT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    mediaid INT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(100),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, num_artifacts=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data['records'])
            
            if len(data['records']) < per_page:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_artifacts=500)
print(f"Fetched {len(artifacts)} artifacts")
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data into structured metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'dated': artifact.get('dated', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', ''),
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear'),
            'rank': artifact.get('rank'),
            'verificationlevel': artifact.get('verificationlevel'),
            'totalpageviews': artifact.get('totalpageviews'),
            'totaluniquepageviews': artifact.get('totaluniquepageviews')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image data from artifacts"""
    media_list = []
    
    for artifact in artifacts:
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media = {
                    'mediaid': image.get('imageid'),
                    'artifact_id': artifact.get('id'),
                    'baseimageurl': image.get('baseimageurl', ''),
                    'format': image.get('format', ''),
                    'height': image.get('height'),
                    'width': image.get('width')
                }
                media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color analysis data from artifacts"""
    color_list = []
    
    for artifact in artifacts:
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'hue': color.get('hue', ''),
                    'percent': color.get('percent')
                }
                color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. SQL Loading with Batch Inserts

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
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
        print(f"Error connecting to database: {e}")
        return None

def batch_insert_metadata(df, connection):
    """Batch insert artifact metadata into SQL"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, dated, classification, department, 
     division, technique, medium, dimensions, creditline, accessionyear, 
     rank, verificationlevel, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting data: {e}")
        connection.rollback()
    finally:
        cursor.close()

# Complete ETL Pipeline
def run_etl_pipeline(num_artifacts=500):
    """Execute complete ETL pipeline"""
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = fetch_artifacts(api_key, num_artifacts)
    
    # Transform
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    connection = create_database_connection()
    if connection:
        batch_insert_metadata(metadata_df, connection)
        # Similar batch inserts for media and colors
        connection.close()
        print("ETL pipeline completed successfully")
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Artifact Distribution by Culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 15
"""

# Query 2: Artifacts by Century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Top Departments by Artifact Count
query_departments = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Query 4: Media Availability Analysis
query_media = """
SELECT 
    CASE 
        WHEN mediaid IS NOT NULL THEN 'Has Media'
        ELSE 'No Media'
    END as media_status,
    COUNT(*) as count
FROM artifactmetadata m
LEFT JOIN artifactmedia a ON m.id = a.artifact_id
GROUP BY media_status
"""

# Query 5: Color Distribution Across Artifacts
query_colors = """
SELECT color, COUNT(DISTINCT artifact_id) as artifact_count
FROM artifactcolors
GROUP BY color
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 6: Most Popular Artifacts by Page Views
query_popular = """
SELECT title, culture, totalpageviews
FROM artifactmetadata
WHERE totalpageviews IS NOT NULL
ORDER BY totalpageviews DESC
LIMIT 20
"""
```

## Streamlit Dashboard

### Basic Streamlit App Structure

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.write("End-to-end ETL and Analytics Application")
    
    # Sidebar for ETL configuration
    st.sidebar.header("ETL Configuration")
    num_artifacts = st.sidebar.slider("Number of Artifacts", 100, 1000, 500)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(num_artifacts)
            st.success("ETL completed successfully!")
    
    # Analytics section
    st.header("📊 SQL Analytics")
    
    query_options = {
        "Artifacts by Culture": query_culture,
        "Artifacts by Century": query_century,
        "Department Distribution": query_departments,
        "Media Availability": query_media,
        "Color Analysis": query_colors,
        "Popular Artifacts": query_popular
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Execute Query"):
        connection = create_database_connection()
        if connection:
            df = pd.read_sql(query_options[selected_query], connection)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                            title=selected_query)
                st.plotly_chart(fig)
            
            connection.close()

if __name__ == "__main__":
    main()
```

### Run the Dashboard

```bash
streamlit run app.py
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(last_loaded_id):
    """Load only new artifacts since last ETL run"""
    query = f"""
    SELECT MAX(id) as max_id FROM artifactmetadata
    """
    # Fetch only artifacts with ID > max_id from API
    # Implement pagination starting from last known ID
```

### Pattern 2: Error Handling for API Rate Limits

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff for rate limits"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limit
            wait_time = 2 ** attempt
            time.sleep(wait_time)
        else:
            break
    
    return None
```

### Pattern 3: Data Quality Checks

```python
def validate_data_quality(df, table_name):
    """Perform data quality checks before loading"""
    checks = {
        'null_percentage': df.isnull().sum() / len(df) * 100,
        'duplicate_ids': df.duplicated(subset=['id']).sum(),
        'total_records': len(df)
    }
    
    st.write(f"Data Quality Report for {table_name}")
    st.json(checks)
    
    return checks
```

## Troubleshooting

### Issue: API Key Authentication Failure
```python
# Verify API key is loaded correctly
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
print(f"API Key loaded: {api_key is not None}")

# Test API connection
test_url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
response = requests.get(test_url)
print(f"Status: {response.status_code}")
```

### Issue: Database Connection Errors
```python
# Test database connection
try:
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    print("Database connection successful")
    connection.close()
except Error as e:
    print(f"Connection failed: {e}")
```

### Issue: Large Dataset Memory Issues
```python
# Process data in chunks
def process_in_chunks(artifacts, chunk_size=100):
    """Process large datasets in smaller chunks"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df = transform_artifact_metadata(chunk)
        # Load chunk to database
        yield metadata_df
```

### Issue: Streamlit Caching for Performance
```python
@st.cache_data(ttl=3600)
def cached_query_execution(query):
    """Cache query results for 1 hour"""
    connection = create_database_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Advanced Usage

### Custom Analytics Query Builder
```python
def build_custom_query(filters):
    """Build dynamic SQL queries based on user filters"""
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    
    if filters.get('department'):
        base_query += f" AND department = '{filters['department']}'"
    
    return base_query
```

This skill provides everything needed to build, deploy, and extend the Harvard Art Museums ETL and analytics pipeline for real-world data engineering projects.
