---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline for Harvard Art Museums API
  - show me how to use the Harvard artifacts collection app
  - help me create an ETL pipeline with Harvard museum data
  - how to analyze Harvard art museum artifacts with SQL
  - build a streamlit dashboard for museum artifact data
  - extract and transform Harvard Art Museums API data
  - create analytics queries for museum artifact collections
  - set up a data engineering pipeline for art museum data
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end data engineering and analytics application using the Harvard Art Museums API. It covers API integration, ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact data visualization.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Fetches artifact data from the Harvard Art Museums API with pagination handling
- Performs ETL operations to transform nested JSON into relational database tables
- Stores structured data in MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly charts

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Obtain an API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create the database schema:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    primaryimageurl VARCHAR(1000),
    imagecount INT,
    videocount INT,
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

## Core API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(num_pages=5, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        num_pages: Number of pages to fetch
        page_size: Number of artifacts per page (max 100)
    
    Returns:
        List of artifact records
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': API_KEY,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(BASE_URL, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

### Rate Limiting and Error Handling

```python
import time

def fetch_artifacts_with_rate_limit(num_pages=5, delay=1):
    """Fetch artifacts with rate limiting to avoid API throttling."""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': API_KEY,
            'size': 100,
            'page': page
        }
        
        try:
            response = requests.get(BASE_URL, params=params, timeout=30)
            
            if response.status_code == 429:  # Too Many Requests
                print("Rate limited. Waiting 60 seconds...")
                time.sleep(60)
                continue
                
            response.raise_for_status()
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            
            time.sleep(delay)  # Polite delay between requests
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            continue
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract metadata from artifact records."""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dimensions': artifact.get('dimensions', 'Unknown')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """Extract media information from artifact records."""
    media_list = []
    
    for artifact in artifacts:
        media = {
            'artifact_id': artifact.get('id'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'imagecount': artifact.get('totalpageviews', 0),
            'videocount': artifact.get('totalvideos', 0)
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color information from artifact records."""
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

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection."""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_metadata_to_db(df, connection):
    """Load metadata DataFrame to database."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     division, dated, accessionyear, technique, medium, dimensions)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def load_media_to_db(df, connection):
    """Load media DataFrame to database."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, primaryimageurl, imagecount, videocount)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

def load_colors_to_db(df, connection):
    """Load colors DataFrame to database."""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    
    try:
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=5):
    """Execute complete ETL pipeline."""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Step 1: Extracting data from API...")
    artifacts = fetch_artifacts_with_rate_limit(num_pages=num_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Step 2: Transforming data...")
    metadata_df = extract_metadata(artifacts)
    media_df = extract_media(artifacts)
    colors_df = extract_colors(artifacts)
    
    # Load
    print("Step 3: Loading data to database...")
    connection = create_database_connection()
    
    if connection:
        load_metadata_to_db(metadata_df, connection)
        load_media_to_db(media_df, connection)
        load_colors_to_db(colors_df, connection)
        connection.close()
        print("ETL Pipeline completed successfully!")
    else:
        print("Failed to connect to database")
```

## Analytical SQL Queries

### Common Analytics Queries

