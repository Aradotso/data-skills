---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts
  - fetch and analyze Harvard Art Museums collection
  - set up data pipeline with museum API
  - visualize Harvard museum artifacts with SQL
  - implement ETL for art collection data
  - query Harvard Art Museums database
  - create Streamlit app for museum analytics
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytics queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for data exploration

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://hvrd.art/api)

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
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Required Python Packages
```python
# requirements.txt
streamlit>=1.25.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.1.0
plotly>=5.15.0
python-dotenv>=1.0.0
```

## Configuration

### Database Setup

Create the database schema:

```python
import mysql.connector
import os

def setup_database():
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    cursor = conn.cursor()
    
    # Create database
    cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
    cursor.execute(f"USE {os.getenv('DB_NAME')}")
    
    # Create artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            period VARCHAR(200),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            url VARCHAR(500),
            credit_line TEXT
        )
    """)
    
    # Create artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(100),
            image_url VARCHAR(1000),
            caption TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Create artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()

setup_database()
```

### API Configuration

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org"

def get_api_config():
    return {
        'apikey': API_KEY,
        'size': 100,  # Records per page
        'page': 1
    }
```

## Core ETL Functions

### 1. Extract Data from API

```python
import requests
import time

def extract_artifacts(num_pages=5):
    """Extract artifact data from Harvard Art Museums API"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': os.getenv('HARVARD_API_KEY'),
            'size': 100,
            'page': page
        }
        
        try:
            response = requests.get(
                f"{BASE_URL}/object",
                params=params,
                timeout=30
            )
            response.raise_for_status()
            data = response.json()
            
            all_artifacts.extend(data.get('records', []))
            
            # Rate limiting - respect API limits
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            continue
    
    return all_artifacts
```

### 2. Transform Data

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform raw API data into structured DataFrames"""
    
    # Transform metadata
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
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'url': artifact.get('url'),
            'credit_line': artifact.get('creditline')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        images = artifact.get('images', [])
        for image in images:
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'image_url': image.get('baseimageurl'),
                'caption': image.get('caption')
            }
            media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
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
```

### 3. Load Data into SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        cursor = conn.cursor()
        
        # Load metadata (using INSERT IGNORE to handle duplicates)
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT IGNORE INTO artifactmetadata 
                (id, title, culture, century, classification, department, 
                 dated, period, technique, medium, dimensions, url, credit_line)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, media_type, image_url, caption)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, percent)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if conn.is_connected():
            cursor.close()
            conn.close()
```

## Analytics Queries

### Sample SQL Analytics Functions

```python
def run_analytics_query(query_name):
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' 
                     ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY media_status
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """
    }
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Collection Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "Analytics Dashboard", "Data Visualization"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_visualization_page()

def show_etl_page():
    st.header("🔄 ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 100, 5)
    
    with col2:
        st.write("")
        st.write("")
        if st.button("Run ETL Pipeline", type="primary"):
            with st.spinner("Extracting data from API..."):
                artifacts = extract_artifacts(num_pages)
                st.success(f"Extracted {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                st.success("Data transformed successfully")
            
            with st.spinner("Loading to database..."):
                load_to_database(metadata_df, media_df, colors_df)
                st.success("Data loaded to database")

def show_analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Media Availability": "media_availability",
        "Top Colors Used": "top_colors",
        "Department Distribution": "department_distribution"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        query_key = query_options[selected_query]
        df = run_analytics_query(query_key)
        
        st.subheader("Query Results")
        st.dataframe(df, use_container_width=True)
        
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
    st.header("📈 Interactive Visualizations")
    
    # Add custom visualization logic here
    pass

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def complete_etl_workflow(num_pages=10):
    """Run complete ETL pipeline"""
    print("Starting ETL pipeline...")
    
    # Step 1: Extract
    print(f"Extracting {num_pages} pages from API...")
    raw_artifacts = extract_artifacts(num_pages)
    print(f"Extracted {len(raw_artifacts)} artifacts")
    
    # Step 2: Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
    print(f"Transformed: {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} colors")
    
    # Step 3: Load
    print("Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    print("ETL pipeline completed successfully!")
    
    return metadata_df, media_df, colors_df
```

### Batch Processing with Error Handling

```python
def batch_etl_with_retry(total_pages=100, batch_size=10, max_retries=3):
    """Process large datasets in batches with retry logic"""
    
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        retries = 0
        while retries < max_retries:
            try:
                print(f"Processing batch: pages {batch_start}-{batch_end-1}")
                
                artifacts = []
                for page in range(batch_start, batch_end):
                    page_data = extract_artifacts_single_page(page)
                    artifacts.extend(page_data)
                
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                load_to_database(metadata_df, media_df, colors_df)
                
                print(f"Batch {batch_start}-{batch_end-1} completed")
                break
                
            except Exception as e:
                retries += 1
                print(f"Batch failed (attempt {retries}/{max_retries}): {e}")
                if retries >= max_retries:
                    print(f"Skipping batch {batch_start}-{batch_end-1} after {max_retries} failures")
                time.sleep(2 ** retries)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for rate limits
import time
from requests.exceptions import HTTPError

def fetch_with_backoff(url, params, max_retries=5):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
def get_db_connection(max_retries=3):
    """Get database connection with retry logic"""
    for attempt in range(max_retries):
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
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2)
    raise Exception("Could not connect to database")
```

### Handling Missing Data
```python
def safe_extract(artifact, key, default=None):
    """Safely extract nested data with default values"""
    value = artifact.get(key, default)
    return value if value else default

def transform_with_validation(raw_artifacts):
    """Transform data with validation and null handling"""
    metadata_list = []
    
    for artifact in raw_artifacts:
        if not artifact.get('id'):
            continue  # Skip artifacts without ID
        
        metadata = {
            'id': artifact.get('id'),
            'title': safe_extract(artifact, 'title', 'Untitled'),
            'culture': safe_extract(artifact, 'culture', 'Unknown'),
            'century': safe_extract(artifact, 'century', 'Unknown'),
            # ... other fields
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)
```

### Memory Management for Large Datasets
```python
def stream_etl_pipeline(total_pages=1000, chunk_size=50):
    """Process large datasets in chunks to manage memory"""
    
    for chunk_start in range(1, total_pages + 1, chunk_size):
        chunk_end = min(chunk_start + chunk_size, total_pages + 1)
        
        # Process chunk
        artifacts = extract_artifacts_range(chunk_start, chunk_end)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        
        # Clear memory
        del artifacts, metadata_df, media_df, colors_df
        
        print(f"Processed pages {chunk_start}-{chunk_end-1}")
```

This skill provides comprehensive guidance for building ETL pipelines with the Harvard Art Museums API, implementing SQL analytics, and creating interactive Streamlit dashboards for data visualization.
