---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL processing, SQL analytics, and Streamlit visualization
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - set up ETL pipeline for museum artifact data
  - create analytics dashboard with Harvard museum data
  - extract and transform Harvard Art Museums API data
  - build streamlit app for museum artifact analytics
  - configure SQL database for Harvard artifacts collection
  - visualize museum artifact data with plotly
  - implement batch ETL for art museum data
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering and analytics solution using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

**Architecture:** API → ETL → SQL → Analytics → Visualization

Key capabilities:
- Extract artifact data from Harvard Art Museums API with pagination
- Transform nested JSON into relational database schema
- Load data into MySQL/TiDB Cloud with batch inserts
- Execute 20+ analytical SQL queries
- Visualize results in interactive Streamlit dashboards

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

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Store credentials securely using environment variables:

```bash
# .env file (create this, do not commit)
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Configuration

```python
import os
import mysql.connector

# Database connection setup
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

### SQL Schema

```sql
-- Create artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    objectnumber VARCHAR(100),
    accessionyear INT,
    division VARCHAR(255)
);

-- Create artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifact colors table
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

### Extract: API Data Collection

```python
import requests
import time

def extract_artifacts(api_key, pages=5, size=100):
    """
    Extract artifact data from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key from environment
        pages: Number of pages to fetch
        size: Records per page (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}/{pages}: {len(artifacts)} artifacts")
            
            # Rate limiting - be respectful to API
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return all_artifacts
```

### Transform: Data Normalization

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """
    Transform nested JSON into relational dataframes
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'objectnumber': artifact.get('objectnumber'),
            'accessionyear': artifact.get('accessionyear'),
            'division': artifact.get('division')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'imagepermissionlevel': artifact.get('imagepermissionlevel')
        }
        media_list.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color_item in colors:
            color = {
                'artifact_id': artifact.get('id'),
                'color': color_item.get('color'),
                'spectrum': color_item.get('spectrum'),
                'percent': color_item.get('percent')
            }
            colors_list.append(color)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load: Batch Insert to SQL

```python
def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert dataframes into SQL database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, classification, 
         department, objectnumber, accessionyear, division)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    metadata_values = [tuple(row) for row in metadata_df.values]
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
        VALUES (%s, %s, %s, %s)
    """
    
    media_values = [tuple(row) for row in media_df.values]
    cursor.executemany(media_query, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """
        
        colors_values = [tuple(row) for row in colors_df.values]
        cursor.executemany(colors_query, colors_values)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

## Analytics Queries

### Sample SQL Analytics

```python
# Query 1: Artifacts by culture
QUERY_ARTIFACTS_BY_CULTURE = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Century distribution
QUERY_CENTURY_DISTRIBUTION = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Media availability
QUERY_MEDIA_AVAILABILITY = """
    SELECT 
        CASE 
            WHEN primaryimageurl IS NOT NULL THEN 'Has Image'
            ELSE 'No Image'
        END as image_status,
        COUNT(*) as count
    FROM artifactmedia
    GROUP BY image_status
"""

