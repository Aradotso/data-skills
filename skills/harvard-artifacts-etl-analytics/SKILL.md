---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines for Harvard Art Museums API with SQL analytics and Streamlit visualization
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit for art data
  - query Harvard museum collection with SQL
  - visualize artifact data with Plotly
  - set up TiDB or MySQL for museum data
  - extract and transform Harvard API JSON data
  - build data engineering pipeline for art collections
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers API integration, ETL pipeline design, SQL database setup, analytical queries, and interactive Streamlit visualization.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App demonstrates a complete data pipeline:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
2. **Transform**: Convert nested JSON into relational tables (metadata, media, colors)
3. **Load**: Batch insert into MySQL/TiDB Cloud with proper foreign key relationships
4. **Analyze**: Execute predefined SQL queries for insights
5. **Visualize**: Display results in interactive Streamlit dashboards with Plotly charts

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
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### API Key Setup

Get your Harvard Art Museums API key:
1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Add to `.env` file

## Database Schema

Create the following tables in your MySQL/TiDB database:

```sql
-- Main artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url TEXT,
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Media/images table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(50),
    base_url TEXT,
    format VARCHAR(50),
    description TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
    INDEX idx_artifact (artifact_id)
);

-- Color information table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_name VARCHAR(100),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
    INDEX idx_artifact (artifact_id)
);
```

## Core API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        page: Page number (1-indexed)
        size: Number of records per page (max 100)
    
    Returns:
        dict: API response with records and pagination info
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Fetch multiple pages
def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts."""
    all_records = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        all_records.extend(data.get('records', []))
        
        # Check if more pages available
        if page >= data.get('info', {}).get('pages', 0):
            break
    
    return all_records
```

### Rate Limiting and Error Handling

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(page, max_retries=3, delay=2):
    """Fetch with exponential backoff retry logic."""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = delay * (2 ** attempt)
                print(f"Error on attempt {attempt + 1}: {e}")
                print(f"Retrying in {wait_time} seconds...")
                time.sleep(wait_time)
            else:
                raise
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def extract_metadata(record):
    """Extract core metadata from artifact record."""
    return {
        'id': record.get('id'),
        'title': record.get('title', 'Unknown'),
        'culture': record.get('culture', 'Unknown'),
        'period': record.get('period', 'Unknown'),
        'century': record.get('century', 'Unknown'),
        'classification': record.get('classification', 'Unknown'),
        'department': record.get('department', 'Unknown'),
        'dated': record.get('dated', 'Unknown'),
        'accession_number': record.get('accessionnumber', 'Unknown'),
        'url': record.get('url', '')
    }

def extract_media(record, artifact_id):
    """Extract media/image information."""
    media_list = []
    images = record.get('images', [])
    
    for img in images:
        media_list.append({
            'artifact_id': artifact_id,
            'media_type': 'image',
            'base_url': img.get('baseimageurl', ''),
            'format': img.get('format', 'unknown'),
            'description': img.get('description', '')
        })
    
    return media_list

def extract_colors(record, artifact_id):
    """Extract color information."""
    color_list = []
    colors = record.get('colors', [])
    
    for color in colors:
        color_list.append({
            'artifact_id': artifact_id,
            'color_hex': color.get('hex', '#000000'),
            'color_name': color.get('color', 'Unknown'),
            'percentage': color.get('percent', 0.0)
        })
    
    return color_list

def transform_artifacts(records):
    """Transform raw API records into structured dataframes."""
    metadata_list = []
    media_list = []
    color_list = []
    
    for record in records:
        artifact_id = record.get('id')
        
        # Extract metadata
        metadata_list.append(extract_metadata(record))
        
        # Extract media
        media_list.extend(extract_media(record, artifact_id))
        
        # Extract colors
        color_list.extend(extract_colors(record, artifact_id))
    
    return {
        'metadata': pd.DataFrame(metadata_list),
        'media': pd.DataFrame(media_list),
        'colors': pd.DataFrame(color_list)
    }
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection."""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def batch_insert_metadata(df):
    """Batch insert metadata into database."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, 
         department, dated, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        conn.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def batch_insert_media(df):
    """Batch insert media records."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, media_type, base_url, format, description)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        conn.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def batch_insert_colors(df):
    """Batch insert color records."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color_hex, color_name, percentage)
        VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    
    try:
        cursor.executemany(insert_query, data)
        conn.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## SQL Analytics Queries

### Common Analytical Queries

```python
def execute_query(query):
    """Execute SQL query and return results as DataFrame."""
    conn = get_db_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    finally:
        conn.close()

# Top cultures by artifact count
QUERY_TOP_CULTURES = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture != 'Unknown'
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 10
"""

# Artifacts by century
QUERY_BY_CENTURY = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY artifact_count DESC
"""

# Department distribution
QUERY_DEPARTMENTS = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    GROUP BY department
    ORDER BY count DESC
"""

