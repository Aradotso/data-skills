---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I use the Harvard Art Museums API for data engineering
  - build an ETL pipeline with Harvard artifacts collection
  - create analytics dashboard with museum artifact data
  - set up SQL database for Harvard Art Museums data
  - visualize art museum data with Streamlit
  - query Harvard artifacts with SQL analytics
  - implement art collection data pipeline
  - analyze museum artifacts with Python and SQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for collecting, transforming, storing, and visualizing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL analytics, and interactive dashboards using Streamlit.

## What It Does

- **Collects** artifact data dynamically from Harvard Art Museums API with pagination
- **Transforms** nested JSON into relational database tables
- **Stores** structured data in MySQL/TiDB Cloud
- **Analyzes** data using predefined SQL queries
- **Visualizes** results through interactive Plotly charts in Streamlit

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Dependencies

```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Database Schema

The project uses three main tables with relational structure:

```sql
-- Artifact Metadata (Main table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']['pages']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, total_pages = fetch_artifacts(api_key, page=1)
```

### Handling Rate Limits

```python
import time

def fetch_all_artifacts(api_key, max_pages=10):
    """
    Fetch multiple pages with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            artifacts, total = fetch_artifacts(api_key, page=page)
            all_artifacts.extend(artifacts)
            
            # Rate limiting - wait between requests
            time.sleep(1)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear')
        })
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('primaryimageurl'),
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri') if artifact.get('images') else None
            })
        
        # Extract color data
        if artifact.get('colors'):
            for color_info in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color_info.get('color'),
                    'spectrum': color_info.get('spectrum'),
                    'percent': color_info.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
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
    Batch insert dataframes into database
    """
    connection = create_database_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata (parent table first)
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, century, classification, department, dated, accessionyear)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        return True
        
    except Error as e:
        print(f"Database insert error: {e}")
        connection.rollback()
        return False
        
    finally:
        cursor.close()
        connection.close()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Top 10 cultures with most artifacts
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Media availability analysis
query_media = """
    SELECT 
        COUNT(DISTINCT am.id) as total_artifacts,
        COUNT(DISTINCT me.artifact_id) as artifacts_with_media,
        ROUND(COUNT(DISTINCT me.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia me ON am.id = me.artifact_id
"""

# Color distribution across artifacts
query_colors = """
    SELECT 
        spectrum,
        COUNT(*) as occurrence,
        ROUND(AVG(percent), 2) as avg_percent
    FROM artifactcolors
    WHERE spectrum IS NOT NULL
    GROUP BY spectrum
    ORDER BY occurrence DESC
"""

# Department-wise classification
query_dept_class = """
    SELECT 
        department,
        classification,
        COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND classification IS NOT NULL
    GROUP BY department, classification
    ORDER BY department, count DESC
"""
```

### Execute Query Function

```python
def execute_analytics_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    connection = create_database_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
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
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "Analytics Dashboard", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_data_collection_page():
    st.header("📥 Data Collection from API")
    
    api_key = st.text_input("Enter Harvard API Key", type="password")
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=100, value=5)
    
    if st.button("Start ETL Process"):
        if not api_key:
            st.error("Please provide API key")
            return
        
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.info(f"Transformed into {len(metadata_df)} records")
        
        with st.spinner("Loading to database..."):
            success = load_to_database(metadata_df, media_df, colors_df)
            if success:
                st.success("✅ ETL process completed successfully!")
            else:
                st.error("❌ Database loading failed")

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures": query_cultures,
        "Century Distribution": query_century,
        "Media Availability": query_media,
        "Color Distribution": query_colors,
        "Department Classification": query_dept_class
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_analytics_query(queries[selected_query])
            
            if df is not None and not df.empty:
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) >= 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1])
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No results found")

if __name__ == "__main__":
    main()
```

### Visualization Examples

```python
def create_culture_chart(df):
    """
    Create interactive bar chart for culture distribution
    """
    fig = px.bar(
        df,
        x='culture',
        y='artifact_count',
        title='Top 10 Cultures by Artifact Count',
        labels={'culture': 'Culture', 'artifact_count': 'Number of Artifacts'},
        color='artifact_count',
        color_continuous_scale='viridis'
    )
    return fig

