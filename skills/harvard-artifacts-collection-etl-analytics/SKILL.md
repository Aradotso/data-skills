---
name: harvard-artifacts-collection-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data analytics dashboard with museum artifacts
  - extract and transform Harvard museum API data
  - set up a data engineering project with art collections
  - query and visualize Harvard artifacts data
  - implement museum data pipeline with SQL and Streamlit
  - analyze Harvard Art Museums collection data
  - create interactive visualizations for museum artifacts
---

# Harvard Artifacts Collection ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive data visualization using Streamlit.

## What It Does

The Harvard Artifacts Collection ETL Analytics project provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations

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
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add the key to your `.env` file

### Database Setup

Create the required database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    dated VARCHAR(200),
    url VARCHAR(500),
    description TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    base_image_url VARCHAR(500),
    has_image BOOLEAN,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core API Integration

### Fetching Artifact Data

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
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch first page of artifacts
data = fetch_artifacts(page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### Handling Pagination

```python
def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts with pagination"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(page=page, size=100)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
            # Check if there are more pages
            if len(artifacts) == 0:
                break
                
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifact_metadata(raw_data):
    """Transform raw API data to metadata DataFrame"""
    metadata_list = []
    
    for artifact in raw_data:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'description': artifact.get('description')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(raw_data):
    """Transform media information to DataFrame"""
    media_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        primary_image = artifact.get('primaryimageurl')
        has_image = 1 if primary_image else 0
        
        media = {
            'artifact_id': artifact_id,
            'media_type': 'image',
            'base_image_url': primary_image,
            'has_image': has_image
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(raw_data):
    """Transform color information to DataFrame"""
    colors_list = []
    
    for artifact in raw_data:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT', 3306),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """Load metadata DataFrame to database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, department, technique, dated, url, description)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), period=VALUES(period)
    """
    
    data_tuples = [tuple(x) for x in df_metadata.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(df_metadata)} metadata records")

def load_media(df_media):
    """Load media DataFrame to database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, media_type, base_image_url, has_image)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df_media.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()

def load_colors(df_colors):
    """Load colors DataFrame to database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (artifact_id, color_hex, color_name, percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df_colors.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=5):
    """Execute complete ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("1. Extracting data from API...")
    raw_artifacts = fetch_all_artifacts(max_pages=num_pages)
    print(f"Extracted {len(raw_artifacts)} artifacts")
    
    # Transform
    print("2. Transforming data...")
    df_metadata = transform_artifact_metadata(raw_artifacts)
    df_media = transform_artifact_media(raw_artifacts)
    df_colors = transform_artifact_colors(raw_artifacts)
    
    # Load
    print("3. Loading data to database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print("ETL Pipeline completed successfully!")
    return df_metadata, df_media, df_colors
```

## SQL Analytics Queries

### Common Analytics Patterns

```python
def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Artifacts by Culture
query_culture = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY count DESC
LIMIT 10
"""

# Artifacts by Century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Top Classifications
query_classification = """
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY count DESC
LIMIT 15
"""

# Media Availability
query_media = """
SELECT 
    has_image,
    COUNT(*) as count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) as percentage
FROM artifactmedia
GROUP BY has_image
"""

# Top Colors Used
query_colors = """
SELECT 
    color_name,
    COUNT(*) as occurrences,
    ROUND(AVG(percentage), 2) as avg_percentage
FROM artifactcolors
WHERE color_name IS NOT NULL
GROUP BY color_name
ORDER BY occurrences DESC
LIMIT 10
"""

# Department-wise Distribution
query_department = """
SELECT 
    department,
    COUNT(*) as artifact_count,
    SUM(CASE WHEN m.has_image = 1 THEN 1 ELSE 0 END) as with_images
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC
"""
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "Analytics Dashboard", "Data Explorer"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "Analytics Dashboard":
        show_analytics_page()
    else:
        show_explorer_page()

def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    with col2:
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                df_meta, df_media, df_colors = run_etl_pipeline(num_pages)
                st.success(f"✅ Loaded {len(df_meta)} artifacts successfully!")
                
                # Show summary
                st.metric("Total Artifacts", len(df_meta))
                st.metric("Total Media Records", len(df_media))
                st.metric("Total Color Records", len(df_colors))

def show_analytics_page():
    st.header("📊 Analytics Dashboard")
    
    # Query selector
    queries = {
        "Artifacts by Culture": query_culture,
        "Artifacts by Century": query_century,
        "Top Classifications": query_classification,
        "Media Availability": query_media,
        "Top Colors": query_colors,
        "Department Distribution": query_department
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df_result = execute_query(queries[selected_query])
            
            # Display results
            st.dataframe(df_result, use_container_width=True)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result, 
                    x=df_result.columns[0], 
                    y=df_result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Advanced Visualization

```python
def create_color_distribution_chart(df_colors):
    """Create interactive color distribution visualization"""
    fig = px.treemap(
        df_colors,
        path=['color_name'],
        values='occurrences',
        title='Color Distribution in Artifacts',
        color='avg_percentage',
        color_continuous_scale='Viridis'
    )
    return fig

def create_timeline_chart(df_century):
    """Create timeline chart for artifacts by century"""
    fig = px.line(
        df_century,
        x='century',
        y='count',
        title='Artifact Collection Timeline',
        markers=True
    )
    fig.update_layout(
        xaxis_title="Century",
        yaxis_title="Number of Artifacts"
    )
    return fig

def create_department_comparison(df_dept):
    """Create department comparison with images"""
    fig = px.bar(
        df_dept,
        x='department',
        y=['artifact_count', 'with_images'],
        title='Artifacts by Department (Total vs With Images)',
        barmode='group'
    )
    return fig
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def fetch_with_rate_limit(max_pages, delay=1):
    """Fetch artifacts with rate limiting"""
    artifacts = []
    
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(page=page)
        artifacts.extend(data['records'])
        
        # Rate limit: wait between requests
        time.sleep(delay)
        
        if page % 10 == 0:
            print(f"Progress: {page}/{max_pages} pages")
    
    return artifacts
```

### Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the latest artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    conn.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Load only new artifacts"""
    latest_id = get_latest_artifact_id()
    print(f"Latest artifact ID: {latest_id}")
    
    # Fetch new artifacts with filter
    # Implementation depends on API support for ID filtering
    pass
```

### Data Quality Checks

```python
def validate_data_quality(df):
    """Check data quality before loading"""
    issues = []
    
    # Check for nulls in critical fields
    if df['id'].isnull().any():
        issues.append("ID field contains null values")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append(f"Found {df['id'].duplicated().sum()} duplicate IDs")
    
    # Check data types
    if not pd.api.types.is_integer_dtype(df['id']):
        issues.append("ID field is not integer type")
    
    return issues
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time
from functools import wraps

def retry_with_backoff(max_retries=3, backoff_factor=2):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    wait_time = backoff_factor ** attempt
                    print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                    time.sleep(wait_time)
        return wrapper
    return decorator

@retry_with_backoff(max_retries=3)
def fetch_artifacts_with_retry(page, size):
    return fetch_artifacts(page, size)
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connection"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=''):
    """Safely get value from dictionary"""
    value = dictionary.get(key, default)
    return value if value is not None else default

def transform_with_defaults(artifact):
    """Transform artifact with default values for missing fields"""
    return {
        'id': artifact.get('id', 0),
        'title': safe_get(artifact, 'title', 'Untitled'),
        'culture': safe_get(artifact, 'culture', 'Unknown'),
        'century': safe_get(artifact, 'century', 'Unknown'),
        # ... other fields
    }
```

### Memory Management for Large Datasets

```python
def chunked_load(df, chunk_size=1000):
    """Load data in chunks to manage memory"""
    total_rows = len(df)
    
    for i in range(0, total_rows, chunk_size):
        chunk = df.iloc[i:i + chunk_size]
        load_metadata(chunk)
        print(f"Loaded chunk {i//chunk_size + 1}: {len(chunk)} rows")
```

This skill provides everything needed to build, deploy, and extend the Harvard Artifacts Collection ETL Analytics application for real-world data engineering scenarios.