# Media availability
QUERY_MEDIA_STATS = """
    SELECT 
        COUNT(DISTINCT artifact_id) as artifacts_with_media,
        COUNT(*) as total_media_items,
        AVG(items_per_artifact) as avg_media_per_artifact
    FROM (
        SELECT artifact_id, COUNT(*) as items_per_artifact
        FROM artifactmedia
        GROUP BY artifact_id
    ) subquery
"""

# Top colors used
QUERY_TOP_COLORS = """
    SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
    FROM artifactcolors
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Artifacts by classification and culture
QUERY_CLASSIFICATION_CULTURE = """
    SELECT 
        classification,
        culture,
        COUNT(*) as count
    FROM artifactmetadata
    WHERE classification != 'Unknown' AND culture != 'Unknown'
    GROUP BY classification, culture
    ORDER BY count DESC
    LIMIT 20
"""
```

## Streamlit Dashboard

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
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select View",
        ["Data Collection", "Analytics Dashboard", "Database Status"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics Dashboard":
        show_analytics()
    else:
        show_database_status()

def show_data_collection():
    """Data collection interface."""
    st.header("📥 Data Collection from API")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 100, 10)
    
    with col2:
        page_size = st.number_input("Records per page", 10, 100, 100)
    
    if st.button("Start ETL Process"):
        with st.spinner("Fetching data from API..."):
            # Fetch data
            records = fetch_all_artifacts(max_pages=num_pages)
            st.success(f"Fetched {len(records)} records")
            
            # Transform
            st.info("Transforming data...")
            dataframes = transform_artifacts(records)
            
            # Load
            st.info("Loading to database...")
            batch_insert_metadata(dataframes['metadata'])
            batch_insert_media(dataframes['media'])
            batch_insert_colors(dataframes['colors'])
            
            st.success("ETL process completed!")

def show_analytics():
    """Analytics dashboard with visualizations."""
    st.header("📊 Analytics Dashboard")
    
    # Query selector
    queries = {
        "Top Cultures": QUERY_TOP_CULTURES,
        "Artifacts by Century": QUERY_BY_CENTURY,
        "Department Distribution": QUERY_DEPARTMENTS,
        "Top Colors": QUERY_TOP_COLORS,
        "Classification & Culture": QUERY_CLASSIFICATION_CULTURE
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(queries[selected_query])
            
            # Display table
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Create visualization
            if len(df.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(
                    df.head(15),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

def show_database_status():
    """Display database statistics."""
    st.header("🗄️ Database Status")
    
    # Count queries
    count_metadata = "SELECT COUNT(*) as count FROM artifactmetadata"
    count_media = "SELECT COUNT(*) as count FROM artifactmedia"
    count_colors = "SELECT COUNT(*) as count FROM artifactcolors"
    
    col1, col2, col3 = st.columns(3)
    
    with col1:
        df = execute_query(count_metadata)
        st.metric("Total Artifacts", df['count'].iloc[0])
    
    with col2:
        df = execute_query(count_media)
        st.metric("Media Records", df['count'].iloc[0])
    
    with col3:
        df = execute_query(count_colors)
        st.metric("Color Records", df['count'].iloc[0])

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Standard run
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8080

# With specific configuration
streamlit run app.py --server.address 0.0.0.0
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID in database."""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    df = execute_query(query)
    return df['max_id'].iloc[0] or 0

def fetch_new_artifacts_only():
    """Fetch only artifacts newer than last loaded."""
    last_id = get_last_artifact_id()
    # Implement pagination starting from last_id
    # This avoids re-processing existing data
```

### Data Quality Checks

```python
def validate_data_quality(df):
    """Validate transformed data before loading."""
    issues = []
    
    # Check for missing IDs
    if df['id'].isnull().any():
        issues.append("Missing artifact IDs found")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append("Duplicate artifact IDs found")
    
    # Check required fields
    required_fields = ['title', 'culture', 'century']
    for field in required_fields:
        null_count = df[field].isnull().sum()
        if null_count > 0:
            issues.append(f"{null_count} nulls in {field}")
    
    return issues
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time

for page in range(1, max_pages):
    data = fetch_artifacts(page)
    time.sleep(0.5)  # 500ms delay between requests
```

### Database Connection Issues
```python
# Test connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Memory Issues with Large Datasets
```python
# Process in smaller chunks
def process_in_chunks(records, chunk_size=1000):
    for i in range(0, len(records), chunk_size):
        chunk = records[i:i + chunk_size]
        dataframes = transform_artifacts(chunk)
        batch_insert_metadata(dataframes['metadata'])
        # Continue with media and colors
```

### Handling Missing Fields
```python
def safe_get(record, key, default='Unknown'):
    """Safely extract value with default fallback."""
    value = record.get(key, default)
    return value if value else default
```

This skill provides comprehensive coverage for building production-ready ETL pipelines with the Harvard Art Museums API, suitable for data engineering portfolios and real-world analytics applications.
