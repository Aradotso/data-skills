---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit dashboards
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for museum artifact data
  - set up SQL analytics dashboard for art collections
  - extract and transform Harvard museum API data
  - visualize art museum data with Streamlit
  - implement artifact metadata ETL pipeline
  - query and analyze museum collection data
  - build interactive dashboards for cultural artifact analytics
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build complete data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational database tables
- **SQL Analytics**: Pre-built analytical queries for cultural data insights
- **Interactive Dashboards**: Streamlit-based visualization interface with Plotly charts
- **Relational Schema**: Properly normalized tables for artifact metadata, media, and color data

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Create requirements.txt if not present
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration (MySQL/TiDB Cloud)
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### API Key Setup

1. Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Set it in your `.env` file
3. API rate limits: 2500 requests/day for free tier

### Database Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create connection to MySQL/TiDB Cloud"""
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
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    url VARCHAR(500),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_pages=5, records_per_page=100):
    """Extract artifact data from Harvard API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': records_per_page,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            print(f"Fetched page {page}: {len(artifacts)} records")
        else:
            print(f"Error on page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### Transform: Process and Normalize Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifacts into normalized metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', ''),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dimensions': artifact.get('dimensions', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'media_id': image.get('id'),
                'baseimageurl': image.get('baseimageurl', ''),
                'iiifbaseuri': image.get('iiifbaseuri', ''),
                'publiccaption': image.get('publiccaption', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color palette data"""
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### Load: Batch Insert to SQL

```python
def load_to_sql(df, table_name, connection):
    """Load DataFrame to SQL table with batch insert"""
    cursor = connection.cursor()
    
    # Get column names
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    data_tuples = [tuple(row) for row in df.values]
    
    # Batch insert
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} rows into {table_name}")
    cursor.close()

# Complete ETL workflow
def run_etl_pipeline(num_pages=5):
    """Execute complete ETL pipeline"""
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_artifacts(num_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading data to SQL...")
    connection = create_database_connection()
    
    if connection:
        load_to_sql(metadata_df, 'artifactmetadata', connection)
        load_to_sql(media_df, 'artifactmedia', connection)
        load_to_sql(colors_df, 'artifactcolors', connection)
        connection.close()
        print("ETL pipeline completed successfully!")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by Culture
QUERY_ARTIFACTS_BY_CULTURE = """
SELECT culture, COUNT(*) as artifact_count 
FROM artifactmetadata 
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture 
ORDER BY artifact_count DESC 
LIMIT 20
"""

# Query 2: Artifacts by Century
QUERY_ARTIFACTS_BY_CENTURY = """
SELECT century, COUNT(*) as count 
FROM artifactmetadata 
WHERE century IS NOT NULL 
GROUP BY century 
ORDER BY count DESC
"""

# Query 3: Media Availability
QUERY_MEDIA_AVAILABILITY = """
SELECT 
    CASE WHEN am.artifact_id IS NOT NULL THEN 'With Media' ELSE 'Without Media' END as media_status,
    COUNT(*) as count
FROM artifactmetadata a
LEFT JOIN artifactmedia am ON a.id = am.artifact_id
GROUP BY media_status
"""

# Query 4: Top Colors in Collection
QUERY_TOP_COLORS = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY frequency DESC
LIMIT 15
"""

# Query 5: Department Distribution
QUERY_DEPARTMENT_DISTRIBUTION = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""

# Query 6: Classification Types
QUERY_CLASSIFICATION_TYPES = """
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY count DESC
LIMIT 20
"""

# Query 7: Accession Year Trends
QUERY_ACCESSION_TRENDS = """
SELECT accessionyear, COUNT(*) as acquisitions
FROM artifactmetadata
WHERE accessionyear IS NOT NULL AND accessionyear > 1800
GROUP BY accessionyear
ORDER BY accessionyear
"""

def execute_query(query, connection):
    """Execute SQL query and return DataFrame"""
    try:
        df = pd.read_sql(query, connection)
        return df
    except Exception as e:
        print(f"Query error: {e}")
        return None
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        
        # ETL Controls
        st.subheader("ETL Pipeline")
        num_pages = st.number_input("Pages to fetch", min_value=1, max_value=20, value=5)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL..."):
                run_etl_pipeline(num_pages=num_pages)
                st.success("ETL completed!")
    
    # Analytics Dashboard
    st.header("📊 Analytics Dashboard")
    
    connection = create_database_connection()
    
    if connection:
        # Query selector
        queries = {
            "Artifacts by Culture": QUERY_ARTIFACTS_BY_CULTURE,
            "Artifacts by Century": QUERY_ARTIFACTS_BY_CENTURY,
            "Media Availability": QUERY_MEDIA_AVAILABILITY,
            "Top Colors": QUERY_TOP_COLORS,
            "Department Distribution": QUERY_DEPARTMENT_DISTRIBUTION,
            "Classification Types": QUERY_CLASSIFICATION_TYPES,
            "Accession Trends": QUERY_ACCESSION_TRENDS
        }
        
        selected_query = st.selectbox("Select Analysis", list(queries.keys()))
        
        if st.button("Run Query"):
            query = queries[selected_query]
            df = execute_query(query, connection)
            
            if df is not None and not df.empty:
                # Display table
                st.subheader("Query Results")
                st.dataframe(df)
                
                # Visualization
                st.subheader("Visualization")
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df, 
                        x=df.columns[0], 
                        y=df.columns[1],
                        title=selected_query
                    )
                    st.plotly_chart(fig, use_container_width=True)
        
        connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(url, params, delay=0.5):
    """Fetch data with rate limiting"""
    response = requests.get(url, params=params)
    time.sleep(delay)  # Respect API rate limits
    return response
```

