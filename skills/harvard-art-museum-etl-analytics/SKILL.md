---
name: harvard-art-museum-etl-analytics
description: Build end-to-end data pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit dashboards
triggers:
  - build an ETL pipeline with Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - set up Harvard Art Museums API data collection
  - analyze art museum data with SQL queries
  - visualize artifact collections with Streamlit
  - implement data engineering pipeline for museum data
  - extract and transform Harvard museum API data
  - build museum artifact analytics application
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates production-ready ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Connects to Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts artifact metadata, transforms nested JSON, loads into relational SQL databases
- **SQL Analytics**: 20+ predefined analytical queries for insights on artifacts, media, colors, and cultures
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts
- **Database Design**: Normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

Architecture flow: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
# Create .env file with:
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_database_host
# DB_USER=your_database_user
# DB_PASSWORD=your_database_password
# DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to `.env` file as `HARVARD_API_KEY`

## Configuration

### Database Setup

The application uses MySQL/TiDB Cloud. Configure connection in `.env`:

```bash
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Database Schema

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

def create_database_schema():
    """Create the database schema for Harvard artifacts"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            classification VARCHAR(255),
            century VARCHAR(100),
            dated VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            medium VARCHAR(500),
            technique VARCHAR(500),
            period VARCHAR(255),
            provenance TEXT,
            commentary TEXT,
            url VARCHAR(500),
            accession_number VARCHAR(100)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(1000),
            base_image_url VARCHAR(1000),
            thumbnail_url VARCHAR(1000),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
    print("Database schema created successfully")

create_database_schema()
```

## Core API Integration

### Fetch Artifacts from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        page (int): Page number for pagination
        size (int): Number of records per page (max 100)
    
    Returns:
        dict: JSON response with artifact data
    """
    api_key = os.getenv('HARVARD_API_KEY')
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

# Example usage
data = fetch_artifacts(page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched {len(data['records'])} artifacts")
```

### Paginated Data Collection

```python
import time

def collect_all_artifacts(max_pages=10):
    """
    Collect artifacts with pagination and rate limiting
    
    Args:
        max_pages (int): Maximum number of pages to fetch
    
    Returns:
        list: Combined list of all artifact records
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(page=page, size=100)
            artifacts = data['records']
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
            # Rate limiting - be respectful to API
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract artifact metadata into structured format"""
    metadata = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'provenance': artifact.get('provenance'),
            'commentary': artifact.get('commentary'),
            'url': artifact.get('url'),
            'accession_number': artifact.get('accessionyear')
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def extract_media(artifacts):
    """Extract media/image information"""
    media_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_record = {
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl'),
                'base_image_url': image.get('iiifbaseuri'),
                'thumbnail_url': image.get('thumbnailurl')
            }
            media_data.append(media_record)
    
    return pd.DataFrame(media_data)

def extract_colors(artifacts):
    """Extract color information from artifacts"""
    color_data = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            }
            color_data.append(color_record)
    
    return pd.DataFrame(color_data)
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed data into MySQL database
    
    Args:
        metadata_df (DataFrame): Artifact metadata
        media_df (DataFrame): Media information
        colors_df (DataFrame): Color information
    """
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        cursor = conn.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, classification, century, dated, department, 
                 division, medium, technique, period, provenance, commentary, url, accession_number)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, image_url, base_image_url, thumbnail_url)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color_hex, color_percent)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if conn.is_connected():
            cursor.close()
            conn.close()

# Complete ETL pipeline
artifacts = collect_all_artifacts(max_pages=5)
metadata_df = extract_metadata(artifacts)
media_df = extract_media(artifacts)
colors_df = extract_colors(artifacts)
load_to_database(metadata_df, media_df, colors_df)
```

## SQL Analytics Queries

### Example Analytical Queries

```python
def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Query 1: Top 10 cultures by artifact count
query_cultures = """
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

# Query 3: Department distribution
query_departments = """
    SELECT department, COUNT(*) as total_artifacts
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY total_artifacts DESC
"""

# Query 4: Most common colors across artifacts
query_colors = """
    SELECT color_hex, COUNT(*) as frequency, AVG(color_percent) as avg_percent
    FROM artifactcolors
    GROUP BY color_hex
    ORDER BY frequency DESC
    LIMIT 15
