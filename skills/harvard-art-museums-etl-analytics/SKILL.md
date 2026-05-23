---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics application using Harvard Art Museums API with SQL database and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create analytics dashboard for art collection
  - extract data from Harvard Art Museums API
  - set up data engineering workflow with Streamlit
  - design SQL schema for artifact metadata
  - visualize museum collection insights
  - implement batch data ingestion pipeline
  - analyze art museum data with SQL queries
---

# Harvard Art Museums ETL Analytics Application

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact data, transforms nested JSON into relational tables, loads it into SQL databases (MySQL/TiDB Cloud), and provides interactive analytics dashboards through Streamlit.

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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

### API Key Setup

1. Request an API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Store it in your `.env` file or configure it through the Streamlit interface

## Running the Application

```bash
streamlit run app.py
```

The application will launch at `http://localhost:8501`

## Database Schema

### Tables Structure

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    objectnumber VARCHAR(100),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percentage FLOAT,
    color_spectrum VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Extract artifact data from Harvard Art Museums API
    """
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
        raise Exception(f"API Error: {response.status_code}")

# Usage with pagination
api_key = os.getenv('HARVARD_API_KEY')
all_artifacts = []

for page in range(1, 6):  # Fetch 5 pages
    artifacts, info = fetch_artifacts(api_key, page=page)
    all_artifacts.extend(artifacts)
    print(f"Fetched page {page}: {len(artifacts)} artifacts")
```

### Transform: Processing Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API response into structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'objectnumber': artifact.get('objectnumber', 'Unknown'),
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image information
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl', ''),
                'media_type': img.get('format', 'Unknown'),
                'caption': img.get('caption', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color information
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percentage': color.get('percent', 0),
                'color_spectrum': color.get('spectrum', 'Unknown')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Inserting into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Establish connection to MySQL/TiDB Cloud
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

def batch_insert_metadata(df, connection):
    """
    Batch insert artifact metadata with ON DUPLICATE KEY UPDATE
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, division, department, dated, objectnumber, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
        title=VALUES(title),
        culture=VALUES(culture),
        century=VALUES(century)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()

def batch_insert_media(df, connection):
    """
    Batch insert media data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, image_url, media_type, caption)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()
```

## Analytics Queries

### Sample SQL Queries

```python
# Query 1: Artifact count by culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
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

# Query 3: Department distribution
query_department = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Query 4: Color analysis
query_colors = """
SELECT ac.color_spectrum, COUNT(*) as frequency,
       AVG(ac.color_percentage) as avg_percentage
FROM artifactcolors ac
GROUP BY ac.color_spectrum
ORDER BY frequency DESC
"""

# Query 5: Media availability
query_media = """
SELECT 
    CASE 
        WHEN COUNT(am.media_id) > 0 THEN 'Has Media'
        ELSE 'No Media'
    END as media_status,
    COUNT(DISTINCT a.id) as artifact_count
