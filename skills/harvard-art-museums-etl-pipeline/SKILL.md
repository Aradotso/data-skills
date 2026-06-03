---
name: harvard-art-museums-etl-pipeline
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API, ETL processes, SQL analytics, and Streamlit dashboards
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up Harvard Art Museums data collection and analytics
  - build a Streamlit dashboard for art collection data
  - extract and transform Harvard museum API data
  - create SQL analytics for art artifacts
  - implement museum data pipeline with Python
  - visualize Harvard Art Museums data with Plotly
---

# Harvard Art Museums ETL Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables you to build complete data engineering pipelines using the Harvard Art Museums API. The project demonstrates real-world ETL (Extract, Transform, Load) operations, SQL database design, analytical queries, and interactive visualization using Streamlit and Plotly.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract nested JSON, transform to relational format, load into SQL databases
- **Database Design**: Structured tables for artifact metadata, media, and color information
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

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

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file or configure environment variables:

```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Configuration

The project supports MySQL and TiDB Cloud. Ensure you have:

1. A running MySQL/TiDB instance
2. Database credentials with CREATE, INSERT, SELECT permissions
3. Network access configured (for cloud databases)

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
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

# Fetch first page
data = fetch_artifacts(page=1, size=100)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### 2. ETL Pipeline

**Extract and Transform**:

```python
import pandas as pd

def extract_artifact_metadata(artifacts):
    """Extract metadata from artifact records"""
    metadata = []
    
    for artifact in artifacts:
        record = {
            'object_id': artifact.get('objectid'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'period': artifact.get('period', 'Unknown')
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def extract_artifact_media(artifacts):
    """Extract media/image information"""
    media_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'object_id': object_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            }
            media_records.append(media)
    
    return pd.DataFrame(media_records)

def extract_artifact_colors(artifacts):
    """Extract color information"""
    color_records = []
    
    for artifact in artifacts:
        object_id = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'object_id': object_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return pd.DataFrame(color_records)
```

**Load to SQL**:

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
        print(f"Database connection error: {e}")
        return None

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            medium VARCHAR(500),
            department VARCHAR(255),
            technique VARCHAR(500),
            dated VARCHAR(255),
            period VARCHAR(255)
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url VARCHAR(500),
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_data(connection, df, table_name):
    """Batch insert data into SQL table"""
    cursor = connection.cursor()
    
    # Prepare insert query
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(query, data_tuples)
    
    connection.commit()
    cursor.close()
    print(f"Inserted {cursor.rowcount} rows into {table_name}")
```

### 3. Complete ETL Execution

```python
def run_etl_pipeline(num_pages=5, page_size=100):
    """Execute full ETL pipeline"""
    connection = create_connection()
    
    if not connection:
        return
    
    # Create tables
    create_tables(connection)
    
    # Fetch and process data
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = fetch_artifacts(page=page, size=page_size)
        artifacts = data['records']
        
        # Transform
        metadata_df = extract_artifact_metadata(artifacts)
        media_df = extract_artifact_media(artifacts)
        colors_df = extract_artifact_colors(artifacts)
        
        # Load
        load_data(connection, metadata_df, 'artifactmetadata')
        load_data(connection, media_df, 'artifactmedia')
        load_data(connection, colors_df, 'artifactcolors')
    
    connection.close()
    print("ETL pipeline completed successfully!")

# Execute pipeline
run_etl_pipeline(num_pages=10, page_size=100)
```

### 4. SQL Analytics Queries

```python
def run_analytical_query(connection, query_name):
    """Execute predefined analytical queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century != 'Unknown'
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'media_availability': """
            SELECT 
                CASE WHEN COUNT(am.image_id) > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia am ON a.object_id = am.object_id
            GROUP BY media_status
        """,
        
        'top_colors': """
            SELECT color, COUNT(*) as frequency
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department != 'Unknown'
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Configuration")
api_key = st.sidebar.text_input("Harvard API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))

# ETL Section
st.header("1️⃣ Data Collection")
col1, col2 = st.columns(2)

with col1:
    num_pages = st.number_input("Number of Pages", min_value=1, max_value=100, value=5)
    
with col2:
    page_size = st.number_input("Records per Page", min_value=10, max_value=100, value=100)

if st.button("Run ETL Pipeline"):
    with st.spinner("Running ETL pipeline..."):
        run_etl_pipeline(num_pages=num_pages, page_size=page_size)
        st.success("ETL pipeline completed!")

# Analytics Section
st.header("2️⃣ SQL Analytics")

query_options = {
    "Artifacts by Culture": "artifacts_by_culture",
    "Artifacts by Century": "artifacts_by_century",
    "Media Availability": "media_availability",
    "Top Colors": "top_colors",
    "Department Distribution": "department_distribution"
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))

if st.button("Execute Query"):
    connection = create_connection()
    if connection:
        df_results = run_analytical_query(connection, query_options[selected_query])
        
        # Display table
        st.dataframe(df_results, use_container_width=True)
        
        # Visualization
        if len(df_results) > 0:
            x_col = df_results.columns[0]
            y_col = df_results.columns[1]
            
            fig = px.bar(df_results, x=x_col, y=y_col, title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
        
        connection.close()
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(max_records=1000):
    """Fetch artifacts with automatic pagination"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(page=page, size=size)
        artifacts = data['records']
        
        if not artifacts:
            break
        
        all_artifacts.extend(artifacts)
        page += 1
    
    return all_artifacts[:max_records]
```

### Error Handling and Retry Logic

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s")
            time.sleep(wait_time)
```

### Data Quality Validation

```python
def validate_artifact_data(df):
    """Validate extracted data quality"""
    issues = []
    
    # Check for nulls in critical fields
    if df['object_id'].isnull().any():
        issues.append("Missing object IDs")
    
    # Check for duplicates
    if df['object_id'].duplicated().any():
        issues.append("Duplicate object IDs found")
    
    # Check data types
    if df['object_id'].dtype != 'int64':
        issues.append("Invalid object_id type")
    
    return issues
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_with_rate_limit(page, delay=1):
    """Add delay between API calls"""
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Database Connection Issues

Check connection with diagnostic function:

```python
def test_database_connection():
    """Test database connectivity"""
    try:
        conn = create_connection()
        if conn and conn.is_connected():
            print("✅ Database connection successful")
            conn.close()
            return True
        else:
            print("❌ Failed to connect to database")
            return False
    except Error as e:
        print(f"❌ Connection error: {e}")
        return False
```

### Missing Environment Variables

```python
def check_environment():
    """Verify required environment variables"""
    required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
    missing = [var for var in required_vars if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing environment variables: {', '.join(missing)}")
```

### Data Type Mismatches

Handle nullable fields properly:

```python
def safe_insert(value, default='Unknown'):
    """Handle None values for SQL insert"""
    return value if value is not None else default

# Apply when creating DataFrames
metadata['culture'] = metadata['culture'].apply(lambda x: safe_insert(x))
```

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(filters):
    """Build dynamic SQL queries based on user filters"""
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        base_query += f" AND culture = '{filters['culture']}'"
    
    if filters.get('century'):
        base_query += f" AND century = '{filters['century']}'"
    
    if filters.get('department'):
        base_query += f" AND department = '{filters['department']}'"
    
    return base_query
```

This skill provides everything needed to build production-ready data engineering pipelines with the Harvard Art Museums API, from initial data collection to interactive analytics dashboards.