# Query 4: Top colors across artifacts
QUERY_TOP_COLORS = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 10
"""

# Query 5: Department breakdown
QUERY_DEPARTMENT_BREAKDOWN = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
import os

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "SQL Analytics", "Data Visualization"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualization_page()

def show_etl_page():
    """ETL Pipeline execution page"""
    st.header("ETL Pipeline Execution")
    
    col1, col2 = st.columns(2)
    
    with col1:
        pages = st.number_input("Number of Pages", min_value=1, max_value=10, value=2)
    with col2:
        size = st.number_input("Records per Page", min_value=10, max_value=100, value=50)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            artifacts = extract_artifacts(api_key, pages, size)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded successfully!")
        
        # Show sample data
        st.subheader("Sample Metadata")
        st.dataframe(metadata_df.head())

def show_analytics_page():
    """SQL Analytics page"""
    st.header("SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": QUERY_ARTIFACTS_BY_CULTURE,
        "Century Distribution": QUERY_CENTURY_DISTRIBUTION,
        "Media Availability": QUERY_MEDIA_AVAILABILITY,
        "Top Colors": QUERY_TOP_COLORS,
        "Department Breakdown": QUERY_DEPARTMENT_BREAKDOWN
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            df = execute_query(queries[selected_query])
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualization_page():
    """Interactive visualization page"""
    st.header("Data Visualizations")
    
    # Example: Culture distribution
    df_culture = execute_query(QUERY_ARTIFACTS_BY_CULTURE)
    
    col1, col2 = st.columns(2)
    
    with col1:
        fig1 = px.bar(
            df_culture,
            x='culture',
            y='artifact_count',
            title="Top 10 Cultures by Artifact Count",
            color='artifact_count',
            color_continuous_scale='viridis'
        )
        st.plotly_chart(fig1, use_container_width=True)
    
    with col2:
        fig2 = px.pie(
            df_culture,
            values='artifact_count',
            names='culture',
            title="Culture Distribution"
        )
        st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Run the Application

```bash
streamlit run app.py
```

## Common Patterns

### Pagination Handling

```python
def extract_all_artifacts(api_key, max_records=1000):
    """Extract artifacts with automatic pagination"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        params = {
            'apikey': api_key,
            'size': size,
            'page': page
        }
        
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params=params
        )
        
        if response.status_code != 200:
            break
        
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
        
        all_artifacts.extend(records)
        page += 1
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

### Error Handling in ETL

```python
def safe_extract_field(artifact, field, default=None):
    """Safely extract field with fallback"""
    try:
        return artifact.get(field, default)
    except (KeyError, AttributeError):
        return default

def robust_transform(artifacts):
    """Transform with error handling"""
    metadata_list = []
    
    for artifact in artifacts:
        try:
            metadata = {
                'id': safe_extract_field(artifact, 'id'),
                'title': safe_extract_field(artifact, 'title', 'Unknown'),
                'culture': safe_extract_field(artifact, 'culture'),
                # ... other fields
            }
            metadata_list.append(metadata)
        except Exception as e:
            print(f"Error processing artifact {artifact.get('id')}: {e}")
            continue
    
    return pd.DataFrame(metadata_list)
```

## Troubleshooting

### API Rate Limiting

If you encounter 429 errors:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retry():
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues

```python
import mysql.connector
from mysql.connector import Error

def safe_db_connection():
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        return conn
    except Error as e:
        st.error(f"Database connection failed: {e}")
        return None
```

### Memory Management for Large Datasets

```python
def batch_load_artifacts(artifacts, batch_size=1000):
    """Load artifacts in batches to manage memory"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Loaded batch {i//batch_size + 1}")
```

### Handling Missing Data

```python
# Clean dataframe before loading
metadata_df = metadata_df.fillna({
    'title': 'Untitled',
    'culture': 'Unknown',
    'century': 'Unknown',
    'accessionyear': 0
})

# Or drop rows with critical missing fields
metadata_df = metadata_df.dropna(subset=['id', 'title'])
```

## Advanced Usage

### Custom Query Builder

```python
def build_dynamic_query(table, filters, group_by=None, limit=None):
    """Build SQL query dynamically"""
    query = f"SELECT * FROM {table} WHERE 1=1"
    
    for field, value in filters.items():
        if isinstance(value, str):
            query += f" AND {field} = '{value}'"
        else:
            query += f" AND {field} = {value}"
    
    if group_by:
        query += f" GROUP BY {group_by}"
    
    if limit:
        query += f" LIMIT {limit}"
    
    return query
```

### Export Results

```python
def export_to_csv(df, filename):
    """Export dataframe to CSV"""
    df.to_csv(filename, index=False)
    st.success(f"Exported to {filename}")

# In Streamlit
if st.button("Download Results"):
    csv = df.to_csv(index=False).encode('utf-8')
    st.download_button(
        label="Download CSV",
        data=csv,
        file_name="harvard_artifacts.csv",
        mime="text/csv"
    )
```
