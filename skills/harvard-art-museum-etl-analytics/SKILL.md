---
name: harvard-art-museum-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, Streamlit, and SQL
triggers:
  - how do I use the Harvard Art Museums API for data engineering
  - build an ETL pipeline with Harvard museum data
  - create analytics dashboard with Harvard artifacts collection
  - query Harvard Art Museums API and load to SQL database
  - set up Streamlit app for museum artifact analytics
  - extract and transform Harvard museum API data
  - build data pipeline for art museum collections
  - analyze Harvard Art Museums data with SQL
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in building end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive data visualization using Streamlit.

## What This Project Does

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:

- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational database tables
- Loads structured data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly charts

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
```

## Running the Application

```bash
# Launch the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Core Components

### 1. API Integration

Extract data from Harvard Art Museums API with pagination:

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
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
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
def collect_artifacts(api_key, total_records=500):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
    
    return all_artifacts[:total_records]
```

### 2. ETL Pipeline

Transform nested JSON into relational tables:

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Extract and transform artifact metadata"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Extract media/image data"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'height': img.get('height'),
                'width': img.get('width'),
                'format': img.get('format')
            })
    
    return pd.DataFrame(media)

def transform_artifact_colors(artifacts):
    """Extract color data"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_list = artifact.get('colors', [])
        
        for color in color_list:
            colors.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Schema

Create normalized SQL tables:

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            medium TEXT,
            dated VARCHAR(200),
            creditline TEXT,
            accessionyear INT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            baseimageurl VARCHAR(500),
            height INT,
            width INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### 4. Load Data to SQL

Batch insert for performance:

```python
def load_metadata(connection, df):
    """Load metadata into database"""
    cursor = connection.cursor()
    
    # Prepare bulk insert
    insert_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, 
         department, division, medium, dated, creditline, accessionyear)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    # Convert DataFrame to list of tuples
    data = df.to_records(index=False).tolist()
    
    # Batch insert
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    return cursor.rowcount

def load_media(connection, df):
    """Load media data into database"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, image_id, baseimageurl, height, width, format)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
    
    return cursor.rowcount
```

### 5. Analytical SQL Queries

Example queries for insights:

```python
ANALYTICAL_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top 10 Most Common Colors": """
        SELECT color, COUNT(*) as usage_count
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Multiple Images": """
        SELECT am.artifact_id, am.title, COUNT(media.image_id) as image_count
        FROM artifactmetadata am
        JOIN artifactmedia media ON am.artifact_id = media.artifact_id
        GROUP BY am.artifact_id, am.title
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Average Color Diversity by Culture": """
        SELECT m.culture, AVG(color_count) as avg_colors
        FROM artifactmetadata m
        JOIN (
            SELECT artifact_id, COUNT(DISTINCT color) as color_count
            FROM artifactcolors
            GROUP BY artifact_id
        ) c ON m.artifact_id = c.artifact_id
        WHERE m.culture IS NOT NULL
        GROUP BY m.culture
        ORDER BY avg_colors DESC
        LIMIT 15
    """
}

def execute_query(connection, query):
    """Execute analytical query and return results"""
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    df = pd.DataFrame(results, columns=columns)
    cursor.close()
    
    return df
```

### 6. Streamlit Dashboard

Build interactive analytics dashboard:

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for configuration
    with st.sidebar:
        st.header("Configuration")
        
        # Data collection
        if st.button("🔄 Collect New Data"):
            with st.spinner("Fetching artifacts from API..."):
                api_key = os.getenv('HARVARD_API_KEY')
                artifacts = collect_artifacts(api_key, total_records=500)
                
                # Transform
                metadata_df = transform_artifact_metadata(artifacts)
                media_df = transform_artifact_media(artifacts)
                colors_df = transform_artifact_colors(artifacts)
                
                # Load to database
                conn = create_database_connection()
                create_tables(conn)
                
                rows = load_metadata(conn, metadata_df)
                st.success(f"✅ Loaded {rows} artifacts")
                
                conn.close()
    
    # Analytics Section
    st.header("📊 Analytics")
    
    # Query selector
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        conn = create_database_connection()
        
        with st.spinner("Executing query..."):
            results_df = execute_query(conn, ANALYTICAL_QUERIES[query_name])
            
            # Display results
            st.subheader("Results")
            st.dataframe(results_df)
            
            # Visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
        
        conn.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
from dotenv import load_dotenv
import os

def run_etl_pipeline():
    """Complete ETL pipeline execution"""
    load_dotenv()
    
    # 1. Extract
    print("Extracting data from API...")
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = collect_artifacts(api_key, total_records=1000)
    
    # 2. Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # 3. Load
    print("Loading to database...")
    conn = create_database_connection()
    create_tables(conn)
    
    metadata_rows = load_metadata(conn, metadata_df)
    media_rows = load_media(conn, media_df)
    colors_rows = load_data_generic(conn, colors_df, 'artifactcolors')
    
    print(f"✅ ETL Complete: {metadata_rows} metadata, {media_rows} media, {colors_rows} colors")
    
    conn.close()
```

### Error Handling for API Requests

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s...")
            time.sleep(wait_time)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=1):
    """Fetch with rate limiting"""
    all_data = []
    
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, page)
        all_data.extend(data.get('records', []))
        time.sleep(delay)  # Respect rate limits
    
    return all_data
```

### Database Connection Issues

```python
def safe_db_connection():
    """Connection with error handling"""
    max_attempts = 3
    
    for attempt in range(max_attempts):
        try:
            conn = create_database_connection()
            if conn and conn.is_connected():
                return conn
        except Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(2)
    
    raise Exception("Could not establish database connection")
```

### Handling NULL Values

```python
def clean_dataframe(df):
    """Clean DataFrame before database insert"""
    # Replace None with empty strings for VARCHAR columns
    df = df.fillna({
        'title': '',
        'culture': '',
        'century': ''
    })
    
    # Replace None with 0 for numeric columns
    df = df.fillna({
        'accessionyear': 0,
        'height': 0,
        'width': 0
    })
    
    return df
```

## Key Features Reference

- **API Pagination**: Automatically handles multi-page API responses
- **Batch Inserts**: Optimized bulk loading with `executemany()`
- **Foreign Keys**: Maintains relational integrity across tables
- **Interactive Queries**: 20+ predefined analytical queries
- **Real-time Visualization**: Auto-generated Plotly charts from query results
- **Environment Config**: Secure credential management with `.env`

This skill enables AI coding agents to build complete data engineering pipelines from API to analytics dashboard using Harvard's museum collections.
