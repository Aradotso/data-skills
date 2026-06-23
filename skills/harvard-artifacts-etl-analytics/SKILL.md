---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - fetch and analyze Harvard museum collection data
  - build analytics dashboard for art museum data
  - set up SQL database for Harvard artifacts
  - extract transform load pipeline for museum API
  - visualize Harvard Art Museums data with Streamlit
  - design relational schema for artifact metadata
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection application:
- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** data using 20+ predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

Architecture: `API → ETL → SQL → Analytics → Visualization`

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

1. Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api
2. Create a `.env` file in project root:

```bash
HARVARD_API_KEY=your_api_key_here
```

### Database Configuration

```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = 'https://api.harvardartmuseums.org/object'
```

## Database Schema

The application uses three normalized tables:

```sql
-- Artifact Metadata (parent table)
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    division VARCHAR(255)
);

-- Artifact Media (child table)
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    primaryimageurl VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors (child table)
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard API with pagination"""
    url = f"https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API Error on page {page}: {e}")
        return None

def collect_multiple_pages(api_key, num_pages=5):
    """Collect artifacts across multiple pages with rate limiting"""
    all_records = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_records.extend(data['records'])
            time.sleep(1)  # Rate limiting: 1 second between requests
        else:
            break
    
    return all_records
```