```python
def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame."""
    try:
        cursor = connection.cursor(dictionary=True)
        cursor.execute(query)
        results = cursor.fetchall()
        cursor.close()
        return pd.DataFrame(results)
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()

# Query 1: Top 10 cultures by artifact count
query_top_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture != 'Unknown'
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century != 'Unknown'
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Department distribution
query_departments = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC;
"""

# Query 4: Average images per classification
query_avg_images = """
SELECT 
    m.classification,
    AVG(a.imagecount) as avg_images,
    COUNT(*) as artifact_count
FROM artifactmetadata m
JOIN artifactmedia a ON m.id = a.artifact_id
GROUP BY m.classification
HAVING artifact_count > 5
ORDER BY avg_images DESC;
"""

# Query 5: Most common colors
query_top_colors = """
SELECT 
    color,
    COUNT(*) as occurrences,
    AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY occurrences DESC
LIMIT 15;
"""

# Query 6: Artifacts with complete data
query_complete_data = """
SELECT 
    m.title,
    m.culture,
    m.century,
    a.imagecount,
    COUNT(c.color_id) as color_count
FROM artifactmetadata m
JOIN artifactmedia a ON m.id = a.artifact_id
LEFT JOIN artifactcolors c ON m.id = c.artifact_id
WHERE a.primaryimageurl IS NOT NULL
GROUP BY m.id, m.title, m.culture, m.century, a.imagecount
HAVING color_count > 0
ORDER BY color_count DESC
LIMIT 20;
"""
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums - Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualization_page()

def show_data_collection_page():
    """Page for ETL pipeline execution."""
    st.header("📊 Data Collection & ETL")
    
    num_pages = st.number_input(
        "Number of pages to fetch",
        min_value=1,
        max_value=50,
        value=5
    )
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline(num_pages=num_pages)
                st.success("ETL pipeline completed successfully!")
            except Exception as e:
                st.error(f"ETL pipeline failed: {e}")

def show_analytics_page():
    """Page for SQL query execution."""
    st.header("🔍 SQL Analytics")
    
    queries = {
        "Top Cultures": query_top_cultures,
        "Artifacts by Century": query_by_century,
        "Department Distribution": query_departments,
        "Average Images per Classification": query_avg_images,
        "Most Common Colors": query_top_colors,
        "Complete Data Artifacts": query_complete_data
    }
    
    selected_query = st.selectbox("Select a query", list(queries.keys()))
    
    if st.button("Execute Query"):
        connection = create_database_connection()
        if connection:
            df = execute_query(connection, queries[selected_query])
            st.dataframe(df, use_container_width=True)
            connection.close()
        else:
            st.error("Failed to connect to database")

def show_visualization_page():
    """Page for interactive visualizations."""
    st.header("📈 Data Visualizations")
    
    connection = create_database_connection()
    
    if connection:
        # Culture distribution chart
        st.subheader("Top Cultures by Artifact Count")
        df_cultures = execute_query(connection, query_top_cultures)
        fig_cultures = px.bar(
            df_cultures,
            x='culture',
            y='artifact_count',
            title="Top 10 Cultures"
        )
        st.plotly_chart(fig_cultures, use_container_width=True)
        
        # Color distribution chart
        st.subheader("Most Common Colors in Artifacts")
        df_colors = execute_query(connection, query_top_colors)
        fig_colors = px.bar(
            df_colors,
            x='color',
            y='occurrences',
            color='avg_percent',
            title="Color Distribution"
        )
        st.plotly_chart(fig_colors, use_container_width=True)
        
        connection.close()
    else:
        st.error("Failed to connect to database")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The application will open at http://localhost:8501
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(df, connection, table_name, batch_size=1000):
    """Insert DataFrame in batches for better performance."""
    cursor = connection.cursor()
    
    for start in range(0, len(df), batch_size):
        batch = df.iloc[start:start + batch_size]
        records = batch.to_records(index=False).tolist()
        
        # Dynamic insert query based on table
        placeholders = ', '.join(['%s'] * len(batch.columns))
        columns = ', '.join(batch.columns)
        query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        try:
            cursor.executemany(query, records)
            connection.commit()
        except Error as e:
            print(f"Batch insert error: {e}")
            connection.rollback()
    
    cursor.close()
```

### Data Quality Checks

```python
def validate_artifacts(df):
    """Validate artifact data before loading."""
    issues = []
    
    # Check for null IDs
    if df['id'].isnull().any():
        issues.append("Found null artifact IDs")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append("Found duplicate artifact IDs")
    
    # Check data types
    if not pd.api.types.is_numeric_dtype(df['accessionyear']):
        issues.append("Invalid accessionyear data type")
    
    if issues:
        print("Data validation issues found:")
        for issue in issues:
            print(f"  - {issue}")
        return False
    
    return True
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
if not API_KEY:
    raise ValueError("HARVARD_API_KEY not found in environment variables")

# Test API connection
def test_api_connection():
    """Test Harvard API connection."""
    params = {'apikey': API_KEY, 'size': 1}
    try:
        response = requests.get(BASE_URL, params=params, timeout=10)
        response.raise_for_status()
        print("API connection successful")
        return True
    except Exception as e:
        print(f"API connection failed: {e}")
        return False
```

### Database Connection Issues

```python
def verify_database_schema():
    """Verify all required tables exist."""
    connection = create_database_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    required_tables = ['artifactmetadata', 'artifactmedia', 'artifactcolors']
    
    for table in required_tables:
        try:
            cursor.execute(f"SELECT 1 FROM {table} LIMIT 1")
            print(f"Table {table} exists")
        except Error:
            print(f"Table {table} not found - run schema creation script")
            return False
    
    cursor.close()
    connection.close()
    return True
```

### Handle Missing Data

```python
def safe_extract(artifact, key, default='Unknown'):
    """Safely extract data with default fallback."""
    value = artifact.get(key, default)
    return value if value else default

# Use in extraction
metadata = {
    'culture': safe_extract(artifact, 'culture'),
    'century': safe_extract(artifact, 'century'),
    'accessionyear': safe_extract(artifact, 'accessionyear', None)
}
```

This skill provides comprehensive guidance for building data engineering pipelines with the Harvard Art Museums API, from API integration through to production-ready analytics dashboards.
