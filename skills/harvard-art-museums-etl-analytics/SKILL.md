---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL pipelines, SQL storage, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - set up Harvard Art Museums API data collection
  - design SQL schema for artifact metadata
  - visualize museum collection data with Streamlit
  - implement batch data loading for art museum APIs
  - query and analyze Harvard artifacts database
  - build data engineering project with museum API
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipeline construction, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Art Museums ETL Analytics application:
- Extracts artifact data from the Harvard Art Museums API with pagination
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata
- Visualizes insights through interactive Streamlit dashboards with Plotly

Architecture flow: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database credentials
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

def create_database_connection():
    """Establish connection to MySQL/TiDB database"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None
```

### Database Schema

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500),
    dated VARCHAR(200),
    accession_year INT,
    url TEXT,
    INDEX idx_culture (culture),
    INDEX idx_century (century),
    INDEX idx_department (department)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    width INT,
    height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    hue VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import os

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifact data from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        size: Number of records per page (max 100)
        page: Page number for pagination
    
    Returns:
        JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API request error: {e}")
        return None

def collect_all_artifacts(api_key, total_records=1000):
    """Collect artifacts with pagination handling"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(api_key, size=size, page=page)
        
        if not data or 'records' not in data:
            break
        
        all_artifacts.extend(data['records'])
        
        if len(data['records']) < size:
            break
        
        page += 1
    
    return all_artifacts[:total_records]
```

### Transform: Data Normalization

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data to structured metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'dated': artifact.get('dated', '')[:200],
            'accession_year': artifact.get('accessionyear'),
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image information"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl'),
                'media_type': 'image',
                'width': image.get('width'),
                'height': image.get('height')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color palette information"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'percentage': color.get('percent'),
                'hue': color.get('hue', '')[:50]
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Batch Database Insertion

```python
def load_metadata_to_db(df, connection):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, 
     division, medium, technique, dated, accession_year, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(row) for row in df.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()

def load_media_to_db(df, connection):
    """Batch insert media records"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, image_url, media_type, width, height)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

def load_colors_to_db(df, connection):
    """Batch insert color records"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color, percentage, hue)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Analytics Queries

### Common Analytical Patterns

