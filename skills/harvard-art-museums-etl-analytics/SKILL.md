---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit visualization dashboards
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering workflow with museum artifacts
  - set up analytics dashboard for Harvard Art Museums API
  - extract and transform Harvard museum collection data
  - build SQL analytics for art museum datasets
  - create interactive visualizations for museum artifact data
  - implement data pipeline with Streamlit and museum APIs
  - analyze Harvard Art Museums collection with Python
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering and analytics application that demonstrates real-world ETL pipelines using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases, and visualizes insights through interactive Streamlit dashboards.

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

**Key Components:**
- API integration with Harvard Art Museums
- ETL pipeline with pagination and rate limiting
- Relational database design (MySQL/TiDB Cloud)
- SQL analytics with 20+ predefined queries
- Interactive Plotly visualizations in Streamlit

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

# Run the Streamlit application
streamlit run app.py
```

**Required Dependencies:**
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

Obtain your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org"
```

### Database Configuration

```python
import mysql.connector
from mysql.connector import Error

def create_connection():
    """Create database connection"""
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
```

### Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    alt_text TEXT,
    base_image_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### 1. Extract: Fetch Data from API

```python
import requests
import time

def extract_artifacts(api_key, page=1, size=100, max_pages=10):
    """
    Extract artifact data from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        page: Starting page number
        size: Number of records per page
        max_pages: Maximum pages to fetch
    
    Returns:
        List of artifact dictionaries
    """
    all_artifacts = []
    
    for current_page in range(page, page + max_pages):
        url = f"{BASE_URL}/object"
        params = {
            'apikey': api_key,
            'page': current_page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(url, params=params, timeout=30)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            if not records:
                break
                
            all_artifacts.extend(records)
            
            # Rate limiting
            time.sleep(0.5)
            
            print(f"Fetched page {current_page}, total records: {len(all_artifacts)}")
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {current_page}: {e}")
            break
    
    return all_artifacts
```

### 2. Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured DataFrames
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accession_number': artifact.get('accessionyear'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': img.get('iiifbaseuri'),
                'alt_text': img.get('alttext'),
                'base_image_url': img.get('baseimageurl')
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            }
            colors_list.append(color_entry)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### 3. Load: Insert into SQL Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed data into SQL database using batch inserts
    """
    connection = create_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         dated, medium, dimensions, creditline, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        
        metadata_values = metadata_df.fillna('').values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, alt_text, base_image_url)
        VALUES (%s, %s, %s, %s)
        """
        media_values = media_df.fillna('').values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
        """
        colors_values = colors_df.fillna(0).values.tolist()
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

## SQL Analytics Queries

### Example Analytical Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Top 10 Cultures": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Images' ELSE 'No Images' END as image_status,
            COUNT(*) as artifact_count
        FROM (
            SELECT a.id, COUNT(m.media_id) as media_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) as subquery
        GROUP BY image_status
    """,
    
    "Most Common Colors": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(query_name):
    """Execute analytical query and return results as DataFrame"""
    connection = create_connection()
    if not connection:
        return None
    
    try:
        query = ANALYTICS_QUERIES[query_name]
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        connection.close()
```

## Streamlit Dashboard

### Complete App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_etl_page():
    """ETL pipeline execution interface"""
    st.header("Extract, Transform, Load Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of Pages to Fetch", 1, 50, 5)
        page_size = st.number_input("Records per Page", 10, 100, 50)
    
    with col2:
        if st.button("Run ETL Pipeline"):
            with st.spinner("Extracting data from API..."):
                api_key = os.getenv('HARVARD_API_KEY')
                raw_data = extract_artifacts(api_key, page=1, size=page_size, max_pages=num_pages)
                st.success(f"✅ Extracted {len(raw_data)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(raw_data)
                st.success(f"✅ Transformed into {len(metadata_df)} metadata records")
            
            with st.spinner("Loading to database..."):
                success = load_to_database(metadata_df, media_df, colors_df)
                if success:
                    st.success("✅ Data loaded successfully!")
                else:
                    st.error("❌ Error loading data")
            
            # Show preview
            st.subheader("Data Preview")
            st.dataframe(metadata_df.head(10))

def show_analytics_page():
    """SQL analytics execution interface"""
    st.header("SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            df = execute_query(query_name)
            
            if df is not None and not df.empty:
                st.success(f"✅ Query returned {len(df)} rows")
                
                # Display results
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No results returned")
    
    # Show SQL query
    with st.expander("View SQL Query"):
        st.code(ANALYTICS_QUERIES[query_name], language='sql')

def show_visualizations_page():
    """Custom visualization interface"""
    st.header("Custom Visualizations")
    
    # Example: Culture distribution
    df = execute_query("Top 10 Cultures")
    if df is not None:
        fig = px.pie(df, values='count', names='culture', title='Top Cultures')
        st.plotly_chart(fig, use_container_width=True)
    
    # Example: Century timeline
    df = execute_query("Artifacts by Century")
    if df is not None:
        fig = px.bar(df, x='century', y='artifact_count', title='Artifacts by Century')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def incremental_etl(last_sync_id=0):
    """Load only new artifacts since last sync"""
    connection = create_connection()
    cursor = connection.cursor()
    
    # Get max ID from database
    cursor.execute("SELECT COALESCE(MAX(id), 0) FROM artifactmetadata")
    last_id = cursor.fetchone()[0]
    
    # Fetch only newer records
    url = f"{BASE_URL}/object"
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'q': f'id:>{last_id}',
        'size': 100
    }
    
    response = requests.get(url, params=params)
    new_artifacts = response.json().get('records', [])
    
    return new_artifacts
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_etl_execution():
    """ETL with comprehensive error handling"""
    try:
        logger.info("Starting ETL pipeline")
        raw_data = extract_artifacts(os.getenv('HARVARD_API_KEY'))
        
        if not raw_data:
            logger.warning("No data extracted")
            return False
        
        metadata_df, media_df, colors_df = transform_artifacts(raw_data)
        success = load_to_database(metadata_df, media_df, colors_df)
        
        logger.info(f"ETL completed: {len(metadata_df)} records processed")
        return success
        
    except Exception as e:
        logger.error(f"ETL failed: {str(e)}", exc_info=True)
        return False
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retry():
    session = requests.Session()
    retry = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues
```python
# Use connection pooling
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection_from_pool():
    return connection_pool.get_connection()
```

### Memory Management for Large Datasets
```python
# Use chunking for large data loads
def load_large_dataset_in_chunks(df, chunk_size=1000):
    connection = create_connection()
    for start in range(0, len(df), chunk_size):
        chunk = df.iloc[start:start + chunk_size]
        chunk.to_sql('artifactmetadata', connection, if_exists='append', index=False)
        print(f"Loaded chunk {start // chunk_size + 1}")
```
