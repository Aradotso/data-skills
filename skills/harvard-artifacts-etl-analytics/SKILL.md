---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - set up data engineering pipeline with Harvard artifacts API
  - create analytics dashboard for museum collection data
  - extract and analyze Harvard Art Museums artifacts
  - build Streamlit app for art museum data visualization
  - implement SQL analytics on Harvard museum collections
  - process Harvard Art Museums API with Python ETL
  - visualize museum artifact data with Plotly
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database schemas
- Loads data into MySQL/TiDB Cloud with proper foreign key relationships
- Executes analytical SQL queries for insights
- Visualizes results through interactive Plotly charts in Streamlit

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema():
    """Initialize database with required tables"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    
    cursor = connection.cursor()
    
    # Create database
    cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
    cursor.execute(f"USE {os.getenv('DB_NAME')}")
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            accession_year INT,
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            credit_line TEXT,
            provenance TEXT,
            description TEXT
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            media_count INT,
            has_images BOOLEAN,
            primary_image_url VARCHAR(1000),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()
    connection.close()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from Harvard API

```python
import requests
import time
from typing import List, Dict

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """Extract artifacts from Harvard Art Museums API"""
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

def collect_multiple_pages(api_key: str, num_pages: int = 5) -> List[Dict]:
    """Collect artifacts across multiple pages with rate limiting"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Collected page {page}: {len(artifacts)} artifacts")
            time.sleep(1)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Convert JSON to Relational Format

