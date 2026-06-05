---
name: harvard-art-museums-etl-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering pipeline with Harvard Art Museums API
  - set up artifact collection analytics with Streamlit
  - implement SQL analytics for museum collections
  - build a data visualization dashboard for art museums
  - create an end-to-end data pipeline with API integration
  - analyze Harvard Art Museums data with Python
  - design relational database for artifact metadata
---

# Harvard Art Museums ETL Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

The Harvard Artifacts Collection Data Engineering Analytics App is an end-to-end data engineering solution that demonstrates real-world ETL patterns using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational tables, loads data into SQL databases, and provides interactive analytics through Streamlit dashboards.

**Architecture Flow:**
```
Harvard API → ETL (Python) → SQL Database → Analytics Queries → Streamlit Dashboard → Plotly Visualizations
```

## Installation

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
export DB_NAME="your_database_name"

# Run the Streamlit application
streamlit run app.py
```

## Database Schema

The application uses three relational tables:

**artifactmetadata** (main table)
- `objectid` (PRIMARY KEY)
- `title`
- `culture`
- `century`
- `department`
- `classification`
- `dated`
- `technique`

**artifactmedia**
- `id` (PRIMARY KEY)
- `objectid` (FOREIGN KEY → artifactmetadata)
- `baseimageurl`
- `format`
- `height`
- `width`

**artifactcolors**
- `id` (PRIMARY KEY)
- `objectid` (FOREIGN KEY → artifactmetadata)
- `color`
- `spectrum`
- `hue`
- `percent`

## API Integration

### Fetching Data from Harvard Art Museums API

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
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key, max_pages=5):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    import time
    
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(data['records'])
        
        # Rate limiting - be respectful to the API
        time.sleep(1)
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract: Parse JSON Response

```python
import pandas as pd

def extract_metadata(artifacts):
    """
    Extract artifact metadata into a DataFrame
    """
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'technique': artifact.get('technique')
        })
    
    return pd.DataFrame(metadata)
```

### Transform: Flatten Nested Data

```python
def extract_media(artifacts):
    """
    Extract and flatten nested media information
    """
    media_data = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_data.append({
                'objectid': objectid,
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
            })
    
    return pd.DataFrame(media_data)

def extract_colors(artifacts):
    """
    Extract color information from artifacts
    """
    color_data = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_data)
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
        return None

def load_metadata(df, connection):
    """
    Batch insert metadata into SQL database
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, department, classification, dated, technique)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} rows into artifactmetadata")
    except Error as e:
        print(f"Error inserting data: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Analytics Queries

### Sample SQL Analytics

```python
def get_artifacts_by_culture(connection, limit=10):
    """
    Get artifact count by culture
    """
    query = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT %s
    """
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query, (limit,))
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)

def get_color_distribution(connection):
    """
    Analyze color usage across artifacts
    """
    query = """
    SELECT c.color, c.spectrum, 
           COUNT(DISTINCT c.objectid) as artifact_count,
           AVG(c.percent) as avg_percent
    FROM artifactcolors c
    GROUP BY c.color, c.spectrum
    ORDER BY artifact_count DESC
    LIMIT 20
    """
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)

def get_media_availability(connection):
    """
    Check image availability by department
    """
    query = """
    SELECT 
        am.department,
        COUNT(DISTINCT am.objectid) as total_artifacts,
        COUNT(DISTINCT m.objectid) as artifacts_with_media,
        ROUND(COUNT(DISTINCT m.objectid) * 100.0 / COUNT(DISTINCT am.objectid), 2) as media_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia m ON am.objectid = m.objectid
    WHERE am.department IS NOT NULL
    GROUP BY am.department
    ORDER BY media_percentage DESC
    """
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

## Streamlit Dashboard

### Basic Dashboard Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                      value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    connection = create_database_connection()
    
    if connection:
        st.success("✅ Database connected")
        
        # Analytics section
        st.header("📊 Analytics")
        
        analysis_type = st.selectbox(
            "Select Analysis",
            ["Artifacts by Culture", "Color Distribution", "Media Availability"]
        )
        
        if analysis_type == "Artifacts by Culture":
            df = get_artifacts_by_culture(connection)
            
            # Display table
            st.dataframe(df)
            
            # Visualization
            fig = px.bar(df, x='culture', y='artifact_count',
                        title='Artifact Count by Culture')
            st.plotly_chart(fig)
        
        elif analysis_type == "Color Distribution":
            df = get_color_distribution(connection)
            st.dataframe(df)
            
            fig = px.scatter(df, x='avg_percent', y='artifact_count',
                           color='spectrum', hover_data=['color'],
                           title='Color Usage Analysis')
            st.plotly_chart(fig)
        
        connection.close()

if __name__ == "__main__":
    main()
```

### Interactive Query Builder

```python
def run_custom_query(connection):
    """
    Allow users to run custom SQL queries
    """
    st.subheader("Custom SQL Query")
    
    query = st.text_area("Enter your SQL query:", height=150)
    
    if st.button("Execute Query"):
        try:
            df = pd.read_sql(query, connection)
            st.success(f"Query returned {len(df)} rows")
            st.dataframe(df)
            
            # Auto-detect numeric columns for visualization
            numeric_cols = df.select_dtypes(include=['int64', 'float64']).columns
            if len(numeric_cols) >= 1:
                st.subheader("Visualization")
                x_col = st.selectbox("X-axis", df.columns)
                y_col = st.selectbox("Y-axis", numeric_cols)
                
                fig = px.bar(df, x=x_col, y=y_col)
                st.plotly_chart(fig)
        except Exception as e:
            st.error(f"Query error: {e}")
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(api_key, max_pages=3):
    """
    Complete ETL pipeline execution
    """
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(api_key, max_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Load
    print("Loading data into database...")
    connection = create_database_connection()
    
    if connection:
        load_metadata(metadata_df, connection)
        # Similar functions for media and colors
        connection.close()
        print("ETL pipeline completed successfully!")
    else:
        print("Database connection failed")
```

## Troubleshooting

**API Rate Limiting**
- Add `time.sleep(1)` between requests
- Use smaller page sizes (50 instead of 100)
- Check API quota in Harvard dashboard

**Database Connection Issues**
- Verify environment variables are set correctly
- Check firewall rules for database host
- Ensure database user has INSERT/SELECT privileges

**Missing Data in Queries**
- Some artifacts may not have all fields (culture, colors, etc.)
- Use `WHERE field IS NOT NULL` to filter
- Use `LEFT JOIN` when combining tables

**Streamlit Performance**
- Use `@st.cache_data` for expensive queries
- Limit result sets with `LIMIT` clauses
- Consider pagination for large datasets

```python
@st.cache_data
def cached_query(query_string):
    """Cache query results to improve performance"""
    connection = create_database_connection()
    df = pd.read_sql(query_string, connection)
    connection.close()
    return df
```