### Transform: Normalize JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON into normalized dataframes"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in raw_data:
        artifact_id = record.get('id')
        
        # Extract metadata
        metadata = {
            'id': artifact_id,
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:255],
            'century': record.get('century', '')[:100],
            'classification': record.get('classification', '')[:255],
            'department': record.get('department', '')[:255],
            'dated': record.get('dated', '')[:255],
            'accessionyear': record.get('accessionyear'),
            'technique': record.get('technique', '')[:500],
            'medium': record.get('medium', '')[:500],
            'division': record.get('division', '')[:255]
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact_id,
            'baseimageurl': record.get('baseimageurl', '')[:1000],
            'primaryimageurl': record.get('primaryimageurl', '')[:1000]
        }
        media_list.append(media)
        
        # Extract color data
        if 'colors' in record and record['colors']:
            for color_obj in record['colors']:
                colors_list.append({
                    'artifact_id': artifact_id,
                    'color': color_obj.get('color', '')[:50],
                    'percentage': color_obj.get('percent', 0.0)
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection(db_config):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(**db_config)
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def batch_insert_metadata(connection, df_metadata):
    """Batch insert artifact metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, dated, 
     accessionyear, technique, medium, division)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE 
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(row) for row in df_metadata.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(connection, df_media):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, baseimageurl, primaryimageurl)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_media.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_colors(connection, df_colors):
    """Batch insert color data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, percentage)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_colors.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Complete ETL Pipeline

```python
from config import DB_CONFIG, API_KEY

def run_etl_pipeline(api_key, db_config, num_pages=5):
    """Execute complete ETL pipeline"""
    
    # EXTRACT
    print("Starting extraction...")
    raw_data = collect_multiple_pages(api_key, num_pages)
    print(f"Extracted {len(raw_data)} artifacts")
    
    # TRANSFORM
    print("Starting transformation...")
    df_metadata, df_media, df_colors = transform_artifacts(raw_data)
    print(f"Transformed into {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} color records")
    
    # LOAD
    print("Starting load...")
    connection = create_connection(db_config)
    
    if connection:
        batch_insert_metadata(connection, df_metadata)
        batch_insert_media(connection, df_media)
        batch_insert_colors(connection, df_colors)
        connection.close()
        print("ETL pipeline completed successfully")
    else:
        print("ETL pipeline failed: No database connection")

# Execute pipeline
if __name__ == "__main__":
    run_etl_pipeline(API_KEY, DB_CONFIG, num_pages=10)
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN primaryimageurl IS NOT NULL AND primaryimageurl != '' THEN 'Has Primary Image'
                WHEN baseimageurl IS NOT NULL AND baseimageurl != '' THEN 'Has Base Image Only'
                ELSE 'No Image'
            END as media_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Classification Distribution": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL AND classification != ''
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 12
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as count
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL AND accessionyear > 1800
        GROUP BY accessionyear
        ORDER BY accessionyear ASC
    """,
    
    "Culture-Century Matrix": """
        SELECT culture, century, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND century IS NOT NULL
        GROUP BY culture, century
        HAVING count > 5
        ORDER BY count DESC
        LIMIT 20
    """
}
```

### Execute Queries and Fetch Results

```python
def execute_analytical_query(connection, query_name, query_sql):
    """Execute SQL query and return results as dataframe"""
    try:
        cursor = connection.cursor(dictionary=True)
        cursor.execute(query_sql)
        results = cursor.fetchall()
        cursor.close()
        return pd.DataFrame(results)
    except Error as e:
        print(f"Query execution error for '{query_name}': {e}")
        return pd.DataFrame()
```

## Streamlit Dashboard Application

```python
# app.py
import streamlit as st
import plotly.express as px
from config import DB_CONFIG, API_KEY

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")
st.markdown("**End-to-End Data Engineering & Analytics Application**")

# Sidebar navigation
menu = st.sidebar.selectbox(
    "Navigation",
    ["Home", "ETL Pipeline", "SQL Analytics", "Data Exploration"]
)

if menu == "Home":
    st.header("Project Overview")
    st.write("""
    This application demonstrates:
    - Real-time data extraction from Harvard Art Museums API
    - ETL pipeline processing
    - SQL database management
    - Interactive analytics and visualization
    """)
    
    col1, col2, col3 = st.columns(3)
    with col1:
        st.metric("Data Source", "Harvard API")
    with col2:
        st.metric("Database", "MySQL/TiDB")
    with col3:
        st.metric("Visualization", "Plotly")

elif menu == "ETL Pipeline":
    st.header("🔄 ETL Pipeline Execution")
    
    num_pages = st.slider("Number of API pages to fetch", 1, 20, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            progress_bar = st.progress(0)
            
            # Extract
            st.info("Extracting data from API...")
            raw_data = collect_multiple_pages(API_KEY, num_pages)
            progress_bar.progress(33)
            
            # Transform
            st.info("Transforming data...")
            df_metadata, df_media, df_colors = transform_artifacts(raw_data)
            progress_bar.progress(66)
            
            # Load
            st.info("Loading data into database...")
            connection = create_connection(DB_CONFIG)
            if connection:
                batch_insert_metadata(connection, df_metadata)
                batch_insert_media(connection, df_media)
                batch_insert_colors(connection, df_colors)
                connection.close()
            progress_bar.progress(100)
            
            st.success(f"✅ ETL Complete! Processed {len(raw_data)} artifacts")
            st.write(f"- Metadata records: {len(df_metadata)}")
            st.write(f"- Media records: {len(df_media)}")
            st.write(f"- Color records: {len(df_colors)}")

elif menu == "SQL Analytics":
    st.header("📊 SQL Analytics Dashboard")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        connection = create_connection(DB_CONFIG)
        if connection:
            query_sql = ANALYTICAL_QUERIES[query_name]
            
            # Display query
            with st.expander("View SQL Query"):
                st.code(query_sql, language="sql")
            
            # Execute and display results
            df_results = execute_analytical_query(connection, query_name, query_sql)
            connection.close()
            
            if not df_results.empty:
                st.subheader("Query Results")
                st.dataframe(df_results, use_container_width=True)
                
                # Auto-generate visualization
                if len(df_results.columns) >= 2:
                    st.subheader("Visualization")
                    x_col = df_results.columns[0]
                    y_col = df_results.columns[1]
                    
                    fig = px.bar(
                        df_results.head(15),
                        x=x_col,
                        y=y_col,
                        title=query_name,
                        color=y_col,
                        color_continuous_scale='Blues'
                    )
                    fig.update_layout(xaxis_tickangle=-45)
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No results found")

elif menu == "Data Exploration":
    st.header("🔍 Data Exploration")
    
    connection = create_connection(DB_CONFIG)
    if connection:
        table = st.selectbox("Select Table", ["artifactmetadata", "artifactmedia", "artifactcolors"])
        
        query = f"SELECT * FROM {table} LIMIT 100"
        df = execute_analytical_query(connection, "exploration", query)
        
        st.write(f"**Table:** {table}")
        st.write(f"**Rows displayed:** {len(df)}")
        st.dataframe(df, use_container_width=True)
        
        connection.close()
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_max_artifact_id(connection):
    """Get the highest artifact ID already in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    url = f"https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': last_id
    }
    response = requests.get(url, params=params)
    return response.json().get('records', [])
```

### Pattern 2: Error Handling with Retry Logic

```python
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=2):
    """Decorator for retrying failed API calls"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s...")
                    time.sleep(delay)
            return None
        return wrapper
    return decorator

@retry_on_failure(max_retries=3)
def fetch_artifacts_with_retry(api_key, page):
    """Fetch artifacts with automatic retry on failure"""
    return fetch_artifacts(api_key, page)
```

### Pattern 3: Data Quality Validation

```python
def validate_artifact_data(df_metadata):
    """Validate data quality before loading"""
    issues = []
    
    # Check for required fields
    if df_metadata['id'].isnull().any():
        issues.append("Missing artifact IDs")
    
    # Check for duplicates
    if df_metadata['id'].duplicated().any():
        issues.append("Duplicate artifact IDs found")
    
    # Check field lengths
    if (df_metadata['title'].str.len() > 500).any():
        issues.append("Title exceeds max length")
    
    if issues:
        print("Data quality issues:", issues)
        return False
    return True
```

## Troubleshooting

### Issue: API Rate Limiting

**Problem:** Getting 429 (Too Many Requests) errors

**Solution:**
```python
import time

def fetch_with_rate_limit(api_key, pages, delay=1.5):
    """Add delay between requests"""
    for page in range(1, pages + 1):
        data = fetch_artifacts(api_key, page)
        time.sleep(delay)  # Increase delay if still rate limited
        yield data
```

### Issue: Database Connection Timeout

**Problem:** Connection drops during large batch inserts

**Solution:**
```python
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'connect_timeout': 30,  # Increase timeout
    'autocommit': False,
    'pool_size': 5,
    'pool_reset_session': True
}
```

### Issue: Large Dataset Memory Usage

**Problem:** Out of memory when processing thousands of records

**Solution:**
```python
def chunked_insert(connection, df, chunk_size=1000):
    """Insert data in chunks to manage memory"""
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i + chunk_size]
        batch_insert_metadata(connection, chunk)
        print(f"Processed {i + len(chunk)} / {len(df)} records")
```

### Issue: Missing or Malformed API Data

**Problem:** Some records have missing or unexpected field structures

**Solution:**
```python
def safe_get(record, key, default='', max_length=None):
    """Safely extract field with default and length validation"""
    value = record.get(key, default)
    if value is None:
        value = default
    value = str(value)
    if max_length and len(value) > max_length:
        value = value[:max_length]
    return value

# Usage in transform function
metadata = {
    'id': record.get('id'),
    'title': safe_get(record, 'title', max_length=500),
    'culture': safe_get(record, 'culture', max_length=255),
    # ... etc
}
```

### Issue: Streamlit App Performance

**Problem:** Dashboard slow with large result sets

**Solution:**
```python
@st.cache_data(ttl=600)  # Cache for 10 minutes
def cached_query_execution(query_name, query_sql):
    """Cache query results to improve performance"""
    connection = create_connection(DB_CONFIG)
    df = execute_analytical_query(connection, query_name, query_sql)
    connection.close()
    return df

# Use in Streamlit
df_results = cached_query_execution(query_name, ANALYTICAL_QUERIES[query_name])
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Run ETL pipeline standalone
python etl_pipeline.py

# Run specific analytics query
python run_analytics.py --query "Artifacts by Culture"
```

This skill provides comprehensive coverage of building ETL pipelines, SQL analytics, and visualization dashboards using the Harvard Art Museums API with Python, MySQL, and Streamlit.
