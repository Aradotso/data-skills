---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums data using Python, SQL, and Streamlit
triggers:
  - how do I use the Harvard Art Museums API for data engineering
  - build an ETL pipeline for museum artifacts data
  - create analytics dashboard with Streamlit and museum data
  - extract and load Harvard artifacts into SQL database
  - visualize museum collection data with Plotly
  - set up data pipeline for art museum analytics
  - query and analyze Harvard Art Museums collection
  - transform nested JSON museum data into relational tables
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipeline construction, SQL database design, analytical queries, and interactive Streamlit dashboards for museum collection data.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** collections using 20+ predefined SQL queries
- **Visualizes** insights through interactive Plotly charts in a Streamlit interface

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

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

Get your Harvard API key from: https://www.harvardartmuseums.org/collections/api

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Initialize database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create database schema
def create_tables(conn):
    cursor = conn.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            accessionyear INT,
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            url VARCHAR(500)
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(50),
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    conn.commit()
    cursor.close()
```

## Key API Patterns

### Extract: Fetch Data from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages
def fetch_multiple_pages(num_pages=5):
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
    
    return all_artifacts
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """
    Transform raw API response into normalized DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'baseimageurl': img.get('baseimageurl'),
                    'iiifbaseuri': img.get('iiifbaseuri')
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
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

### Load: Batch Insert into SQL

```python
def load_data_to_sql(metadata_df, media_df, colors_df):
    """
    Batch load DataFrames into SQL database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, accessionyear, technique, medium, dimensions, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """, tuple(row))
    
    # Load media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, iiifbaseuri)
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
    cursor.close()
    conn.close()
```

## Analytics Queries

### Sample SQL Analytics

```python
# Query 1: Top 10 cultures by artifact count
def get_top_cultures():
    conn = get_db_connection()
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Query 2: Artifacts by century
def get_artifacts_by_century():
    conn = get_db_connection()
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Query 3: Most common colors
def get_color_distribution():
    conn = get_db_connection()
    query = """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Query 4: Department statistics
def get_department_stats():
    conn = get_db_connection()
    query = """
        SELECT 
            department,
            COUNT(*) as total_artifacts,
            COUNT(DISTINCT classification) as unique_classifications,
            MIN(accessionyear) as earliest_year,
            MAX(accessionyear) as latest_year
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["ETL Pipeline", "Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_etl_page():
    st.header("ETL Pipeline Control")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_multiple_pages(num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_data_to_sql(metadata_df, media_df, colors_df)
            st.success("Data loaded to SQL database")
        
        # Show preview
        st.subheader("Preview")
        st.dataframe(metadata_df.head())

def show_analytics_page():
    st.header("SQL Analytics")
    
    query_options = {
        "Top 10 Cultures": get_top_cultures,
        "Artifacts by Century": get_artifacts_by_century,
        "Color Distribution": get_color_distribution,
        "Department Statistics": get_department_stats
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        df = query_options[selected_query]()
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations_page():
    st.header("Interactive Visualizations")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Culture Distribution")
        df = get_top_cultures()
        fig = px.pie(df, values='count', names='culture', title='Top Cultures')
        st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        st.subheader("Accession Timeline")
        df = get_artifacts_by_century()
        fig = px.line(df, x='century', y='count', title='Artifacts Over Time')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(num_pages=5):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        # Extract
        artifacts = fetch_multiple_pages(num_pages)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
        
        # Validate
        assert not metadata_df.empty, "Metadata DataFrame is empty"
        
        # Load
        load_data_to_sql(metadata_df, media_df, colors_df)
        
        return {
            'status': 'success',
            'artifacts_processed': len(artifacts),
            'metadata_rows': len(metadata_df),
            'media_rows': len(media_df),
            'colors_rows': len(colors_df)
        }
    
    except requests.exceptions.RequestException as e:
        return {'status': 'error', 'message': f'API error: {str(e)}'}
    except mysql.connector.Error as e:
        return {'status': 'error', 'message': f'Database error: {str(e)}'}
    except Exception as e:
        return {'status': 'error', 'message': f'Unexpected error: {str(e)}'}
```

### Incremental Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_load(num_pages=5):
    """Only load new artifacts not already in database"""
    last_id = get_last_artifact_id()
    artifacts = fetch_multiple_pages(num_pages)
    
    # Filter new artifacts
    new_artifacts = [a for a in artifacts if a.get('id', 0) > last_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifact_data(new_artifacts)
        load_data_to_sql(metadata_df, media_df, colors_df)
        return len(new_artifacts)
    
    return 0
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff on rate limit"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except Exception as e:
            if '429' in str(e):  # Rate limit error
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def verify_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Exception as e:
        print(f"Database connection failed: {e}")
        return False
```

### Missing API Key

```python
def validate_config():
    """Validate required environment variables"""
    required = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']
    missing = [var for var in required if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing required env vars: {', '.join(missing)}")
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement pagination** when fetching large datasets from the API
3. **Batch insert operations** for better database performance
4. **Normalize data** into separate tables (metadata, media, colors)
5. **Add indexes** on frequently queried columns (culture, century, department)
6. **Cache query results** in Streamlit using `@st.cache_data`
7. **Log ETL operations** for debugging and monitoring

This skill enables AI agents to build complete data engineering pipelines using museum API data, SQL databases, and interactive analytics dashboards.
