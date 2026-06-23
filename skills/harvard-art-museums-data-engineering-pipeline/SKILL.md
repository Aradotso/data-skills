---
name: harvard-art-museums-data-engineering-pipeline
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API data with MySQL/TiDB and Streamlit
triggers:
  - how do I build a data pipeline with the Harvard Art Museums API
  - create an ETL pipeline for museum artifact data
  - set up analytics dashboard for Harvard Art Museums collection
  - extract and analyze Harvard Art Museums API data
  - build a Streamlit app for museum artifacts visualization
  - implement SQL analytics on Harvard Art Museums data
  - create data engineering pipeline for art collection metadata
  - fetch and store Harvard Art Museums artifacts in MySQL
---

# Harvard Art Museums Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases (MySQL/TiDB Cloud), and building interactive analytics dashboards with Streamlit and Plotly.

## What This Project Does

- **API Data Collection**: Fetches artifact metadata, media, and color information from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational tables
- **SQL Storage**: Creates and populates three core tables (`artifactmetadata`, `artifactmedia`, `artifactcolors`)
- **Analytics Engine**: Executes 20+ predefined SQL queries for insights
- **Visualization Dashboard**: Interactive Streamlit interface with Plotly charts

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

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Getting Harvard Art Museums API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free for educational/non-commercial use)
3. Add to `.env` file

### Database Setup

The application supports MySQL and TiDB Cloud. Ensure your database is accessible and credentials are configured.

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, max_records=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(all_artifacts) < max_records:
        params = {
            'apikey': api_key,
            'size': min(size, max_records - len(all_artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:max_records]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, max_records=500)
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata = {
            'id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accession_year': artifact.get('accessionyear'),
            'object_number': artifact.get('objectnumber')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'image_width': img.get('width'),
                'image_height': img.get('height'),
                'format': img.get('format')
            }
            media_list.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_name': color.get('color'),
                'hex_value': color.get('hue'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Database Schema Creation

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema(connection):
    """
    Create the three core tables for artifact data
    """
    cursor = connection.cursor()
    
    # Create artifactmetadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            accession_year INT,
            object_number VARCHAR(100)
        )
    """)
    
    # Create artifactmedia table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(500),
            image_width INT,
            image_height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Create artifactcolors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_name VARCHAR(100),
            hex_value VARCHAR(50),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Data Loading (ETL Load Phase)

```python
def load_data_to_db(metadata_df, media_df, colors_df, connection):
    """
    Batch insert data into MySQL database
    """
    cursor = connection.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, dated, accession_year, object_number)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Load media
    if not media_df.empty:
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, image_url, image_width, image_height, format)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
    
    # Load colors
    if not colors_df.empty:
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color_name, hex_value, percentage)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
    
    connection.commit()
    cursor.close()
```

### 5. Analytical Queries

```python
# Example analytical queries

# Query 1: Artifacts by culture
query_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Most common colors
query_colors = """
    SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query 4: Media availability
query_media = """
    SELECT 
        COUNT(DISTINCT am.id) as total_artifacts,
        COUNT(DISTINCT med.artifact_id) as artifacts_with_media,
        ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia med ON am.id = med.artifact_id
"""

# Query 5: Department distribution
query_department = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""
```

### 6. Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def run_query_and_visualize(connection, query, chart_title):
    """
    Execute SQL query and create visualization
    """
    df = pd.read_sql(query, connection)
    
    st.subheader(chart_title)
    st.dataframe(df)
    
    if len(df.columns) >= 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=chart_title
        )
        st.plotly_chart(fig)

# Streamlit app structure
def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Database connection
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # Query selection
    query_options = {
        "Artifacts by Culture": query_culture,
        "Artifacts by Century": query_century,
        "Most Common Colors": query_colors,
        "Media Availability": query_media,
        "Department Distribution": query_department
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Analysis"):
        run_query_and_visualize(
            connection, 
            query_options[selected_query],
            selected_query
        )
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Full ETL Pipeline Execution

```python
from dotenv import load_dotenv
import os

load_dotenv()

# Step 1: Extract
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, max_records=1000)

# Step 2: Transform
metadata_df, media_df, colors_df = transform_artifacts(artifacts)

# Step 3: Load
connection = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

create_database_schema(connection)
load_data_to_db(metadata_df, media_df, colors_df, connection)
connection.close()

print(f"ETL Complete: {len(metadata_df)} artifacts loaded")
```

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, max_records=100, delay=0.5):
    """
    Fetch artifacts with rate limiting
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < max_records:
        params = {
            'apikey': api_key,
            'size': 100,
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            all_artifacts.extend(records)
            page += 1
            
            # Rate limiting
            time.sleep(delay)
        else:
            break
    
    return all_artifacts[:max_records]
```

## Troubleshooting

### API Key Issues

**Problem**: `401 Unauthorized` error
**Solution**: Verify your API key is correct and active at https://www.harvardartmuseums.org/collections/api

### Database Connection Errors

**Problem**: `mysql.connector.errors.DatabaseError`
**Solution**: 
- Check database credentials in `.env`
- Ensure database exists: `CREATE DATABASE harvard_artifacts;`
- Verify network access to database host

### Empty DataFrames

**Problem**: No data in transformed dataframes
**Solution**: Check API response structure - some fields may be null. Add null checks:

```python
title = artifact.get('title') or 'Unknown'
culture = artifact.get('culture') or 'Not Specified'
```

### Foreign Key Constraint Errors

**Problem**: Cannot insert into `artifactmedia` or `artifactcolors`
**Solution**: Ensure `artifactmetadata` is loaded first, and artifact IDs exist:

```python
# Load in correct order
load_metadata(metadata_df, connection)
load_media(media_df, connection)
load_colors(colors_df, connection)
```

### Streamlit Port Already in Use

**Problem**: `Address already in use`
**Solution**: 
```bash
streamlit run app.py --server.port 8502
```

### Memory Issues with Large Datasets

**Problem**: Out of memory when fetching thousands of artifacts
**Solution**: Use chunked processing:

```python
def process_in_chunks(api_key, total_records=10000, chunk_size=500):
    for offset in range(0, total_records, chunk_size):
        artifacts = fetch_artifacts(api_key, max_records=chunk_size)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_data_to_db(metadata_df, media_df, colors_df, connection)
```
