---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with museum artifacts
  - set up Harvard API data collection and analytics
  - implement SQL analytics dashboard for art collections
  - extract and visualize Harvard museum data
  - build artifact collection data pipeline
  - create museum data engineering application
  - set up art museum API integration with database
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact data visualization.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database schemas
- Loads structured data into MySQL/TiDB Cloud databases
- Provides 20+ predefined analytical SQL queries
- Visualizes results through interactive Streamlit dashboards with Plotly charts

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

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

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store in environment variable:
```bash
export HARVARD_API_KEY='your_api_key_here'
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud connection parameters:

```python
# Store as environment variables
export DB_HOST='your_database_host'
export DB_USER='your_database_user'
export DB_PASSWORD='your_database_password'
export DB_NAME='harvard_artifacts'
export DB_PORT='3306'
```

### 3. Streamlit Configuration

Create `.streamlit/secrets.toml`:
```toml
[database]
host = "your_database_host"
user = "your_database_user"
password = "your_database_password"
database = "harvard_artifacts"
port = 3306

[api]
harvard_key = "your_api_key"
```

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata
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
    objectnumber VARCHAR(100),
    url TEXT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    imagecount INT,
    videocount INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import os
import time

def fetch_artifacts(api_key, num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        num_records: Total records to fetch
        page_size: Records per API call (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            records = data.get('records', [])
            artifacts.extend(records)
            
            print(f"Fetched page {page}: {len(records)} records")
            
            if not records or len(artifacts) >= num_records:
                break
                
            page += 1
            time.sleep(0.5)  # Rate limiting
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching data: {e}")
            break
    
    return artifacts[:num_records]
```

### Transform: Data Cleaning and Structuring

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured DataFrames for each table.
    
    Args:
        raw_data: List of artifact dictionaries from API
    
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
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear'),
            'objectnumber': artifact.get('objectnumber', '')[:100],
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl', '')[:500],
            'primaryimageurl': artifact.get('primaryimageurl', '')[:500],
            'imagecount': artifact.get('imagecount', 0),
            'videocount': artifact.get('videocount', 0)
        }
        media_list.append(media)
        
        # Extract color information
        for color in artifact.get('colors', []):
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_entry)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load: Batch Insert into SQL

```python
import mysql.connector
from mysql.connector import Error

def create_connection(host, user, password, database):
    """Create MySQL database connection."""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        print("Database connection successful")
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_data_to_sql(metadata_df, media_df, colors_df, connection):
    """
    Load DataFrames into SQL database using batch inserts.
    
    Args:
        metadata_df: Artifact metadata DataFrame
        media_df: Media information DataFrame
        colors_df: Color information DataFrame
        connection: MySQL connection object
    """
    cursor = connection.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         dated, medium, dimensions, creditline, accessionyear, objectnumber, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        print(f"Inserted {len(metadata_values)} metadata records")
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, imagecount, videocount)
        VALUES (%s, %s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        print(f"Inserted {len(media_values)} media records")
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
        print(f"Inserted {len(colors_values)} color records")
        
        connection.commit()
        print("All data loaded successfully")
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import os

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums - Data Engineering & Analytics")
st.markdown("---")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Select Module",
    ["Data Collection", "ETL Pipeline", "SQL Analytics", "Visualizations"]
)

# Initialize connection
@st.cache_resource
def init_connection():
    return mysql.connector.connect(
        host=st.secrets["database"]["host"],
        user=st.secrets["database"]["user"],
        password=st.secrets["database"]["password"],
        database=st.secrets["database"]["database"]
    )

conn = init_connection()
```

### Data Collection Module

```python
if page == "Data Collection":
    st.header("📥 Data Collection from Harvard API")
    
    col1, col2 = st.columns(2)
    with col1:
        num_records = st.number_input("Number of records to fetch", 
                                       min_value=10, max_value=1000, 
                                       value=100, step=10)
    with col2:
        api_key = st.secrets["api"]["harvard_key"]
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts from Harvard API..."):
            artifacts = fetch_artifacts(api_key, num_records)
            st.success(f"✅ Fetched {len(artifacts)} artifacts")
            
            # Preview data
            st.subheader("Data Preview")
            preview_df = pd.DataFrame(artifacts)
            st.dataframe(preview_df.head(10))
            
            # Store in session state for ETL
            st.session_state['raw_artifacts'] = artifacts
```

### ETL Pipeline Module

```python
if page == "ETL Pipeline":
    st.header("⚙️ ETL Pipeline Execution")
    
    if 'raw_artifacts' not in st.session_state:
        st.warning("⚠️ Please fetch data first from Data Collection module")
    else:
        if st.button("Run ETL Pipeline"):
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(
                    st.session_state['raw_artifacts']
                )
                
                st.success("✅ Transformation complete")
                
                # Show table statistics
                col1, col2, col3 = st.columns(3)
                with col1:
                    st.metric("Metadata Records", len(metadata_df))
                with col2:
                    st.metric("Media Records", len(media_df))
                with col3:
                    st.metric("Color Records", len(colors_df))
                
            with st.spinner("Loading data to database..."):
                load_data_to_sql(metadata_df, media_df, colors_df, conn)
                st.success("✅ Data loaded to SQL database")
```

