---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - fetch and analyze Harvard Art Museums collection
  - create a data engineering pipeline with SQL analytics
  - visualize museum artifacts data with Streamlit
  - extract transform load Harvard API data
  - build analytics dashboard for art collections
  - process Harvard Art Museums metadata
  - create relational database from museum API
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a production-grade ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into SQL databases (MySQL/TiDB), and provides interactive analytics dashboards using Streamlit.

## What This Project Does

- **API Integration**: Fetches artifact metadata, media details, and color information from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational database schema
- **SQL Storage**: Creates and manages three related tables (artifactmetadata, artifactmedia, artifactcolors)
- **Analytics**: Provides 20+ predefined SQL queries for insights into artifact distributions, media availability, and cultural patterns
- **Visualization**: Interactive Plotly charts rendered through Streamlit interface

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
MYSQL_HOST=your_database_host
MYSQL_USER=your_database_user
MYSQL_PASSWORD=your_database_password
MYSQL_DATABASE=harvard_artifacts
```

### API Key Setup

Obtain a free API key from Harvard Art Museums:
1. Visit https://harvardartmuseums.org/collections/api
2. Register for an API key
3. Add to your `.env` file

### Database Setup

The application supports MySQL and TiDB Cloud. Ensure your database is accessible and credentials are configured.

## Project Structure

```
├── app.py                 # Main Streamlit application
├── etl_pipeline.py        # ETL logic for data extraction and transformation
├── sql_queries.py         # Predefined analytical SQL queries
├── database.py            # Database connection and operations
├── config.py              # Configuration management
└── requirements.txt       # Python dependencies
```

## Core Components

### 1. ETL Pipeline

**Extract**: Fetch data from Harvard API with pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    artifacts = []
    pages = (num_records // page_size) + (1 if num_records % page_size > 0 else 0)
    
    for page in range(1, pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

**Transform**: Convert nested JSON to flat relational structure

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform artifacts into metadata table structure
    """
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'provenance': artifact.get('provenance'),
            'description': artifact.get('description')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_artifact_media(artifacts):
    """
    Extract media information from artifacts
    """
    media_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for image in images:
            record = {
                'objectid': objectid,
                'imageid': image.get('imageid'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'width': image.get('width'),
                'height': image.get('height')
            }
            media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """
    Extract color palette data from artifacts
    """
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

**Load**: Batch insert into SQL database

```python
import mysql.connector
from mysql.connector import Error
import os

def get_db_connection():
    """
    Create database connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('MYSQL_HOST'),
            user=os.getenv('MYSQL_USER'),
            password=os.getenv('MYSQL_PASSWORD'),
            database=os.getenv('MYSQL_DATABASE')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def create_tables(connection):
    """
    Create artifact tables if they don't exist
    """
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            technique TEXT,
            medium TEXT,
            dated VARCHAR(255),
            provenance TEXT,
            description TEXT
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl TEXT,
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_dataframe_to_sql(df, table_name, connection):
    """
    Batch insert DataFrame into SQL table
    """
    cursor = connection.cursor()
    
    # Prepare insert statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    # Batch insert
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(data)} records into {table_name}")
```

### 2. Analytics Queries

Example analytical SQL queries:

```python
# Distribution by culture
CULTURE_DISTRIBUTION = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 20
"""

# Media availability analysis
MEDIA_AVAILABILITY = """
    SELECT 
        m.department,
        COUNT(DISTINCT m.objectid) as total_artifacts,
        COUNT(DISTINCT med.objectid) as artifacts_with_media,
        ROUND(COUNT(DISTINCT med.objectid) * 100.0 / COUNT(DISTINCT m.objectid), 2) as media_percentage
    FROM artifactmetadata m
    LEFT JOIN artifactmedia med ON m.objectid = med.objectid
    GROUP BY m.department
    ORDER BY media_percentage DESC
"""

# Color usage patterns
COLOR_USAGE = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percentage
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 15
"""

# Century distribution
CENTURY_DISTRIBUTION = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY artifact_count DESC
"""

