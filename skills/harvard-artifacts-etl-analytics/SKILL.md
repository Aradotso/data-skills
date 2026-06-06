---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Streamlit and SQL
  - set up Harvard artifacts collection data pipeline
  - query and visualize museum artifact metadata
  - implement batch data loading for art collections
  - analyze artifact colors and media with SQL
  - build end-to-end data engineering project
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Structured schema for artifact metadata, media, and color information
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts

**Architecture Flow**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
# Create .env file with:
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_database_host
# DB_USER=your_database_user
# DB_PASSWORD=your_database_password
# DB_NAME=your_database_name
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

### 1. Harvard API Key Setup

Get your API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform

Store in environment variable:

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

### 2. Database Configuration

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None
```

### 3. Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    description TEXT,
    provenance TEXT,
    commentary TEXT,
    dated_begin INT,
    dated_end INT,
    accession_year INT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### 1. Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """Fetch artifacts from Harvard API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            if 'records' in data:
                all_artifacts.extend(data['records'])
                print(f"Fetched page {page}: {len(data['records'])} artifacts")
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### 2. Transform: Process JSON to Relational Format

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifacts into metadata dataframe"""
    metadata = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'commentary': artifact.get('commentary'),
            'dated_begin': artifact.get('datebegin'),
            'dated_end': artifact.get('dateend'),
            'accession_year': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Transform media/images data"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'media_type': 'image'
            })
    
    return pd.DataFrame(media)

def transform_artifact_colors(artifacts):
    """Transform color data"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### 3. Load: Batch Insert into SQL

```python
def batch_insert_metadata(connection, df):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, 
     dated, period, technique, medium, dimensions, creditline, description,
     provenance, commentary, dated_begin, dated_end, accession_year, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(connection, df):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, media_type)
    VALUES (%s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## SQL Analytics Queries

### Common Analytical Patterns

```python
# Query 1: Artifact count by culture
QUERY_BY_CULTURE = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 20;
"""

# Query 2: Artifacts by century
QUERY_BY_CENTURY = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Department distribution
QUERY_BY_DEPARTMENT = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Query 4: Media availability analysis
QUERY_MEDIA_STATS = """
SELECT 
    a.classification,
    COUNT(DISTINCT a.id) as total_artifacts,
    COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
    ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / COUNT(DISTINCT a.id), 2) as media_percentage
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.classification
HAVING total_artifacts > 10
ORDER BY media_percentage DESC;
"""

# Query 5: Color analysis
QUERY_COLOR_DISTRIBUTION = """
SELECT color, COUNT(*) as frequency
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY frequency DESC
LIMIT 15;
"""

# Query 6: Time period analysis
QUERY_TIME_PERIOD = """
SELECT 
    FLOOR(dated_begin / 100) * 100 as century_start,
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE dated_begin IS NOT NULL
GROUP BY century_start
ORDER BY century_start;
"""
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    
    queries = {
        "Artifacts by Culture": QUERY_BY_CULTURE,
        "Artifacts by Century": QUERY_BY_CENTURY,
        "Department Distribution": QUERY_BY_DEPARTMENT,
        "Media Availability": QUERY_MEDIA_STATS,
        "Color Distribution": QUERY_COLOR_DISTRIBUTION,
        "Time Period Analysis": QUERY_TIME_PERIOD
    }
    
    selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))
    
    # Execute query
    if st.sidebar.button("Run Query"):
        connection = create_connection()
        if connection:
            df = pd.read_sql(queries[selected_query], connection)
            
            # Display results
            st.subheader(f"Results: {selected_query}")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
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

## Complete ETL Workflow

```python
def run_etl_pipeline(api_key, connection, num_pages=5):
    """Complete ETL pipeline execution"""
    
    # EXTRACT
    st.info("Step 1: Extracting data from Harvard API...")
    artifacts = fetch_artifacts(api_key, num_pages=num_pages)
    st.success(f"Extracted {len(artifacts)} artifacts")
    
    # TRANSFORM
    st.info("Step 2: Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    st.success("Data transformation complete")
    
    # LOAD
    st.info("Step 3: Loading data into database...")
    batch_insert_metadata(connection, df_metadata)
    batch_insert_media(connection, df_media)
    batch_insert_colors(connection, df_colors)
    st.success("ETL pipeline complete!")
    
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'colors_count': len(df_colors)
    }
```

## Troubleshooting

### API Rate Limiting
- Add `time.sleep(0.5)` between requests
- Implement exponential backoff for retries
- Monitor API quota limits

### Database Connection Issues
```python
# Test connection before ETL
connection = create_connection()
if connection and connection.is_connected():
    print("Database connected successfully")
else:
    print("Failed to connect - check credentials")
```

### Missing Data Handling
```python
# Handle null values in transformation
record = {
    'title': artifact.get('title', 'Unknown'),
    'culture': artifact.get('culture', None),
    # Use None for SQL NULL values
}
```

### Memory Issues with Large Datasets
```python
# Process in batches
BATCH_SIZE = 1000
for i in range(0, len(df), BATCH_SIZE):
    batch = df[i:i+BATCH_SIZE]
    batch_insert_metadata(connection, batch)
```

### Streamlit Caching for Performance
```python
@st.cache_data
def load_query_results(query):
    connection = create_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

This skill enables building production-ready ETL pipelines and analytics dashboards for museum data using modern data engineering practices.