### SQL Analytics Module

```python
if page == "SQL Analytics":
    st.header("📊 SQL Analytics Dashboard")
    
    # Predefined analytical queries
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        "Most Common Colors": """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """,
        "Artifacts with Most Images": """
            SELECT m.title, m.culture, md.imagecount
            FROM artifactmetadata m
            JOIN artifactmedia md ON m.id = md.artifact_id
            WHERE md.imagecount > 0
            ORDER BY md.imagecount DESC
            LIMIT 10
        """,
        "Classification Summary": """
            SELECT classification, COUNT(*) as count, 
                   COUNT(DISTINCT culture) as unique_cultures
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 10
        """,
        "Accession Year Distribution": """
            SELECT accessionyear, COUNT(*) as artifacts_added
            FROM artifactmetadata
            WHERE accessionyear IS NOT NULL
            GROUP BY accessionyear
            ORDER BY accessionyear DESC
            LIMIT 20
        """,
        "Color Spectrum Analysis": """
            SELECT spectrum, COUNT(DISTINCT artifact_id) as artifact_count,
                   AVG(percent) as avg_coverage
            FROM artifactcolors
            WHERE spectrum IS NOT NULL
            GROUP BY spectrum
            ORDER BY artifact_count DESC
        """
    }
    
    query_name = st.selectbox("Select Analysis Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        query = queries[query_name]
        
        # Display query
        with st.expander("View SQL Query"):
            st.code(query, language="sql")
        
        # Execute query
        result_df = pd.read_sql(query, conn)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(result_df, use_container_width=True)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            st.subheader("Visualization")
            x_col = result_df.columns[0]
            y_col = result_df.columns[1]
            
            fig = px.bar(result_df, x=x_col, y=y_col, 
                        title=query_name,
                        labels={x_col: x_col.replace('_', ' ').title(),
                               y_col: y_col.replace('_', ' ').title()})
            st.plotly_chart(fig, use_container_width=True)
```

### Advanced Analytics with Filters

```python
# Custom query builder
st.subheader("🔍 Custom Query Builder")

col1, col2 = st.columns(2)
with col1:
    filter_culture = st.multiselect(
        "Filter by Culture",
        options=pd.read_sql("SELECT DISTINCT culture FROM artifactmetadata WHERE culture IS NOT NULL", conn)['culture'].tolist()
    )
with col2:
    filter_century = st.multiselect(
        "Filter by Century",
        options=pd.read_sql("SELECT DISTINCT century FROM artifactmetadata WHERE century IS NOT NULL", conn)['century'].tolist()
    )

if st.button("Run Custom Analysis"):
    where_clauses = []
    if filter_culture:
        cultures = "', '".join(filter_culture)
        where_clauses.append(f"culture IN ('{cultures}')")
    if filter_century:
        centuries = "', '".join(filter_century)
        where_clauses.append(f"century IN ('{centuries}')")
    
    where_sql = " AND ".join(where_clauses) if where_clauses else "1=1"
    
    custom_query = f"""
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE {where_sql}
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
    
    result_df = pd.read_sql(custom_query, conn)
    st.dataframe(result_df)
```

## Complete ETL Workflow Script

```python
#!/usr/bin/env python3
"""
Complete ETL workflow for Harvard Artifacts
Run from command line: python etl_workflow.py
"""

import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

def main():
    # Configuration
    API_KEY = os.getenv('HARVARD_API_KEY')
    DB_CONFIG = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    # Step 1: Extract
    print("Step 1: Extracting data from Harvard API...")
    artifacts = fetch_artifacts(API_KEY, num_records=500)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Step 2: Transform
    print("\nStep 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    print(f"Transformed into {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
    
    # Step 3: Load
    print("\nStep 3: Loading data to database...")
    conn = create_connection(**DB_CONFIG)
    if conn:
        load_data_to_sql(metadata_df, media_df, colors_df, conn)
        conn.close()
        print("ETL workflow complete!")
    else:
        print("Failed to connect to database")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def incremental_load(api_key, connection, last_object_id=0):
    """
    Load only new artifacts since last run.
    """
    cursor = connection.cursor()
    
    # Get last loaded artifact ID
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    last_id = result[0] if result[0] else 0
    
    # Fetch only new artifacts
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'after': last_id,
        'size': 100
    }
    
    response = requests.get(base_url, params=params)
    new_artifacts = response.json().get('records', [])
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_data_to_sql(metadata_df, media_df, colors_df, connection)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
    
    cursor.close()
```

