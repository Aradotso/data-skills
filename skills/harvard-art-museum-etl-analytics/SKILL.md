---
name: harvard-art-museum-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, Streamlit, and SQL
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering workflow with Harvard API
  - set up analytics dashboard for museum artifact data
  - extract and analyze Harvard Art Museums collection
  - build SQL analytics on museum artifact metadata
  - create Streamlit app for Harvard museum data visualization
  - implement ETL for art collection API data
  - query and visualize museum artifact data
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Art Museum ETL Analytics application:
- Collects artifact data from the Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations to transform nested JSON into relational database tables
- Stores structured data in MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and color data
- Visualizes results through interactive Streamlit dashboards with Plotly charts

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

### Environment Variables

Set up your configuration using environment variables or a `.env` file:

```python
import os

# Harvard Art Museums API
API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = 'https://api.harvardartmuseums.org/object'

# Database configuration
DB_HOST = os.getenv('DB_HOST', 'localhost')
DB_PORT = int(os.getenv('DB_PORT', 3306))
DB_USER = os.getenv('DB_USER')
DB_PASSWORD = os.getenv('DB_PASSWORD')
DB_NAME = os.getenv('DB_NAME', 'harvard_artifacts')
```

### Database Setup

Create the database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    division VARCHAR(255),
    contact VARCHAR(255),
    description TEXT,
    provenance TEXT,
    commentary TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key Components

### 1. API Data Extraction

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100, max_pages=10):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        page: Starting page number
        size: Results per page (max 100)
        max_pages: Maximum pages to fetch
    
    Returns:
        List of artifact records
    """
    all_records = []
    
    for current_page in range(page, page + max_pages):
        params = {
            'apikey': api_key,
            'page': current_page,
            'size': size
        }
        
        response = requests.get(API_BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            all_records.extend(records)
            
            # Check if more pages exist
            if len(records) < size:
                break
                
            # Rate limiting
            time.sleep(0.5)
        else:
            print(f"Error fetching page {current_page}: {response.status_code}")
            break
    
    return all_records
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(records):
    """
    Transform raw API records into structured metadata DataFrame
    """
    metadata_list = []
    
    for record in records:
        metadata = {
            'objectid': record.get('objectid'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'dated': record.get('dated'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'creditline': record.get('creditline'),
            'division': record.get('division'),
            'contact': record.get('contact'),
            'description': record.get('description'),
            'provenance': record.get('provenance'),
            'commentary': record.get('commentary'),
            'url': record.get('url')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)


def transform_artifact_media(records):
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for record in records:
        objectid = record.get('objectid')
        images = record.get('images', [])
        
        if images:
            for img in images:
                media = {
                    'objectid': objectid,
                    'iiifbaseuri': img.get('iiifbaseuri'),
                    'baseimageurl': img.get('baseimageurl'),
                    'primaryimageurl': record.get('primaryimageurl')
                }
                media_list.append(media)
        else:
            # Store record even without images
            media = {
                'objectid': objectid,
                'iiifbaseuri': None,
                'baseimageurl': None,
                'primaryimageurl': record.get('primaryimageurl')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)


def transform_artifact_colors(records):
    """
    Extract color data from artifacts
    """
    colors_list = []
    
    for record in records:
        objectid = record.get('objectid')
        colors = record.get('colors', [])
        
        for color_data in colors:
            color = {
                'objectid': objectid,
                'color': color_data.get('color'),
                'spectrum': color_data.get('spectrum'),
                'hue': color_data.get('hue'),
                'percent': color_data.get('percent')
            }
            colors_list.append(color)
    
    return pd.DataFrame(colors_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=DB_HOST,
            port=DB_PORT,
            user=DB_USER,
            password=DB_PASSWORD,
            database=DB_NAME
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None


def load_metadata_to_db(df, connection):
    """
    Batch insert artifact metadata into database
    """
    cursor = connection.cursor()
    
    # Prepare INSERT query with ON DUPLICATE KEY UPDATE
    query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, period, century, dated, classification, 
     department, technique, medium, dimensions, creditline, division, 
     contact, description, provenance, commentary, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), period=VALUES(period)
    """
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()