FROM artifactmetadata a
LEFT JOIN artifactmedia am ON a.id = am.artifact_id
GROUP BY media_status
"""
```

### Executing Queries in Streamlit

```python
import streamlit as st
import pandas as pd
import plotly.express as px

def execute_query(query, connection):
    """
    Execute SQL query and return DataFrame
    """
    try:
        df = pd.read_sql(query, connection)
        return df
    except Exception as e:
        st.error(f"Query execution error: {e}")
        return None

def visualize_results(df, title, x_col, y_col):
    """
    Create interactive bar chart with Plotly
    """
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title(), 
                y_col: y_col.replace('_', ' ').title()}
    )
    st.plotly_chart(fig, use_container_width=True)

# Streamlit app implementation
st.title("Harvard Art Museums Analytics Dashboard")

# Execute and display query
connection = create_database_connection()

if connection:
    df = execute_query(query_culture, connection)
    
    if df is not None and not df.empty:
        st.subheader("Top 10 Cultures by Artifact Count")
        st.dataframe(df)
        visualize_results(df, "Artifact Distribution by Culture", 
                         'culture', 'artifact_count')
```

## Complete Streamlit Application Pattern

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

# App configuration
st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🎨",
    layout="wide"
)

# Sidebar configuration
st.sidebar.title("Configuration")
api_key = st.sidebar.text_input("Harvard API Key", 
                                value=os.getenv('HARVARD_API_KEY', ''),
                                type="password")

# Main tabs
tab1, tab2, tab3 = st.tabs(["Data Collection", "Analytics", "Visualizations"])

with tab1:
    st.header("ETL Pipeline")
    
    num_pages = st.number_input("Number of pages to fetch", 
                                min_value=1, max_value=10, value=5)
    
    if st.button("Start ETL Process"):
        with st.spinner("Fetching data from API..."):
            # Run ETL pipeline
            all_artifacts = []
            for page in range(1, num_pages + 1):
                artifacts, info = fetch_artifacts(api_key, page=page)
                all_artifacts.extend(artifacts)
                st.progress(page / num_pages)
            
            st.success(f"Fetched {len(all_artifacts)} artifacts")
            
            # Transform and load
            connection = create_database_connection()
            if connection:
                metadata_df = transform_artifact_metadata(all_artifacts)
                batch_insert_metadata(metadata_df, connection)
                st.success("Data loaded successfully!")

with tab2:
    st.header("SQL Analytics")
    
    query_options = {
        "Culture Distribution": query_culture,
        "Century Analysis": query_century,
        "Department Stats": query_department,
        "Color Patterns": query_colors
    }
    
    selected_query = st.selectbox("Select Query", list(query_options.keys()))
    
    if st.button("Execute Query"):
        connection = create_database_connection()
        if connection:
            df = execute_query(query_options[selected_query], connection)
            st.dataframe(df)

with tab3:
    st.header("Interactive Visualizations")
    # Visualization logic here
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(api_key, page, delay=1):
    """
    Fetch data with rate limiting to avoid API throttling
    """
    time.sleep(delay)
    return fetch_artifacts(api_key, page)
```

### Error Handling in ETL

```python
def safe_etl_pipeline(artifacts):
    """
    ETL with comprehensive error handling
    """
    try:
        metadata_df = transform_artifact_metadata(artifacts)
        media_df = transform_artifact_media(artifacts)
        colors_df = transform_artifact_colors(artifacts)
        
        connection = create_database_connection()
        if connection:
            batch_insert_metadata(metadata_df, connection)
            batch_insert_media(media_df, connection)
            connection.close()
            return True
    except Exception as e:
        print(f"ETL Error: {e}")
        return False
```

## Troubleshooting

### API Connection Issues

- **Problem**: API returns 401 Unauthorized
- **Solution**: Verify API key is correct and active in `.env` file

### Database Connection Failures

- **Problem**: `mysql.connector.errors.ProgrammingError`
- **Solution**: Check database credentials, ensure database exists, verify network access to TiDB Cloud

### Memory Issues with Large Datasets

- **Problem**: Out of memory when fetching many pages
- **Solution**: Implement streaming inserts or reduce batch size

```python
# Process in smaller chunks
def stream_etl(api_key, total_pages, batch_size=2):
    for i in range(1, total_pages + 1, batch_size):
        artifacts = []
        for page in range(i, min(i + batch_size, total_pages + 1)):
            data, _ = fetch_artifacts(api_key, page)
            artifacts.extend(data)
        
        # Process and insert batch
        safe_etl_pipeline(artifacts)
```

### Empty Query Results

- **Problem**: Queries return no data
- **Solution**: Verify ETL pipeline completed successfully, check table names and column names match schema

```python
# Verify data exists
def check_data_exists(connection):
    cursor = connection.cursor()
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    count = cursor.fetchone()[0]
    print(f"Total artifacts in database: {count}")
    return count > 0
```