### Data Quality Validation

```python
def validate_data(metadata_df, media_df, colors_df):
    """
    Validate data quality before loading.
    """
    issues = []
    
    # Check for duplicate IDs
    if metadata_df['id'].duplicated().any():
        issues.append("Duplicate artifact IDs found")
    
    # Check for required fields
    if metadata_df['id'].isnull().any():
        issues.append("Missing artifact IDs")
    
    # Check foreign key integrity
    media_ids = set(media_df['artifact_id'])
    metadata_ids = set(metadata_df['id'])
    if not media_ids.issubset(metadata_ids):
        issues.append("Media records reference non-existent artifacts")
    
    # Check data types
    if not pd.api.types.is_numeric_dtype(metadata_df['accessionyear']):
        issues.append("accessionyear contains non-numeric values")
    
    if issues:
        print("Data quality issues found:")
        for issue in issues:
            print(f"  - {issue}")
        return False
    
    print("✅ Data quality validation passed")
    return True
```

### Error Handling and Retry Logic

```python
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=2):
    """Decorator for retry logic with exponential backoff."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    wait_time = delay * (2 ** attempt)
                    print(f"Attempt {attempt + 1} failed: {e}. Retrying in {wait_time}s...")
                    time.sleep(wait_time)
        return wrapper
    return decorator

@retry_on_failure(max_retries=3, delay=2)
def fetch_with_retry(url, params):
    """Fetch data with automatic retry."""
    response = requests.get(url, params=params, timeout=30)
    response.raise_for_status()
    return response.json()
```

## Troubleshooting

### API Rate Limiting

**Issue:** Getting 429 Too Many Requests errors

**Solution:**
```python
import time

def fetch_with_rate_limit(api_key, num_records, delay=1.0):
    """Add delay between requests to respect rate limits."""
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page, 'size': 100}
        )
        
        if response.status_code == 429:
            retry_after = int(response.headers.get('Retry-After', 60))
            print(f"Rate limited. Waiting {retry_after} seconds...")
            time.sleep(retry_after)
            continue
        
        artifacts.extend(response.json().get('records', []))
        page += 1
        time.sleep(delay)  # Respectful delay
    
    return artifacts
```

### Database Connection Issues

**Issue:** Lost connection to MySQL server during query

**Solution:**
```python
def reconnect_if_needed(connection):
    """Check and reconnect if connection is lost."""
    try:
        connection.ping(reconnect=True, attempts=3, delay=2)
    except Error:
        print("Reconnecting to database...")
        connection = create_connection(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
    return connection
```

### Memory Issues with Large Datasets

**Issue:** Out of memory when processing thousands of records

**Solution:**
```python
def process_in_batches(artifacts, batch_size=100):
    """Process and load data in smaller batches."""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        load_data_to_sql(metadata_df, media_df, colors_df, conn)
        print(f"Processed batch {i//batch_size + 1}: {len(batch)} records")
```

### Streamlit Caching Issues

**Issue:** Stale data displayed in dashboard

**Solution:**
```python
# Clear cache on demand
if st.button("Refresh Data"):
    st.cache_data.clear()
    st.cache_resource.clear()
    st.experimental_rerun()

# Use TTL for automatic cache expiration
@st.cache_data(ttl=3600)  # Cache for 1 hour
def load_dashboard_data():
    return pd.read_sql("SELECT * FROM artifactmetadata LIMIT 1000", conn)
```

### Handling Missing or Null Data

**Issue:** NULL values causing issues in visualizations

**Solution:**
```python
def clean_dataframe(df):
    """Clean DataFrame before visualization."""
    # Replace empty strings with None
    df = df.replace('', None)
    
    # Fill numeric nulls with 0
    numeric_cols = df.select_dtypes(include=['int64', 'float64']).columns
    df[numeric_cols] = df[numeric_cols].fillna(0)
    
    # Fill string nulls with 'Unknown'
    string_cols = df.select_dtypes(include=['object']).columns
    df[string_cols] = df[string_cols].fillna('Unknown')
    
    return df
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY='your_api_key'
export DB_HOST='your_host'
export DB_USER='your_user'
export DB_PASSWORD='your_password'
export DB_NAME='harvard_artifacts'

# Run Streamlit app
streamlit run app.py

# Or run ETL workflow standalone
python etl_workflow.py
```

## Key Performance Optimizations

1. **Batch Inserts**: Use `executemany()` instead of individual inserts
2. **Connection Pooling**: Cache database connections with `@st.cache_resource`
3. **Query Optimization**: Add indexes on frequently queried columns
4. **API Pagination**: Fetch data in optimal page sizes (100 records)
5. **Data Caching**: Cache transformed DataFrames to avoid reprocessing

This skill provides comprehensive guidance for building production-ready ETL pipelines with Harvard Art Museums data, including database design, API integration, analytics, and interactive visualization.