```python
import pandas as pd

def transform_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform artifact data into metadata DataFrame"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'accession_year': artifact.get('accessionyear'),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'credit_line': artifact.get('creditline', ''),
            'provenance': artifact.get('provenance', ''),
            'description': artifact.get('description', '')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def transform_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform media information into DataFrame"""
    media_records = []
    
    for artifact in artifacts:
        images = artifact.get('images', [])
        primary_image = artifact.get('primaryimageurl', '')
        
        record = {
            'objectid': artifact.get('objectid'),
            'media_count': len(images),
            'has_images': len(images) > 0,
            'primary_image_url': primary_image[:1000] if primary_image else None
        }
        media_records.append(record)
    
    return pd.DataFrame(media_records)

def transform_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform color data into DataFrame"""
    color_records = []
    
    for artifact in artifacts:
        colors = artifact.get('colors', [])
        
        for color in colors:
            record = {
                'objectid': artifact.get('objectid'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load: Insert Data into SQL Database

```python
def load_to_database(df: pd.DataFrame, table_name: str, connection):
    """Batch insert DataFrame into SQL table"""
    cursor = connection.cursor()
    
    # Prepare INSERT statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    insert_query = f"""
        INSERT IGNORE INTO {table_name} ({columns})
        VALUES ({placeholders})
    """
    
    # Convert DataFrame to list of tuples
    records = [tuple(row) for row in df.values]
    
    # Batch insert
    cursor.executemany(insert_query, records)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} records into {table_name}")
    cursor.close()

def run_etl_pipeline(api_key: str, connection, num_pages: int = 5):
    """Execute complete ETL pipeline"""
    # Extract
    print("Extracting data from API...")
    artifacts = collect_multiple_pages(api_key, num_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Loading data to database...")
    load_to_database(metadata_df, 'artifactmetadata', connection)
    load_to_database(media_df, 'artifactmedia', connection)
    load_to_database(colors_df, 'artifactcolors', connection)
    
    print("ETL pipeline completed successfully!")
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
# Top 10 cultures with most artifacts
query_top_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century distribution
query_century_dist = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Department-wise media availability
query_media_by_dept = """
    SELECT 
        am.department,
        COUNT(*) as total_artifacts,
        SUM(CASE WHEN media.has_images THEN 1 ELSE 0 END) as with_images,
        ROUND(SUM(CASE WHEN media.has_images THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) as image_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia media ON am.objectid = media.objectid
    WHERE am.department IS NOT NULL
    GROUP BY am.department
    ORDER BY total_artifacts DESC
"""

# Most common color spectrums
query_color_analysis = """
    SELECT 
        spectrum,
        COUNT(*) as usage_count,
        ROUND(AVG(percent), 2) as avg_percentage
    FROM artifactcolors
    WHERE spectrum IS NOT NULL
    GROUP BY spectrum
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Artifacts by classification and period
query_classification_period = """
    SELECT 
        classification,
        period,
        COUNT(*) as count
    FROM artifactmetadata
    WHERE classification IS NOT NULL AND period IS NOT NULL
    GROUP BY classification, period
    HAVING count > 5
    ORDER BY count DESC
    LIMIT 20
"""
```

### Execute Queries in Streamlit

```python
import streamlit as st
import plotly.express as px

def execute_analytics_query(query: str, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

def visualize_query_result(df: pd.DataFrame, x_col: str, y_col: str, title: str):
    """Create interactive bar chart from query results"""
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title(), 
                y_col: y_col.replace('_', ' ').title()}
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

# Streamlit app example
st.title("Harvard Artifacts Analytics Dashboard")

# Query selector
queries = {
    "Top Cultures": query_top_cultures,
    "Century Distribution": query_century_dist,
    "Media by Department": query_media_by_dept,
    "Color Analysis": query_color_analysis
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Analysis"):
    result_df = execute_analytics_query(queries[selected_query], connection)
    
    st.dataframe(result_df)
    
    # Auto-visualize if numeric column exists
    if len(result_df.columns) >= 2:
        fig = visualize_query_result(
            result_df,
            result_df.columns[0],
            result_df.columns[1],
            selected_query
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Streamlit Application Structure

```python
# app.py
import streamlit as st
import os
from dotenv import load_dotenv
import mysql.connector

load_dotenv()

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

# Database connection
@st.cache_resource
def get_database_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Sidebar navigation
page = st.sidebar.selectbox(
    "Navigation",
    ["ETL Pipeline", "SQL Analytics", "Visualizations", "Data Explorer"]
)

if page == "ETL Pipeline":
    st.header("ETL Pipeline Control")
    
    num_pages = st.number_input("Number of pages to collect", 1, 20, 5)
    
    if st.button("Run ETL"):
        with st.spinner("Running ETL pipeline..."):
            connection = get_database_connection()
            api_key = os.getenv('HARVARD_API_KEY')
            run_etl_pipeline(api_key, connection, num_pages)
            st.success("ETL completed!")

elif page == "SQL Analytics":
    st.header("SQL Analytics Dashboard")
    # Analytics queries implementation
    
elif page == "Visualizations":
    st.header("Data Visualizations")
    # Visualization implementation

elif page == "Data Explorer":
    st.header("Raw Data Explorer")
    # Data exploration implementation
```

## Common Patterns

### Handling API Pagination

```python
def fetch_all_artifacts(api_key: str, max_records: int = 1000) -> List[Dict]:
    """Fetch artifacts until max_records reached"""
    all_artifacts = []
    page = 1
    page_size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page, page_size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        time.sleep(1)
    
    return all_artifacts[:max_records]
```

### Error Handling in ETL

```python
def safe_load_to_database(df: pd.DataFrame, table_name: str, connection):
    """Load data with error handling and rollback"""
    cursor = connection.cursor()
    
    try:
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        records = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, records)
        connection.commit()
        
        return cursor.rowcount
        
    except Exception as e:
        connection.rollback()
        st.error(f"Database error: {e}")
        return 0
    finally:
        cursor.close()
```

## Troubleshooting

**API Rate Limiting**: Add `time.sleep(1)` between requests. Harvard API allows ~2500 requests/day.

**Database Connection Issues**: Verify `.env` credentials and ensure database server is accessible.

**Large Data Inserts**: Use batch inserts with `executemany()` instead of individual inserts for performance.

**Streamlit Caching**: Use `@st.cache_resource` for database connections and `@st.cache_data` for query results.

**Missing Data Fields**: Always use `.get()` with defaults when accessing API response fields to handle missing data gracefully.