def load_media_to_db(df, connection):
    """
    Batch insert artifact media data
    """
    cursor = connection.cursor()
    
    query = """
    INSERT INTO artifactmedia (objectid, iiifbaseuri, baseimageurl, primaryimageurl)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()


def load_colors_to_db(df, connection):
    """
    Batch insert artifact color data
    """
    cursor = connection.cursor()
    
    query = """
    INSERT INTO artifactcolors (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

### 4. Analytical SQL Queries

```python
# Sample analytical queries for museum data

ANALYTICAL_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Culture Distribution": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN primaryimageurl IS NOT NULL THEN 'Has Image'
                ELSE 'No Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Classification Types": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts with Provenance": """
        SELECT 
            CASE 
                WHEN provenance IS NOT NULL THEN 'Has Provenance'
                ELSE 'No Provenance'
            END as status,
            COUNT(*) as count
        FROM artifactmetadata
        GROUP BY status
    """
}


def execute_analytical_query(query_name, connection):
    """
    Execute analytical query and return results as DataFrame
    """
    cursor = connection.cursor()
    query = ANALYTICAL_QUERIES.get(query_name)
    
    if not query:
        return None
    
    try:
        cursor.execute(query)
        columns = [desc[0] for desc in cursor.description]
        results = cursor.fetchall()
        df = pd.DataFrame(results, columns=columns)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        cursor.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museum Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museum Data Analytics")
    st.markdown("End-to-end ETL pipeline and analytics dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()


def show_data_collection_page():
    st.header("📥 Data Collection")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv('HARVARD_API_KEY', ''))
    
    col1, col2 = st.columns(2)
    with col1:
        num_pages = st.number_input("Number of Pages", min_value=1, max_value=50, value=5)
    with col2:
        page_size = st.number_input("Records per Page", min_value=10, max_value=100, value=100)
    
    if st.button("🚀 Start ETL Pipeline"):
        if not api_key:
            st.error("Please provide API key")
            return
        
        with st.spinner("Fetching data from API..."):
            records = fetch_artifacts(api_key, page=1, size=page_size, max_pages=num_pages)
            st.success(f"Fetched {len(records)} records")
        
        with st.spinner("Transforming data..."):
            metadata_df = transform_artifact_metadata(records)
            media_df = transform_artifact_media(records)
            colors_df = transform_artifact_colors(records)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            conn = create_db_connection()
            if conn:
                load_metadata_to_db(metadata_df, conn)
                load_media_to_db(media_df, conn)
                load_colors_to_db(colors_df, conn)
                conn.close()
                st.success("✅ ETL Pipeline Complete!")
            else:
                st.error("Database connection failed")


def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        conn = create_db_connection()
        if conn:
            with st.spinner("Executing query..."):
                df = execute_analytical_query(query_name, conn)
                conn.close()
                
                if df is not None and not df.empty:
                    st.subheader("Query Results")
                    st.dataframe(df)
                    
                    # Auto-generate visualization
                    if len(df.columns) == 2:
                        fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                                    title=query_name)
                        st.plotly_chart(fig, use_container_width=True)
                else:
                    st.warning("No results found")
        else:
            st.error("Database connection failed")


def show_visualizations_page():
    st.header("📈 Data Visualizations")
    
    conn = create_db_connection()
    if not conn:
        st.error("Cannot connect to database")
        return
    
    # Department distribution
    st.subheader("Artifact Distribution by Department")
    df = execute_analytical_query("Artifacts by Department", conn)
    if df is not None:
        fig = px.bar(df, x='department', y='artifact_count',
                    title="Top Departments by Artifact Count")
        st.plotly_chart(fig, use_container_width=True)
    
    # Color distribution
    st.subheader("Most Common Colors in Collection")
    df = execute_analytical_query("Most Common Colors", conn)
    if df is not None:
        fig = px.pie(df, names='color', values='frequency',
                    title="Color Distribution")
        st.plotly_chart(fig, use_container_width=True)
    
    conn.close()


if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="localhost"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl_pipeline(api_key, num_pages=10):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("1. Extracting data from API...")
    records = fetch_artifacts(api_key, page=1, size=100, max_pages=num_pages)
    
    # Transform
    print("2. Transforming data...")
    metadata_df = transform_artifact_metadata(records)
    media_df = transform_artifact_media(records)
    colors_df = transform_artifact_colors(records)
    
    # Load
    print("3. Loading to database...")
    conn = create_db_connection()
    if conn:
        load_metadata_to_db(metadata_df, conn)
        load_media_to_db(media_df, conn)
        load_colors_to_db(colors_df, conn)
        conn.close()
        print("ETL pipeline complete!")
    else:
        print("Database connection failed")
```

### Custom Analytics Query

```python
def run_custom_analytics(connection, custom_query):
    """
    Execute custom SQL analytics query
    """
    cursor = connection.cursor()
    
    try:
        cursor.execute(custom_query)
        columns = [desc[0] for desc in cursor.description]
        results = cursor.fetchall()
        return pd.DataFrame(results, columns=columns)
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        cursor.close()
```

## Troubleshooting

### API Rate Limiting

```python
# Add retry logic with exponential backoff
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retries():
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
# Test database connection
def test_db_connection():
    try:
        conn = mysql.connector.connect(
            host=DB_HOST,
            port=DB_PORT,
            user=DB_USER,
            password=DB_PASSWORD,
            database=DB_NAME,
            connection_timeout=10
        )
        if conn.is_connected():
            print("✅ Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Handling Missing Data

```python
# Clean and validate data before insertion
def clean_dataframe(df):
    """
    Clean DataFrame for database insertion
    """
    # Replace NaN with None for SQL NULL
    df = df.where(pd.notnull(df), None)
    
    # Truncate long strings
    for col in df.select_dtypes(include=['object']).columns:
        df[col] = df[col].astype(str).str[:500]
    
    return df
```

### Memory Management for Large Datasets

```python
def fetch_artifacts_chunked(api_key, total_pages=100, chunk_size=10):
    """
    Fetch and process data in chunks to manage memory
    """
    conn = create_db_connection()
    
    for start_page in range(1, total_pages + 1, chunk_size):
        records = fetch_artifacts(api_key, page=start_page, 
                                 size=100, max_pages=chunk_size)
        
        # Transform
        metadata_df = transform_artifact_metadata(records)
        media_df = transform_artifact_media(records)
        colors_df = transform_artifact_colors(records)
        
        # Load
        if conn:
            load_metadata_to_db(metadata_df, conn)
            load_media_to_db(media_df, conn)
            load_colors_to_db(colors_df, conn)
        
        print(f"Processed pages {start_page} to {start_page + chunk_size - 1}")
    
    if conn:
        conn.close()
```

This skill provides AI coding agents with comprehensive knowledge to build ETL pipelines, perform SQL analytics, and create interactive dashboards using the Harvard Art Museums API.
