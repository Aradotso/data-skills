---
name: harvard-art-museums-data-engineering-analytics
description: End-to-end data engineering and analytics application using Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts
  - fetch and store Harvard museum collection data
  - set up data engineering pipeline for art museum API
  - analyze Harvard Art Museums collection with SQL
  - visualize museum artifacts data with Streamlit
  - process Harvard museum API data into database
  - create museum collection analytics application
---

# Harvard Art Museums Data Engineering Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It implements ETL pipelines, relational database design, SQL analytics, and interactive visualizations using Streamlit.

**Architecture:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
sqlalchemy
```

## Configuration

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store the API key securely using environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
```

### Database Configuration

The project supports MySQL or TiDB Cloud. Configure database credentials:

```python
import os

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Database Schema

The application uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    accessionyear INT,
    dated VARCHAR(200),
    primaryimageurl TEXT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri TEXT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only get artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data['records']
total_pages = data['info']['pages']
```

### Transform: Process JSON to Relational Format

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into metadata dataframe"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'accessionyear': artifact.get('accessionyear'),
            'dated': artifact.get('dated'),
            'primaryimageurl': artifact.get('primaryimageurl')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Extract media information from artifacts"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_records.append({
                'artifact_id': artifact_id,
                'iiifbaseuri': image.get('iiifbaseuri'),
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            })
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """Extract color information from artifacts"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

### Load: Insert Data into Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(config):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(**config)
        if connection.is_connected():
            return connection
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
        return None

def batch_insert_metadata(connection, df):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     division, accessionyear, dated, primaryimageurl)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} records")
    except Error as e:
        print(f"Error inserting data: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(connection, df):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, iiifbaseuri, baseimageurl, format, height, width)
    VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_colors(connection, df):
    """Batch insert color data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Complete ETL Pipeline

```python
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """Run complete ETL pipeline"""
    connection = create_database_connection(db_config)
    
    if not connection:
        print("Failed to connect to database")
        return
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}/{num_pages}")
        
        # Extract
        data = fetch_artifacts(api_key, page=page, size=100)
        artifacts = data['records']
        
        # Transform
        metadata_df = transform_artifact_metadata(artifacts)
        media_df = transform_artifact_media(artifacts)
        colors_df = transform_artifact_colors(artifacts)
        
        # Load
        batch_insert_metadata(connection, metadata_df)
        batch_insert_media(connection, media_df)
        batch_insert_colors(connection, colors_df)
        
        print(f"Page {page} completed")
    
    connection.close()
    print("ETL pipeline completed")

# Execute pipeline
api_key = os.getenv('HARVARD_API_KEY')
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

run_etl_pipeline(api_key, db_config, num_pages=10)
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures by artifact count
query_1 = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century distribution
query_2 = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Department-wise artifact distribution
query_3 = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Color usage analysis
query_4 = """
SELECT c.color, COUNT(DISTINCT c.artifact_id) as artifact_count,
       AVG(c.percent) as avg_percentage
FROM artifactcolors c
GROUP BY c.color
ORDER BY artifact_count DESC
LIMIT 15
"""

# Media format distribution
query_5 = """
SELECT format, COUNT(*) as count,
       AVG(width) as avg_width, AVG(height) as avg_height
FROM artifactmedia
WHERE format IS NOT NULL
GROUP BY format
"""

# Artifacts with images vs without
query_6 = """
SELECT 
    CASE WHEN primaryimageurl IS NOT NULL 
         THEN 'With Image' 
         ELSE 'Without Image' 
    END as image_status,
    COUNT(*) as count
FROM artifactmetadata
GROUP BY image_status
"""

# Classification breakdown
query_7 = """
SELECT classification, culture, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND culture IS NOT NULL
GROUP BY classification, culture
ORDER BY count DESC
LIMIT 20
"""

def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    try:
        df = pd.read_sql(query, connection)
        return df
    except Exception as e:
        print(f"Query execution error: {e}")
        return None
```

## Streamlit Application

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    try:
        connection = create_database_connection(db_config)
        st.sidebar.success("✅ Database connected")
    except Exception as e:
        st.sidebar.error(f"❌ Database connection failed: {e}")
        return
    
    # Analytics section
    tab1, tab2, tab3 = st.tabs([
        "📊 Analytics Queries",
        "🔄 ETL Pipeline",
        "📈 Visualizations"
    ])
    
    with tab1:
        show_analytics_queries(connection)
    
    with tab2:
        show_etl_pipeline()
    
    with tab3:
        show_visualizations(connection)

