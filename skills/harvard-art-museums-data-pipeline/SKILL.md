---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, Streamlit, and SQL
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - create an ETL workflow for museum artifacts data
  - set up Harvard Art Museums analytics dashboard
  - build a Streamlit app for art collection data
  - extract and analyze Harvard museum data
  - create SQL database for artifact collections
  - visualize museum data with Plotly and Streamlit
  - implement Harvard Art Museums API integration
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact collections.

## What This Project Does

The Harvard Artifacts Collection application:
- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Performs ETL operations to transform nested JSON into relational tables
- Stores data in SQL databases (MySQL/TiDB Cloud)
- Executes analytical SQL queries for insights
- Visualizes results using Plotly in Streamlit dashboards

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to your `.env` file

## Database Setup

### Create Database Schema

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

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
    accessionyear INT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    imagetype VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## API Integration

### Fetch Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_artifacts=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    per_page = 100  # API max per page
    
    while len(all_artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts = data.get('records', [])
            
            if not artifacts:
                break
                
            all_artifacts.extend(artifacts)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return all_artifacts[:num_artifacts]
```

### Handle Rate Limiting

```python
import time

def fetch_with_rate_limit(url, params, delay=0.5):
    """Fetch data with rate limiting to avoid API throttling"""
    response = requests.get(url, params=params)
    time.sleep(delay)  # Wait between requests
    return response
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform nested JSON into structured DataFrames"""
    
    # Metadata extraction
    metadata_list = []
    for artifact in raw_artifacts:
        metadata_list.append({
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
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url')
        })
    
    df_metadata = pd.DataFrame(metadata_list)
    
    # Media extraction
    media_list = []
    for artifact in raw_artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri'),
                'imagetype': img.get('imagetype')
            })
    
    df_media = pd.DataFrame(media_list)
    
    # Colors extraction
    color_list = []
    for artifact in raw_artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    df_colors = pd.DataFrame(color_list)
    
    return df_metadata, df_media, df_colors
```

### Load to SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            port=os.getenv('DB_PORT', 3306)
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_sql(df_metadata, df_media, df_colors):
    """Batch insert DataFrames into SQL tables"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        for _, row in df_metadata.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, dated, medium, dimensions, creditline, 
                 accessionyear, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in df_media.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, iiifbaseuri, imagetype)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in df_colors.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        return True
    except Error as e:
        print(f"SQL insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## Analytical SQL Queries

### Common Analysis Patterns

```python
# Query: Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query: Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query: Most common colors
query_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query: Department distribution
query_departments = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""

# Query: Artifacts with images
query_images = """
    SELECT 
        COUNT(DISTINCT am.artifact_id) as with_images,
        (SELECT COUNT(*) FROM artifactmetadata) as total
    FROM artifactmedia am
"""

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = get_db_connection()
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
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "Analytics Dashboard", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics Dashboard":
        show_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    """Data collection interface"""
    st.header("📥 Data Collection")
    
    num_artifacts = st.number_input(
        "Number of artifacts to fetch",
        min_value=10,
        max_value=1000,
        value=100
    )
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(num_artifacts)
            
            if artifacts:
                df_metadata, df_media, df_colors = transform_artifacts(artifacts)
                
                st.success(f"Fetched {len(artifacts)} artifacts")
                st.write(f"Metadata records: {len(df_metadata)}")
                st.write(f"Media records: {len(df_media)}")
                st.write(f"Color records: {len(df_colors)}")
                
                if st.button("Load to Database"):
                    success = load_to_sql(df_metadata, df_media, df_colors)
                    if success:
                        st.success("✅ Data loaded successfully!")
                    else:
                        st.error("❌ Failed to load data")

def show_analytics():
    """Analytics dashboard"""
    st.header("📊 Analytics Dashboard")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Popular Colors": query_colors,
        "Department Stats": query_departments
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df_result = execute_query(queries[selected_query])
        
        if df_result is not None and not df_result.empty:
            st.dataframe(df_result)
            
            # Auto-generate chart
            if len(df_result.columns) == 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """Interactive visualizations"""
    st.header("📈 Interactive Visualizations")
    
    viz_type = st.selectbox(
        "Visualization Type",
        ["Bar Chart", "Pie Chart", "Scatter Plot"]
    )
    
    # Example: Culture distribution pie chart
    if viz_type == "Pie Chart":
        df = execute_query(query_cultures)
        if df is not None:
            fig = px.pie(
                df,
                names='culture',
                values='artifact_count',
                title='Artifact Distribution by Culture'
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Complete ETL Workflow Example

```python
def full_etl_pipeline(num_artifacts=100):
    """Complete ETL workflow from API to database"""
    
    # Step 1: Extract
    print("Step 1: Extracting data from API...")
    raw_artifacts = fetch_artifacts(num_artifacts)
    print(f"Extracted {len(raw_artifacts)} artifacts")
    
    # Step 2: Transform
    print("Step 2: Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_artifacts)
    print(f"Transformed into {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} color records")
    
    # Step 3: Load
    print("Step 3: Loading to database...")
    success = load_to_sql(df_metadata, df_media, df_colors)
    
    if success:
        print("✅ ETL pipeline completed successfully!")
        return True
    else:
        print("❌ ETL pipeline failed")
        return False

# Run the pipeline
if __name__ == "__main__":
    full_etl_pipeline(200)
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the highest artifact ID in database"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    df = execute_query(query)
    return df['max_id'].iloc[0] if df is not None else 0

def incremental_load():
    """Load only new artifacts"""
    latest_id = get_latest_artifact_id()
    
    # Fetch artifacts with ID > latest_id
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'q': f'id:>{latest_id}',
        'size': 100
    }
    # Continue with fetch and load...
```

### Pattern 2: Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_fetch(url, params, max_retries=3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.error(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Pattern 3: Data Quality Validation

```python
def validate_dataframe(df, required_columns):
    """Validate DataFrame before loading"""
    missing_cols = set(required_columns) - set(df.columns)
    
    if missing_cols:
        raise ValueError(f"Missing columns: {missing_cols}")
    
    # Check for nulls in critical fields
    critical_nulls = df[required_columns].isnull().sum()
    if critical_nulls.any():
        logger.warning(f"Null values found: {critical_nulls[critical_nulls > 0]}")
    
    return True
```

## Troubleshooting

### API Key Issues
- Verify API key is set in `.env` file
- Check API key is valid at Harvard API portal
- Ensure no extra whitespace in environment variable

### Database Connection Errors
```python
# Test connection
def test_db_connection():
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            print("✅ Database connection successful")
            connection.close()
        else:
            print("❌ Connection failed")
    except Error as e:
        print(f"Connection error: {e}")
```

### API Rate Limiting
- Add delays between requests (0.5-1 second)
- Reduce batch size per request
- Monitor API response headers for rate limit info

### Memory Issues with Large Datasets
```python
# Process in chunks
def chunked_load(artifacts, chunk_size=100):
    """Load data in chunks to manage memory"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        df_metadata, df_media, df_colors = transform_artifacts(chunk)
        load_to_sql(df_metadata, df_media, df_colors)
```

### Streamlit Caching for Performance
```python
@st.cache_data(ttl=3600)  # Cache for 1 hour
def cached_query(query):
    """Cache query results"""
    return execute_query(query)
```

This skill enables you to build production-ready data pipelines for museum artifact collections with proper ETL practices, SQL analytics, and interactive visualizations.
