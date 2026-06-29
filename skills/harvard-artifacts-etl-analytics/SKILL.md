---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - show me how to create a Streamlit analytics dashboard for museum data
  - help me extract and load Harvard museum artifacts into SQL
  - how do I query and visualize Harvard Art Museums collection data
  - build a data engineering pipeline with museum API
  - create interactive analytics app for art collection data
  - set up SQL database for Harvard artifacts ETL
  - visualize museum artifact data with Plotly and Streamlit
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into relational database tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud with batch inserts
- **Analyzes** using 20+ predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# Set environment variable
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

Configure MySQL/TiDB connection:

```python
import os
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    description TEXT,
    provenance TEXT,
    people TEXT,
    url VARCHAR(500),
    lastupdate DATETIME
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    renditionnumber VARCHAR(50),
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

def collect_all_artifacts(api_key, max_records=1000):
    """
    Collect artifacts with pagination handling
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_metadata(artifacts):
    """
    Transform artifact metadata into structured format
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'people': str(artifact.get('people', [])),
            'url': artifact.get('url'),
            'lastupdate': artifact.get('lastupdate')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """
    Extract media information from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'iiifbaseuri': image.get('iiifbaseuri'),
                'baseimageurl': image.get('baseimageurl'),
                'renditionnumber': image.get('renditionnumber'),
                'format': image.get('format')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """
    Extract color data from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Insert into Database

```python
def load_metadata(df, connection):
    """
    Batch insert metadata into database
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, department, classification, 
         medium, dimensions, creditline, description, provenance, people, url, lastupdate)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_media(df, connection):
    """
    Batch insert media data
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, iiifbaseuri, baseimageurl, renditionnumber, format)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_colors(df, connection):
    """
    Batch insert color data
    """
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Artifact count by culture
culture_query = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts with images
media_query = """
    SELECT m.department, COUNT(DISTINCT am.artifact_id) as artifacts_with_images
    FROM artifactmedia am
    JOIN artifactmetadata m ON am.artifact_id = m.id
    GROUP BY m.department
    ORDER BY artifacts_with_images DESC
"""

# Query 3: Color distribution
color_query = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percentage
    FROM artifactcolors
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 15
"""

# Query 4: Artifacts by century
century_query = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
    LIMIT 20
"""

# Query 5: Classification analysis
classification_query = """
    SELECT classification, COUNT(*) as total,
           SUM(CASE WHEN medium IS NOT NULL THEN 1 ELSE 0 END) as with_medium
    FROM artifactmetadata
    GROUP BY classification
    ORDER BY total DESC
    LIMIT 10
"""
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
import os

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums ETL & Analytics Dashboard")

# Sidebar for configuration
with st.sidebar:
    st.header("Configuration")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                             value=os.getenv('HARVARD_API_KEY', ''))
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = collect_all_artifacts(api_key, max_records=500)
            
        with st.spinner("Transforming data..."):
            df_metadata = transform_metadata(artifacts)
            df_media = transform_media(artifacts)
            df_colors = transform_colors(artifacts)
            
        with st.spinner("Loading into database..."):
            connection = mysql.connector.connect(**db_config)
            load_metadata(df_metadata, connection)
            load_media(df_media, connection)
            load_colors(df_colors, connection)
            connection.close()
            
        st.success("ETL Pipeline completed successfully!")

# Analytics Section
st.header("📊 Analytics Dashboard")

# Query selector
query_options = {
    "Artifacts by Culture": culture_query,
    "Media Availability": media_query,
    "Color Distribution": color_query,
    "Century Analysis": century_query,
    "Classification Breakdown": classification_query
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))

if st.button("Run Analysis"):
    connection = mysql.connector.connect(**db_config)
    df_result = pd.read_sql(query_options[selected_query], connection)
    connection.close()
    
    col1, col2 = st.columns([1, 1])
    
    with col1:
        st.subheader("Query Results")
        st.dataframe(df_result)
    
    with col2:
        st.subheader("Visualization")
        
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, 
                        x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

### Advanced Visualization

```python
def create_interactive_chart(df, chart_type='bar'):
    """
    Create interactive Plotly visualizations
    """
    if chart_type == 'bar':
        fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                     color=df.columns[1],
                     color_continuous_scale='Viridis')
    elif chart_type == 'pie':
        fig = px.pie(df, names=df.columns[0], values=df.columns[1])
    elif chart_type == 'scatter':
        fig = px.scatter(df, x=df.columns[0], y=df.columns[1],
                        size=df.columns[1], hover_data=df.columns)
    
    fig.update_layout(
        hovermode='x unified',
        showlegend=True,
        height=500
    )
    
    return fig

# Usage in Streamlit
chart_type = st.radio("Chart Type", ['bar', 'pie', 'scatter'])
fig = create_interactive_chart(df_result, chart_type)
st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(api_key, db_config, max_records=1000):
    """
    Execute full ETL pipeline
    """
    # Extract
    print("Extracting data from API...")
    artifacts = collect_all_artifacts(api_key, max_records)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_metadata(artifacts)
    df_media = transform_media(artifacts)
    df_colors = transform_colors(artifacts)
    
    # Load
    print("Loading into database...")
    connection = mysql.connector.connect(**db_config)
    
    try:
        load_metadata(df_metadata, connection)
        load_media(df_media, connection)
        load_colors(df_colors, connection)
        print(f"Successfully loaded {len(df_metadata)} artifacts")
    finally:
        connection.close()
    
    return {
        'metadata_count': len(df_metadata),
        'media_count': len(df_media),
        'color_count': len(df_colors)
    }
```

### Error Handling

```python
def safe_api_call(api_key, page, retries=3):
    """
    API call with retry logic
    """
    import time
    
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except Exception as e:
            if attempt < retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1}/{retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time
time.sleep(1)  # Wait 1 second between API calls
```

### Database Connection Issues
```python
# Test connection
try:
    connection = mysql.connector.connect(**db_config)
    print("Database connected successfully")
    connection.close()
except mysql.connector.Error as err:
    print(f"Connection failed: {err}")
```

### Memory Management for Large Datasets
```python
# Process in chunks
chunk_size = 100
for i in range(0, len(all_artifacts), chunk_size):
    chunk = all_artifacts[i:i+chunk_size]
    df_chunk = transform_metadata(chunk)
    load_metadata(df_chunk, connection)
```

### Missing Data Handling
```python
# Handle None values in transformation
metadata = {
    'title': artifact.get('title', 'Unknown'),
    'culture': artifact.get('culture') or 'Not Specified',
    'century': artifact.get('century', 'Unknown Period')
}
```
