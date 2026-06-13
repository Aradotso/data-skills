---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I use the Harvard Art Museums API for data engineering
  - build an ETL pipeline with Harvard artifacts data
  - create analytics dashboard for museum collection data
  - extract and transform Harvard museum API data
  - set up SQL database for artifact metadata
  - visualize museum collection data with Streamlit
  - implement data pipeline for Harvard Art Museums
  - query and analyze artifact collection data
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates end-to-end data engineering workflows using the Harvard Art Museums API. It provides ETL pipelines to extract artifact metadata, transform nested JSON into relational structures, load data into SQL databases, and build interactive analytics dashboards with Streamlit.

## What It Does

The Harvard Artifacts Collection Data Engineering & Analytics App enables you to:

- **Extract** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transform** nested JSON responses into normalized relational tables
- **Load** structured data into MySQL/TiDB Cloud databases
- **Analyze** collections using predefined SQL queries
- **Visualize** insights through interactive Plotly charts in a Streamlit dashboard

The application models real-world data engineering patterns: API integration, schema design, batch processing, and analytics visualization.

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (obtain from https://harvardartmuseums.org/collections/api)

### Setup

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
export DB_NAME="harvard_artifacts"
```

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    objectnumber VARCHAR(100),
    division VARCHAR(255),
    accessionyear INT,
    technique VARCHAR(500),
    period VARCHAR(255)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    format VARCHAR(50),
    description TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Collection

Extract artifact data with pagination and error handling:

```python
import requests
import os
import time

def fetch_artifacts(api_key, num_pages=10):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
            
            # Rate limiting - be respectful to API
            time.sleep(1)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_pages=5)
```

### 2. ETL Pipeline

Transform and load data into SQL database:

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(artifacts):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'dated': artifact.get('dated', '')[:255],
            'department': artifact.get('department', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'objectnumber': artifact.get('objectnumber', '')[:100],
            'division': artifact.get('division', '')[:255],
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', '')[:255]
        })
        
        # Extract media
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': image.get('baseimageurl', '')[:1000],
                'format': image.get('format', '')[:50],
                'description': image.get('description', '')
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch load dataframes into MySQL database
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (batch)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, dated, department, classification, 
             objectnumber, division, accessionyear, technique, period)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        if not media_df.empty:
            media_query = """
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, format, description)
                VALUES (%s, %s, %s, %s)
            """
            cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
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

# Complete ETL workflow
artifacts = fetch_artifacts(api_key, num_pages=5)
metadata_df, media_df, colors_df = transform_artifacts(artifacts)
load_to_database(metadata_df, media_df, colors_df)
```

### 3. SQL Analytics Queries

Common analytical queries for artifact insights:

```python
# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Department distribution
query_department = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Color usage analysis
query_colors = """
SELECT c.color, COUNT(DISTINCT c.artifact_id) as artifact_count,
       AVG(c.percent) as avg_percentage
FROM artifactcolors c
GROUP BY c.color
HAVING artifact_count > 10
ORDER BY artifact_count DESC
LIMIT 15
"""

# Artifacts with media
query_media = """
SELECT m.department, 
       COUNT(DISTINCT m.id) as total_artifacts,
       COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
       ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as media_percentage
FROM artifactmetadata m
LEFT JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY m.department
ORDER BY media_percentage DESC
```

### 4. Streamlit Dashboard

Build interactive analytics interface:

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector

def execute_query(query):
    """Execute SQL query and return dataframe"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    df = pd.read_sql(query, connection)
    connection.close()
    return df

# Streamlit app
st.title("🏛️ Harvard Artifacts Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")
query_options = {
    "Top Cultures": query_cultures,
    "Century Distribution": query_century,
    "Department Overview": query_department,
    "Color Analysis": query_colors,
    "Media Availability": query_media
}

selected_query = st.sidebar.selectbox("Select Analysis", list(query_options.keys()))

# Execute and display results
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        df = execute_query(query_options[selected_query])
        
        st.subheader(f"Results: {selected_query}")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=f"{selected_query} Visualization")
            st.plotly_chart(fig, use_container_width=True)
```

## Configuration

### Environment Variables

```bash
# API Configuration
HARVARD_API_KEY=your_api_key_from_harvard_museums

# Database Configuration
DB_HOST=your_mysql_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Streamlit Configuration

Create `.streamlit/config.toml`:

```toml
[server]
port = 8501
headless = true

[theme]
primaryColor = "#A51C30"
backgroundColor = "#FFFFFF"
secondaryBackgroundColor = "#F0F2F6"
textColor = "#262730"
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the highest artifact ID already loaded"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def fetch_new_artifacts(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    # Implementation with filtering
    pass
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_pipeline():
    try:
        artifacts = fetch_artifacts(api_key)
        logger.info(f"Fetched {len(artifacts)} artifacts")
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        logger.info("Transformation complete")
        
        load_to_database(metadata_df, media_df, colors_df)
        logger.info("Data loaded successfully")
        
    except Exception as e:
        logger.error(f"ETL pipeline failed: {e}")
        raise
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests using `time.sleep(1)` to avoid hitting rate limits.

**Database Connection Issues**: Verify environment variables are set correctly and database is accessible.

**Large Data Volumes**: Use batch inserts with `executemany()` instead of individual inserts for better performance.

**Memory Issues**: Process artifacts in chunks rather than loading all data into memory at once.

**Missing Data Fields**: Always use `.get()` with defaults when accessing JSON fields to handle missing values gracefully.