def execute_query(connection, query):
    """
    Execute SQL query and return results as DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 3. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("ETL Configuration")
    num_records = st.sidebar.number_input("Number of records to fetch", 
                                          min_value=10, 
                                          max_value=1000, 
                                          value=100)
    
    # ETL Pipeline execution
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df = transform_artifact_metadata(artifacts)
            media_df = transform_artifact_media(artifacts)
            colors_df = transform_artifact_colors(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            connection = get_db_connection()
            create_tables(connection)
            load_dataframe_to_sql(metadata_df, 'artifactmetadata', connection)
            load_dataframe_to_sql(media_df, 'artifactmedia', connection)
            load_dataframe_to_sql(colors_df, 'artifactcolors', connection)
            connection.close()
            st.success("Data loaded to database")
    
    # Analytics section
    st.header("Analytics Queries")
    
    query_options = {
        "Culture Distribution": CULTURE_DISTRIBUTION,
        "Media Availability by Department": MEDIA_AVAILABILITY,
        "Color Usage Patterns": COLOR_USAGE,
        "Century Distribution": CENTURY_DISTRIBUTION
    }
    
    selected_query = st.selectbox("Select Query", list(query_options.keys()))
    
    if st.button("Execute Query"):
        connection = get_db_connection()
        results_df = execute_query(connection, query_options[selected_query])
        connection.close()
        
        # Display results
        st.dataframe(results_df)
        
        # Visualization
        if len(results_df.columns) >= 2:
            fig = px.bar(results_df, 
                        x=results_df.columns[0], 
                        y=results_df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Launch Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
from etl_pipeline import fetch_artifacts, transform_artifact_metadata, transform_artifact_media, transform_artifact_colors
from database import get_db_connection, create_tables, load_dataframe_to_sql

# 1. Extract
artifacts = fetch_artifacts(num_records=500)

# 2. Transform
metadata_df = transform_artifact_metadata(artifacts)
media_df = transform_artifact_media(artifacts)
colors_df = transform_artifact_colors(artifacts)

# 3. Load
connection = get_db_connection()
create_tables(connection)
load_dataframe_to_sql(metadata_df, 'artifactmetadata', connection)
load_dataframe_to_sql(media_df, 'artifactmedia', connection)
load_dataframe_to_sql(colors_df, 'artifactcolors', connection)
connection.close()
```

### Custom Analytics Query

```python
custom_query = """
    SELECT 
        m.classification,
        COUNT(DISTINCT c.color) as unique_colors,
        COUNT(*) as total_artifacts
    FROM artifactmetadata m
    JOIN artifactcolors c ON m.objectid = c.objectid
    GROUP BY m.classification
    HAVING total_artifacts > 10
    ORDER BY unique_colors DESC
"""

connection = get_db_connection()
results = execute_query(connection, custom_query)
connection.close()

print(results)
```

## Troubleshooting

### API Rate Limiting
If you encounter rate limits, add delays between requests:

```python
import time

def fetch_artifacts_with_delay(num_records=100, delay=0.5):
    artifacts = []
    # Add time.sleep(delay) between API calls
    for page in range(1, pages + 1):
        time.sleep(delay)
        # ... rest of fetch logic
    return artifacts
```

### Database Connection Issues
Verify credentials and network access:

```python
def test_connection():
    try:
        connection = get_db_connection()
        if connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Missing Data Handling
Handle NULL values in transformations:

```python
def safe_get(artifact, key, default=None):
    """Safely extract value with default"""
    return artifact.get(key) if artifact.get(key) else default

record = {
    'objectid': artifact.get('objectid'),
    'title': safe_get(artifact, 'title', 'Untitled'),
    'culture': safe_get(artifact, 'culture', 'Unknown')
}
```

### Large Dataset Performance
Use batch processing for large datasets:

```python
def load_dataframe_in_batches(df, table_name, connection, batch_size=1000):
    """Load data in batches to avoid memory issues"""
    total_rows = len(df)
    for i in range(0, total_rows, batch_size):
        batch = df.iloc[i:i+batch_size]
        load_dataframe_to_sql(batch, table_name, connection)
        print(f"Loaded batch {i//batch_size + 1}: {len(batch)} records")
```