"""

# Query 5: Media availability
query_media = """
    SELECT 
        COUNT(DISTINCT am.id) as artifacts_with_images,
        COUNT(media_id) as total_images
    FROM artifactmetadata am
    JOIN artifactmedia amed ON am.id = amed.artifact_id
"""

# Execute queries
cultures_df = execute_query(query_cultures)
century_df = execute_query(query_century)
departments_df = execute_query(query_departments)
colors_df = execute_query(query_colors)
media_df = execute_query(query_media)
```

## Streamlit Dashboard Application

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    """Main Streamlit application"""
    st.set_page_config(
        page_title="Harvard Art Museum Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museum Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualizations_page()

def show_data_collection_page():
    """Data collection interface"""
    st.header("📥 Data Collection from Harvard API")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_all_artifacts(max_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            # Transform
            metadata_df = extract_metadata(artifacts)
            media_df = extract_media(artifacts)
            colors_df = extract_colors(artifacts)
            
            # Load
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded to database successfully!")

def show_analytics_page():
    """SQL Analytics interface"""
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Departments": query_departments,
        "Popular Colors": query_colors,
        "Media Statistics": query_media
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        query = queries[selected_query]
        st.code(query, language="sql")
        
        df = execute_query(query)
        st.dataframe(df)

def show_visualizations_page():
    """Visualization dashboard"""
    st.header("📈 Data Visualizations")
    
    # Culture distribution
    cultures_df = execute_query(query_cultures)
    fig1 = px.bar(
        cultures_df,
        x='culture',
        y='artifact_count',
        title='Top 10 Cultures by Artifact Count',
        labels={'culture': 'Culture', 'artifact_count': 'Number of Artifacts'}
    )
    st.plotly_chart(fig1, use_container_width=True)
    
    # Century timeline
    century_df = execute_query(query_century)
    fig2 = px.line(
        century_df,
        x='century',
        y='count',
        title='Artifacts by Century',
        markers=True
    )
    st.plotly_chart(fig2, use_container_width=True)
    
    # Department pie chart
    departments_df = execute_query(query_departments)
    fig3 = px.pie(
        departments_df,
        values='total_artifacts',
        names='department',
        title='Department Distribution'
    )
    st.plotly_chart(fig3, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Run the Application

```bash
streamlit run app.py
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(df, table_name, batch_size=1000):
    """Insert data in batches for better performance"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    total_rows = len(df)
    for i in range(0, total_rows, batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch logic here
        conn.commit()
        print(f"Inserted batch {i//batch_size + 1}")
    
    cursor.close()
    conn.close()
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_fetch(page, max_retries=3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            logger.error(f"Attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

## Troubleshooting

### API Rate Limiting

**Problem**: API returns 429 Too Many Requests

**Solution**: Add delays between requests

```python
import time

for page in range(1, 100):
    data = fetch_artifacts(page=page)
    time.sleep(1)  # 1 second delay between requests
```

### Database Connection Issues

**Problem**: "Can't connect to MySQL server"

**Solution**: Verify credentials and connection

```python
def test_db_connection():
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        print("Database connection successful!")
        conn.close()
    except Error as e:
        print(f"Connection failed: {e}")
```

### Missing API Key

**Problem**: "Invalid API key"

**Solution**: Ensure `.env` file exists and is properly loaded

```python
from dotenv import load_dotenv
import os

load_dotenv()

api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Memory Issues with Large Datasets

**Problem**: Out of memory when processing thousands of artifacts

**Solution**: Use chunking and streaming

```python
def process_in_chunks(artifacts, chunk_size=500):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata_df = extract_metadata(chunk)
        load_to_database(metadata_df, pd.DataFrame(), pd.DataFrame())
        print(f"Processed chunk {i//chunk_size + 1}")
```

### Streamlit Caching for Performance

```python
@st.cache_data(ttl=3600)  # Cache for 1 hour
def cached_query(query):
    """Cache SQL query results"""
    return execute_query(query)

# Use in Streamlit app
df = cached_query(query_cultures)
```

This skill equips AI coding agents to build complete data engineering pipelines using the Harvard Art Museums API, from initial data collection through analytics and visualization.