def create_color_pie_chart(df):
    """
    Create pie chart for color spectrum distribution
    """
    fig = px.pie(
        df,
        values='occurrence',
        names='spectrum',
        title='Color Spectrum Distribution in Artifacts'
    )
    return fig
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard API
HARVARD_API_KEY=your_api_key_from_harvard

# Database Configuration
DB_HOST=gateway01.ap-southeast-1.prod.aws.tidbcloud.com
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=4000
```

### Loading Configuration

```python
from dotenv import load_dotenv
import os

load_dotenv()

config = {
    'api_key': os.getenv('HARVARD_API_KEY'),
    'db_host': os.getenv('DB_HOST'),
    'db_user': os.getenv('DB_USER'),
    'db_password': os.getenv('DB_PASSWORD'),
    'db_name': os.getenv('DB_NAME'),
    'db_port': int(os.getenv('DB_PORT', 3306))
}
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_complete_etl_pipeline(api_key, num_pages=10):
    """
    Run the complete ETL process
    """
    print("Step 1: Extract - Fetching data from API...")
    raw_artifacts = fetch_all_artifacts(api_key, max_pages=num_pages)
    print(f"Extracted {len(raw_artifacts)} artifacts")
    
    print("Step 2: Transform - Processing nested JSON...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
    print(f"Transformed {len(metadata_df)} metadata records")
    
    print("Step 3: Load - Inserting into database...")
    success = load_to_database(metadata_df, media_df, colors_df)
    
    if success:
        print("✅ ETL pipeline completed successfully")
        return True
    else:
        print("❌ ETL pipeline failed")
        return False
```

### Error Handling Pattern

```python
def safe_etl_execution(api_key, num_pages):
    """
    ETL with comprehensive error handling
    """
    try:
        artifacts = fetch_all_artifacts(api_key, num_pages)
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
        return None
    
    try:
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    except Exception as e:
        print(f"Data transformation failed: {e}")
        return None
    
    try:
        load_to_database(metadata_df, media_df, colors_df)
        return len(metadata_df)
    except Error as e:
        print(f"Database loading failed: {e}")
        return None
```

## Troubleshooting

### API Issues

**Problem:** API rate limiting errors

```python
# Solution: Implement exponential backoff
import time

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
```

**Problem:** Invalid API key

```python
# Solution: Validate before processing
def validate_api_key(api_key):
    test_url = "https://api.harvardartmuseums.org/object"
    response = requests.get(test_url, params={'apikey': api_key, 'size': 1})
    return response.status_code == 200
```

### Database Issues

**Problem:** Connection timeout

```python
# Solution: Add connection pooling
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return connection_pool.get_connection()
```

**Problem:** Foreign key constraint errors

```python
# Solution: Insert in correct order with error handling
def safe_insert_with_relations(metadata_df, media_df, colors_df):
    connection = create_database_connection()
    cursor = connection.cursor()
    
    try:
        # Always insert parent records first
        cursor.executemany(
            "INSERT IGNORE INTO artifactmetadata (...) VALUES (...)",
            metadata_df.values.tolist()
        )
        connection.commit()
        
        # Then child records
        cursor.executemany(
            "INSERT IGNORE INTO artifactmedia (...) VALUES (...)",
            media_df.values.tolist()
        )
        connection.commit()
        
    except Error as e:
        connection.rollback()
        raise e
    finally:
        cursor.close()
        connection.close()
```

### Streamlit Issues

**Problem:** Session state persistence

```python
# Solution: Use st.session_state
if 'artifacts_loaded' not in st.session_state:
    st.session_state.artifacts_loaded = False

if st.button("Load Data"):
    # Process data...
    st.session_state.artifacts_loaded = True
```

**Problem:** Large dataframe display

```python
# Solution: Paginate results
def display_paginated_dataframe(df, page_size=50):
    total_pages = len(df) // page_size + 1
    page = st.number_input('Page', min_value=1, max_value=total_pages, value=1)
    
    start_idx = (page - 1) * page_size
    end_idx = start_idx + page_size
    
    st.dataframe(df.iloc[start_idx:end_idx])
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline from command line
python etl_pipeline.py --api-key $HARVARD_API_KEY --pages 20

# Run specific analytics query
python run_analytics.py --query "top_cultures"
```
