---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifact collection
  - extract and store Harvard API data in SQL
  - analyze museum artifacts with SQL queries
  - visualize art collection data with Streamlit
  - set up data engineering pipeline for Harvard museums
  - query and analyze art museum metadata
  - build museum artifact data warehouse
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering pipelines using the Harvard Art Museums API. It covers ETL operations, SQL database design, analytics queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:
- Fetches artifact data from Harvard Art Museums API with pagination
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata
- Visualizes insights through Streamlit dashboards with Plotly

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Requirements
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup
Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"
```

### Database Configuration
```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Database Schema

### Create Tables
```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    division VARCHAR(200),
    department VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    dated VARCHAR(200),
    url VARCHAR(500),
    creditline TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    imagetype VARCHAR(100),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    spectrum VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API
```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

def extract_all_artifacts(api_key, max_pages=10):
    """Extract multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            time.sleep(1)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Transform: Normalize JSON Data
```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact metadata into DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'division': artifact.get('division', '')[:200],
            'department': artifact.get('department', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'medium': artifact.get('medium', '')[:500],
            'dated': artifact.get('dated', '')[:200],
            'url': artifact.get('url', '')[:500],
            'creditline': artifact.get('creditline', '')
        })
    
    return pd.DataFrame(metadata)

def transform_media(artifacts):
    """Transform media data"""
    media = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for image in images:
            media.append({
                'objectid': objectid,
                'iiifbaseuri': image.get('iiifbaseuri', '')[:500],
                'baseimageurl': image.get('baseimageurl', '')[:500],
                'imagetype': image.get('imagetype', '')[:100]
            })
    
    return pd.DataFrame(media)

def transform_colors(artifacts):
    """Transform color data"""
    colors = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        color_data = artifact.get('colors', [])
        
        for color in color_data:
            colors.append({
                'objectid': objectid,
                'color': color.get('color', '')[:50],
                'percentage': color.get('percent', 0),
                'spectrum': color.get('spectrum', '')[:50]
            })
    
    return pd.DataFrame(colors)
```

### Load: Insert into Database
```python
def load_metadata(df, connection):
    """Load metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, period, century, division, 
     department, classification, medium, dated, url, creditline)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE 
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} metadata records")

def load_media(df, connection):
    """Load media data into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (objectid, iiifbaseuri, baseimageurl, imagetype)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} media records")

def load_colors(df, connection):
    """Load color data into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (objectid, color, percentage, spectrum)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    print(f"Loaded {cursor.rowcount} color records")
```

## Analytical SQL Queries

### Common Analytics Patterns
```python
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20;
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC;
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN m.objectid IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY media_status;
    """,
    
    "color_distribution": """
        SELECT spectrum, color, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY spectrum, color
        ORDER BY avg_percentage DESC
        LIMIT 15;
    """,
    
    "department_classification": """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND classification IS NOT NULL
        GROUP BY department, classification
        ORDER BY count DESC
        LIMIT 20;
    """,
    
    "artifacts_with_images_and_colors": """
        SELECT 
            a.title,
            a.culture,
            COUNT(DISTINCT m.id) as image_count,
            COUNT(DISTINCT c.id) as color_count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.objectid = m.objectid
        LEFT JOIN artifactcolors c ON a.objectid = c.objectid
        GROUP BY a.objectid, a.title, a.culture
        HAVING image_count > 0 AND color_count > 0
        ORDER BY image_count DESC, color_count DESC
        LIMIT 25;
    """
}

def execute_query(connection, query_name):
    """Execute analytical query and return results"""
    cursor = connection.cursor(dictionary=True)
    query = ANALYTICS_QUERIES.get(query_name)
    
    cursor.execute(query)
    results = cursor.fetchall()
    
    return pd.DataFrame(results)
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
        "Select Page",
        ["ETL Pipeline", "Analytics Dashboard", "Data Exploration"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_exploration_page()

def show_etl_page():
    """ETL pipeline interface"""
    st.header("📥 ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
        
    with col2:
        page_size = st.number_input("Artifacts per page", 10, 100, 100)
    
    if st.button("🚀 Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = extract_all_artifacts(
                os.getenv('HARVARD_API_KEY'),
                max_pages=num_pages
            )
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df = transform_metadata(artifacts)
            media_df = transform_media(artifacts)
            colors_df = transform_colors(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            conn = get_db_connection()
            load_metadata(metadata_df, conn)
            load_media(media_df, conn)
            load_colors(colors_df, conn)
            conn.close()
            st.success("Data loaded to database")

def show_analytics_page():
    """Analytics dashboard interface"""
    st.header("📊 Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = execute_query(conn, query_name)
        conn.close()
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(
                df.head(20),
                x=df.columns[0],
                y=df.columns[1],
                title=f"{query_name.replace('_', ' ').title()}"
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow
```python
def run_complete_etl(api_key, db_connection, max_pages=10):
    """Complete ETL workflow"""
    # Extract
    artifacts = extract_all_artifacts(api_key, max_pages)
    
    # Transform
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    load_metadata(metadata_df, db_connection)
    load_media(media_df, db_connection)
    load_colors(colors_df, db_connection)
    
    return {
        'artifacts': len(artifacts),
        'metadata_rows': len(metadata_df),
        'media_rows': len(media_df),
        'color_rows': len(colors_df)
    }
```

### Error Handling for API Calls
```python
def safe_api_call(api_key, page, retries=3):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            data = fetch_artifacts(api_key, page)
            return data
        except requests.exceptions.RequestException as e:
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time
time.sleep(1)  # Wait 1 second between API calls

# Use session for connection pooling
session = requests.Session()
response = session.get(BASE_URL, params=params)
```

### Database Connection Issues
```python
# Test connection
try:
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    print("Database connection successful")
except Error as e:
    print(f"Database error: {e}")
```

### Memory Management for Large Datasets
```python
# Process in batches
def batch_load(df, connection, batch_size=1000):
    """Load data in batches"""
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        load_metadata(batch, connection)
        print(f"Loaded batch {i//batch_size + 1}")
```

### Handle Missing Data
```python
# Safe data extraction
def safe_get(data, key, default=''):
    """Safely get value from dict"""
    return data.get(key, default) if data.get(key) is not None else default
```

This skill provides comprehensive guidance for building production-ready ETL pipelines with the Harvard Art Museums API, including database design, analytics, and visualization components.
