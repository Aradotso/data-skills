---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards for museum artifact data using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data engineering app with museum artifact data
  - set up Harvard artifacts collection analytics dashboard
  - implement SQL analytics for art museum data
  - build streamlit dashboard for Harvard Art Museums
  - extract and transform Harvard museum API data
  - create relational database from art museum JSON
  - visualize museum artifact data with Python
---

# Harvard Artifacts Collection Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL patterns, SQL database design, and interactive analytics dashboards using Streamlit.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper schema design
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly dashboards via Streamlit

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

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```bash
# Create .env file
echo "HARVARD_API_KEY=your_api_key_here" > .env
```

### Database Configuration

Set up MySQL/TiDB connection:

```python
# config.py or .env
DB_HOST=your_host
DB_PORT=4000
DB_USER=your_user
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    provenance TEXT,
    url VARCHAR(500),
    lastupdate DATETIME
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagepermissionlevel INT,
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

## ETL Pipeline Implementation

### Extract: Fetching Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: API key from environment
        page: Page number for pagination
        size: Records per page (max 100)
    
    Returns:
        JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

def collect_all_artifacts(api_key, max_records=1000):
    """
    Collect multiple pages of artifacts with pagination
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=size)
        
        records = data.get('records', [])
        if not records:
            break
            
        all_artifacts.extend(records)
        
        # Check if more pages available
        if len(all_artifacts) >= data.get('info', {}).get('totalrecords', 0):
            break
            
        page += 1
        
    return all_artifacts[:max_records]
```

### Transform: Normalize JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON artifacts into normalized dataframes
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
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
            'classification': artifact.get('classification'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'provenance': artifact.get('provenance'),
            'url': artifact.get('url'),
            'lastupdate': artifact.get('lastupdate')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if artifact.get('primaryimageurl') or artifact.get('baseimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'primaryimageurl': artifact.get('primaryimageurl'),
                'imagepermissionlevel': artifact.get('imagepermissionlevel', 0)
            }
            media_list.append(media)
        
        # Extract color information
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_record)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL/TiDB database connection"""
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

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed dataframes into SQL database
    """
    connection = create_database_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata (batch insert for performance)
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, classification, century, dated, department, 
             division, technique, medium, dimensions, creditline, provenance, url, lastupdate)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE lastupdate=VALUES(lastupdate)
        """
        
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, primaryimageurl, imagepermissionlevel)
            VALUES (%s, %s, %s, %s)
        """
        
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
        
    finally:
        cursor.close()
        connection.close()
```

## Analytics Queries

### Example SQL Queries for Insights

```python
# Sample analytical queries dictionary
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Color Analysis": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN primaryimageurl IS NOT NULL THEN 'Has Primary Image'
                ELSE 'No Primary Image'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "Most Common Techniques": """
        SELECT technique, COUNT(*) as count
        FROM artifactmetadata
        WHERE technique IS NOT NULL
        GROUP BY technique
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_query(query_name):
    """Execute analytical query and return dataframe"""
    connection = create_database_connection()
    cursor = connection.cursor()
    
    query = ANALYTICS_QUERIES[query_name]
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    cursor.close()
    connection.close()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    """ETL pipeline interface"""
    st.header("📥 Data Collection & ETL")
    
    api_key = st.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))
    num_records = st.number_input("Number of Records", min_value=10, max_value=10000, value=100)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_all_artifacts(api_key, max_records=num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            success = load_to_database(metadata_df, media_df, colors_df)
            if success:
                st.success("✅ ETL Pipeline completed successfully!")
            else:
                st.error("❌ Failed to load data")

def show_analytics():
    """SQL analytics interface"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(query_name)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """Interactive visualizations"""
    st.header("📈 Data Visualizations")
    
    viz_type = st.selectbox("Visualization Type", 
                            ["Culture Distribution", "Color Analysis", "Timeline"])
    
    df = execute_query(ANALYTICS_QUERIES[viz_type])
    
    if viz_type == "Culture Distribution":
        fig = px.pie(df, names='culture', values='count', 
                     title='Artifacts by Culture')
    elif viz_type == "Color Analysis":
        fig = px.bar(df, x='color', y='frequency', 
                     title='Color Frequency in Artifacts')
    
    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, num_records):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        artifacts = collect_all_artifacts(api_key, max_records=num_records)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate data
        if metadata_df.empty:
            raise ValueError("Metadata transformation failed")
        
        # Load
        success = load_to_database(metadata_df, media_df, colors_df)
        
        return {
            'success': success,
            'records_processed': len(metadata_df),
            'media_records': len(media_df),
            'color_records': len(colors_df)
        }
        
    except requests.RequestException as e:
        return {'success': False, 'error': f'API Error: {str(e)}'}
    except Error as e:
        return {'success': False, 'error': f'Database Error: {str(e)}'}
    except Exception as e:
        return {'success': False, 'error': f'Unexpected Error: {str(e)}'}
```

### Incremental Data Loading

```python
def get_last_update_time():
    """Get timestamp of last data update"""
    connection = create_database_connection()
    cursor = connection.cursor()
    
    query = "SELECT MAX(lastupdate) FROM artifactmetadata"
    cursor.execute(query)
    result = cursor.fetchone()[0]
    
    cursor.close()
    connection.close()
    
    return result

def fetch_new_artifacts_only(api_key, last_update):
    """Fetch only artifacts updated since last ETL run"""
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'lastupdate',
        'sortorder': 'desc'
    }
    
    # Filter by update time if available
    if last_update:
        params['updatedafter'] = last_update
    
    response = requests.get("https://api.harvardartmuseums.org/object", params=params)
    return response.json()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """Fetch data with rate limiting to avoid API throttling"""
    time.sleep(delay)  # Wait between requests
    return fetch_artifacts(api_key, page=page)
```

### Database Connection Issues

```python
def test_database_connection():
    """Test database connectivity"""
    try:
        connection = create_database_connection()
        if connection and connection.is_connected():
            print("✅ Database connection successful")
            connection.close()
            return True
        else:
            print("❌ Failed to connect to database")
            return False
    except Error as e:
        print(f"❌ Connection error: {e}")
        return False
```

### Memory Optimization for Large Datasets

```python
def batch_process_artifacts(api_key, total_records, batch_size=1000):
    """Process large datasets in batches to avoid memory issues"""
    num_batches = (total_records + batch_size - 1) // batch_size
    
    for batch_num in range(num_batches):
        start_page = batch_num * (batch_size // 100) + 1
        
        # Fetch batch
        artifacts = collect_all_artifacts(api_key, max_records=batch_size)
        
        # Transform and load immediately
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        
        print(f"Processed batch {batch_num + 1}/{num_batches}")
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```
