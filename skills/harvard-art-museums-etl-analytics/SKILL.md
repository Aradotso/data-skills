---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, MySQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up analytics dashboard for Harvard Art Museums data
  - extract and analyze art museum collection data
  - build a Streamlit app for art museum analytics
  - connect to Harvard Art Museums API and store in SQL
  - visualize museum artifact data with Plotly
  - implement batch ETL for museum collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases with proper schema design
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards

**Architecture**: API → ETL → SQL → Analytics → Visualization

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

### Environment Setup

Create a `.env` file for sensitive credentials:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Configuration

Load environment variables in your Python scripts:

```python
import os
from dotenv import load_dotenv

load_dotenv()

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

### Harvard API Key

Get your free API key from: https://docs.api.harvardartmuseums.org/

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Extract artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, metadata = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {metadata['totalrecords']}")
```

### Transform: Clean and Structure Data

```python
import pandas as pd

def transform_artifact_data(raw_artifacts):
    """
    Transform nested JSON into flat dataframes for each table
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions')
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact['id'],
                    'image_url': image.get('baseimageurl'),
                    'media_type': 'image'
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact['id'],
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percent': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Usage
df_metadata, df_media, df_colors = transform_artifact_data(artifacts)
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(config):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(**config)
        return connection
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
        return None

def batch_insert_metadata(df, connection):
    """
    Batch insert artifact metadata with duplicate handling
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     dated, accessionyear, technique, medium, dimensions)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title = VALUES(title),
    culture = VALUES(culture)
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

def batch_insert_media(df, connection):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, media_type)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

# Usage
connection = create_database_connection(db_config)
if connection:
    batch_insert_metadata(df_metadata, connection)
    batch_insert_media(df_media, connection)
    connection.close()
```

## Analytics Queries

### Common Analytical Patterns

```python
def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    return pd.DataFrame(results, columns=columns)

# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Artifacts by century
query_century = """
SELECT century, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY artifact_count DESC
"""

# Media availability analysis
query_media = """
SELECT 
    CASE WHEN am.media_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
    COUNT(DISTINCT a.id) as artifact_count
FROM artifactmetadata a
LEFT JOIN artifactmedia am ON a.id = am.artifact_id
GROUP BY media_status
"""

# Color distribution
query_colors = """
SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
FROM artifactcolors
WHERE spectrum IS NOT NULL
GROUP BY spectrum
ORDER BY count DESC
"""

# Usage
connection = create_database_connection(db_config)
df_cultures = execute_query(connection, query_cultures)
df_century = execute_query(connection, query_century)
connection.close()
```

## Streamlit Dashboard

### Basic Dashboard Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for API configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    
    num_records = st.slider("Records to Fetch", 10, 100, 50)
    
    if st.button("Fetch & Load Data"):
        with st.spinner("Running ETL Pipeline..."):
            # Run ETL
            artifacts, _ = fetch_artifacts(api_key, size=num_records)
            df_meta, df_media, df_colors = transform_artifact_data(artifacts)
            
            connection = create_database_connection(db_config)
            batch_insert_metadata(df_meta, connection)
            batch_insert_media(df_media, connection)
            connection.close()
            
            st.success(f"Loaded {len(df_meta)} artifacts successfully!")

# Analytics Section
st.header("📊 Analytics Queries")

query_options = {
    "Top Cultures": query_cultures,
    "Artifacts by Century": query_century,
    "Media Availability": query_media,
    "Color Distribution": query_colors
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))

if st.button("Run Query"):
    connection = create_database_connection(db_config)
    results_df = execute_query(connection, query_options[selected_query])
    connection.close()
    
    # Display results
    st.dataframe(results_df)
    
    # Auto-generate visualization
    if len(results_df.columns) >= 2:
        fig = px.bar(results_df, x=results_df.columns[0], y=results_df.columns[1],
                     title=selected_query)
        st.plotly_chart(fig, use_container_width=True)
```

### Advanced Visualization

```python
def create_culture_chart(df):
    """Create interactive culture distribution chart"""
    fig = px.bar(
        df,
        x='culture',
        y='count',
        title='Top 10 Cultures in Collection',
        labels={'culture': 'Culture', 'count': 'Number of Artifacts'},
        color='count',
        color_continuous_scale='Viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_pie_chart(df):
    """Create color spectrum distribution pie chart"""
    fig = px.pie(
        df,
        values='count',
        names='spectrum',
        title='Color Spectrum Distribution',
        hole=0.3
    )
    return fig

# Usage in Streamlit
st.plotly_chart(create_culture_chart(df_cultures), use_container_width=True)
st.plotly_chart(create_color_pie_chart(df_colors), use_container_width=True)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Run with specific port
streamlit run app.py --server.port 8501

# Run in development mode with auto-reload
streamlit run app.py --server.runOnSave true
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_multiple_pages(api_key, total_pages=5, delay=1):
    """Fetch multiple pages with rate limiting"""
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        artifacts, info = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(artifacts)
        
        print(f"Fetched page {page}/{total_pages}")
        time.sleep(delay)  # Rate limiting
    
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, db_config):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        artifacts, _ = fetch_artifacts(api_key)
        
        # Transform
        df_meta, df_media, df_colors = transform_artifact_data(artifacts)
        
        # Validate
        if df_meta.empty:
            raise ValueError("No metadata extracted")
        
        # Load
        connection = create_database_connection(db_config)
        if not connection:
            raise Exception("Database connection failed")
        
        batch_insert_metadata(df_meta, connection)
        batch_insert_media(df_media, connection)
        
        connection.close()
        return True
        
    except Exception as e:
        print(f"ETL Pipeline Error: {e}")
        return False
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    """Test if API key is valid and service is reachable"""
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1},
            timeout=10
        )
        if response.status_code == 200:
            print("✓ API connection successful")
            return True
        elif response.status_code == 401:
            print("✗ Invalid API key")
            return False
        else:
            print(f"✗ API error: {response.status_code}")
            return False
    except requests.exceptions.RequestException as e:
        print(f"✗ Connection error: {e}")
        return False
```

### Database Connection Issues

```python
# Test database connectivity
def test_database_connection(config):
    """Test database connection and permissions"""
    try:
        connection = mysql.connector.connect(**config)
        cursor = connection.cursor()
        cursor.execute("SELECT 1")
        cursor.fetchone()
        cursor.close()
        connection.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database error: {e}")
        return False
```

### Common Issues

**Issue**: "Table doesn't exist" error
```python
# Create tables if they don't exist
def initialize_database(connection):
    """Create all required tables"""
    cursor = connection.cursor()
    
    # Run schema creation scripts
    cursor.execute("""CREATE TABLE IF NOT EXISTS artifactmetadata (...)""")
    cursor.execute("""CREATE TABLE IF NOT EXISTS artifactmedia (...)""")
    cursor.execute("""CREATE TABLE IF NOT EXISTS artifactcolors (...)""")
    
    connection.commit()
    cursor.close()
```

**Issue**: Duplicate key errors during insert
- Use `ON DUPLICATE KEY UPDATE` in insert queries (shown in batch_insert_metadata example)

**Issue**: API rate limiting (429 errors)
- Add delays between requests (shown in fetch_multiple_pages example)
- Reduce batch size

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API.