### Error Handling in ETL

```python
def safe_extract(artifacts, key, default=''):
    """Safely extract nested JSON values"""
    try:
        return artifacts.get(key, default)
    except (AttributeError, KeyError):
        return default
```

### Dynamic Query Builder

```python
def build_filter_query(table, filters):
    """Build SQL query with dynamic filters"""
    query = f"SELECT * FROM {table} WHERE 1=1"
    
    for column, value in filters.items():
        if value:
            query += f" AND {column} = '{value}'"
    
    return query
```

## Troubleshooting

### API Connection Issues

```python
# Check API key validity
def validate_api_key():
    api_key = os.getenv('HARVARD_API_KEY')
    test_url = f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
    
    response = requests.get(test_url)
    if response.status_code == 200:
        print("API key is valid")
        return True
    else:
        print(f"API key error: {response.status_code}")
        return False
```

### Database Connection Errors

```python
# Test database connection
def test_db_connection():
    try:
        connection = create_database_connection()
        if connection and connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def chunked_etl(num_pages, chunk_size=2):
    """Process ETL in chunks to manage memory"""
    for start_page in range(1, num_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size, num_pages + 1)
        print(f"Processing pages {start_page} to {end_page-1}")
        
        artifacts = fetch_artifacts(
            start_page=start_page, 
            end_page=end_page
        )
        
        # Process and load chunk
        # ... transform and load logic
```

### Data Quality Checks

```python
def validate_data_quality(df, table_name):
    """Perform data quality checks"""
    print(f"\n{table_name} Quality Report:")
    print(f"Total rows: {len(df)}")
    print(f"Null values:\n{df.isnull().sum()}")
    print(f"Duplicate rows: {df.duplicated().sum()}")
    
    return df.dropna(subset=['id']).drop_duplicates(subset=['id'])
```

## Advanced Features

### Incremental ETL

```python
def incremental_load(connection):
    """Load only new artifacts since last ETL run"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    last_id = cursor.fetchone()[0] or 0
    
    # Fetch artifacts with ID > last_id
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'after': last_id
    }
    # Continue with ETL...
```

This skill provides comprehensive coverage of building production-ready data engineering pipelines with API integration, ETL workflows, SQL analytics, and interactive dashboards using the Harvard Art Museums API.