```python
# Top cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 15
"""

# Artifacts by century distribution
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC
"""

# Department distribution
query_departments = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Media availability analysis
query_media = """
SELECT 
    COUNT(DISTINCT am.id) as artifacts_with_media,
    COUNT(DISTINCT m.artifact_id) as total_media_files
FROM artifactmetadata am
LEFT JOIN artifactmedia m ON am.id = m.artifact_id
"""

# Color palette analysis
query_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 20
"""

# Classification by period
query_classification = """
SELECT classification, period, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND period IS NOT NULL
GROUP BY classification, period
ORDER BY count DESC
LIMIT 20
"""

def execute_analytics_query(connection, query):
    """Execute analytical query and return DataFrame"""
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
import os
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

def main():
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    connection = create_database_connection()
    
    if not connection:
        st.error("Database connection failed. Check credentials.")
        return
    
    # Navigation
    page = st.sidebar.selectbox(
        "Select Operation",
        ["Data Collection", "Analytics Dashboard", "Query Builder"]
    )
    
    if page == "Data Collection":
        show_data_collection_page(api_key, connection)
    elif page == "Analytics Dashboard":
        show_analytics_dashboard(connection)
    elif page == "Query Builder":
        show_query_builder(connection)
    
    connection.close()

def show_data_collection_page(api_key, connection):
    """ETL data collection interface"""
    st.header("📥 Data Collection & ETL")
    
    num_records = st.number_input("Number of artifacts to collect", 
                                   min_value=10, max_value=5000, value=100)
    
    if st.button("Start Data Collection"):
        with st.spinner("Collecting artifact data..."):
            artifacts = collect_all_artifacts(api_key, num_records)
            
            if artifacts:
                st.success(f"Collected {len(artifacts)} artifacts")
                
                # Transform
                st.info("Transforming data...")
                df_metadata = transform_artifact_metadata(artifacts)
                df_media = transform_artifact_media(artifacts)
                df_colors = transform_artifact_colors(artifacts)
                
                # Load
                st.info("Loading to database...")
                load_metadata_to_db(df_metadata, connection)
                load_media_to_db(df_media, connection)
                load_colors_to_db(df_colors, connection)
                
                st.success("ETL pipeline completed successfully!")

def show_analytics_dashboard(connection):
    """Pre-built analytics visualizations"""
    st.header("📊 Analytics Dashboard")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Top Cultures")
        df_cultures = execute_analytics_query(connection, query_cultures)
        if not df_cultures.empty:
            fig = px.bar(df_cultures, x='culture', y='artifact_count',
                        title="Artifacts by Culture")
            st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        st.subheader("Century Distribution")
        df_century = execute_analytics_query(connection, query_century)
        if not df_century.empty:
            fig = px.bar(df_century, x='century', y='count',
                        title="Artifacts by Century")
            st.plotly_chart(fig, use_container_width=True)
    
    st.subheader("Department Overview")
    df_dept = execute_analytics_query(connection, query_departments)
    if not df_dept.empty:
        fig = px.pie(df_dept, values='total_artifacts', names='department',
                    title="Department Distribution")
        st.plotly_chart(fig, use_container_width=True)

def show_query_builder(connection):
    """Custom SQL query interface"""
    st.header("🔍 Custom Query Builder")
    
    query = st.text_area("Enter SQL Query", height=150,
                         value="SELECT * FROM artifactmetadata LIMIT 10")
    
    if st.button("Execute Query"):
        df_result = execute_analytics_query(connection, query)
        if not df_result.empty:
            st.dataframe(df_result)
            
            # Auto-visualization for numeric columns
            numeric_cols = df_result.select_dtypes(include=['number']).columns
            if len(numeric_cols) > 0:
                st.subheader("Visualization")
                chart_type = st.selectbox("Chart Type", ["bar", "line", "scatter"])
                
                if chart_type == "bar" and len(df_result.columns) >= 2:
                    fig = px.bar(df_result, x=df_result.columns[0], 
                                y=df_result.columns[1])
                    st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl_pipeline(api_key, db_config, num_records=1000):
    """Complete ETL pipeline execution"""
    # Extract
    print("Extracting data from API...")
    artifacts = collect_all_artifacts(api_key, num_records)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading to database...")
    connection = create_database_connection()
    
    load_metadata_to_db(df_metadata, connection)
    load_media_to_db(df_media, connection)
    load_colors_to_db(df_colors, connection)
    
    connection.close()
    print("ETL pipeline completed!")
```

### Incremental Data Loading

```python
def get_latest_artifact_id(connection):
    """Get the highest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load(api_key, connection):
    """Load only new artifacts"""
    last_id = get_latest_artifact_id(connection)
    
    # Fetch new artifacts with ID filter
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'id': f'>{last_id}'
    }
    
    response = requests.get("https://api.harvardartmuseums.org/object", 
                           params=params)
    new_artifacts = response.json().get('records', [])
    
    # Process and load
    if new_artifacts:
        df_metadata = transform_artifact_metadata(new_artifacts)
        load_metadata_to_db(df_metadata, connection)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, max_retries=3):
    """Handle API rate limits with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            
            if response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
                continue
            
            response.raise_for_status()
            return response.json()
        
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
    
    return None
```

### Database Connection Pooling

```python
from mysql.connector import pooling

def create_connection_pool():
    """Create database connection pool for performance"""
    try:
        pool = pooling.MySQLConnectionPool(
            pool_name="harvard_pool",
            pool_size=5,
            host=os.getenv('DB_HOST'),
            database=os.getenv('DB_NAME'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD')
        )
        return pool
    except Error as e:
        print(f"Connection pool error: {e}")
        return None
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(api_key, total_records, batch_size=100):
    """Process large datasets in batches"""
    connection = create_database_connection()
    
    for offset in range(0, total_records, batch_size):
        page = (offset // batch_size) + 1
        
        artifacts = fetch_artifacts(api_key, size=batch_size, page=page)
        
        if not artifacts:
            break
        
        # Transform and load immediately to free memory
        df_metadata = transform_artifact_metadata(artifacts)
        load_metadata_to_db(df_metadata, connection)
        
        # Clear DataFrame from memory
        del df_metadata
    
    connection.close()
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards with the Harvard Art Museums API, SQL databases, and Streamlit visualization.