def show_analytics_queries(connection):
    """Display analytics query interface"""
    st.header("SQL Analytics Queries")
    
    queries = {
        "Top 10 Cultures": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        "Century Distribution": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        "Color Analysis": """
            SELECT color, COUNT(DISTINCT artifact_id) as artifacts,
                   AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY artifacts DESC
            LIMIT 15
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Executing query..."):
            df = execute_query(connection, queries[selected_query])
            
            if df is not None and not df.empty:
                st.subheader("Results")
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=f"{selected_query} - Visualization"
                    )
                    st.plotly_chart(fig, use_container_width=True)

def show_visualizations(connection):
    """Display pre-built visualizations"""
    st.header("Interactive Visualizations")
    
    col1, col2 = st.columns(2)
    
    with col1:
        # Department distribution
        query = """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            GROUP BY department
        """
        df = execute_query(connection, query)
        
        if df is not None:
            fig = px.pie(
                df,
                values='count',
                names='department',
                title='Artifacts by Department'
            )
            st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        # Color spectrum
        query = """
            SELECT spectrum, COUNT(*) as count
            FROM artifactcolors
            WHERE spectrum IS NOT NULL
            GROUP BY spectrum
        """
        df = execute_query(connection, query)
        
        if df is not None:
            fig = px.bar(
                df,
                x='spectrum',
                y='count',
                title='Color Spectrum Distribution',
                color='count'
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Running the Streamlit App

```bash
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(api_key, max_pages=None):
    """Fetch all artifacts with pagination"""
    all_artifacts = []
    page = 1
    
    while True:
        data = fetch_artifacts(api_key, page=page)
        artifacts = data['records']
        all_artifacts.extend(artifacts)
        
        total_pages = data['info']['pages']
        
        if max_pages and page >= max_pages:
            break
        
        if page >= total_pages:
            break
        
        page += 1
        
    return all_artifacts
```

### Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, page, delay=1):
    """Fetch data with rate limiting"""
    data = fetch_artifacts(api_key, page=page)
    time.sleep(delay)  # Wait 1 second between requests
    return data
```

### Error Handling

```python
def safe_fetch_artifacts(api_key, page, retries=3):
    """Fetch artifacts with retry logic"""
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

### API Connection Issues

**Problem:** API requests failing with 401 Unauthorized
**Solution:** Verify API key is correctly set in environment variables

```python
import os
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY environment variable not set")
```

### Database Connection Errors

**Problem:** Unable to connect to MySQL/TiDB
**Solution:** Check database credentials and network connectivity

```python
def test_db_connection(config):
    """Test database connection"""
    try:
        conn = mysql.connector.connect(**config)
        if conn.is_connected():
            print("✅ Database connection successful")
            db_info = conn.get_server_info()
            print(f"MySQL Server version: {db_info}")
            conn.close()
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Memory Issues with Large Datasets

**Problem:** Out of memory when processing large number of artifacts
**Solution:** Use batch processing

```python
def etl_with_batching(api_key, db_config, batch_size=100):
    """Process data in smaller batches"""
    connection = create_database_connection(db_config)
    
    for page in range(1, 100):  # Process 100 pages
        artifacts = fetch_artifacts(api_key, page=page, size=batch_size)
        
        # Process immediately and clear from memory
        metadata_df = transform_artifact_metadata(artifacts['records'])
        batch_insert_metadata(connection, metadata_df)
        
        del metadata_df  # Free memory
        
    connection.close()
```

### Duplicate Key Errors

**Problem:** Duplicate primary key violations during insert
**Solution:** Use INSERT ... ON DUPLICATE KEY UPDATE

```python
insert_query = """
INSERT INTO artifactmetadata (id, title, culture)
VALUES (%s, %s, %s)
ON DUPLICATE KEY UPDATE
    title = VALUES(title),
    culture = VALUES(culture)
"""
```

## Performance Optimization

### Index Creation

```sql
-- Add indexes for frequently queried columns
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_century ON artifactmetadata(century);
CREATE INDEX idx_department ON artifactmetadata(department);
CREATE INDEX idx_artifact_id ON artifactmedia(artifact_id);
CREATE INDEX idx_color ON artifactcolors(artifact_id, color);
```

### Query Optimization

```python
# Use connection pooling for better performance
from sqlalchemy import create_engine

engine = create_engine(
    f"mysql+mysqlconnector://{user}:{password}@{host}:{port}/{database}",
    pool_size=10,
    max_overflow=20
)

def execute_query_optimized(query):
    """Execute query using SQLAlchemy engine"""
    return pd.read_sql(query, engine)
```
