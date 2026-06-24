---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards for museum artifact data using Harvard Art Museums API, MySQL, and Streamlit
triggers:
  - build a museum artifact data pipeline
  - create ETL pipeline for Harvard Art Museums API
  - set up artifact analytics dashboard with Streamlit
  - extract and analyze museum collection data
  - build data engineering pipeline for art museum artifacts
  - create SQL analytics for Harvard artifacts
  - develop art museum data visualization app
  - implement museum collection ETL workflow
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Run 20+ predefined analytical queries on structured museum data
- **Interactive Dashboard**: Streamlit-based interface for data collection, querying, and visualization
- **Data Visualization**: Plotly charts for exploring artifacts by culture, century, department, and color patterns

**Architecture Flow**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization

## Installation

### Prerequisites

```bash
# Python 3.8 or higher required
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Database Setup

**Option 1: MySQL Local**
```bash
# Install MySQL
# macOS
brew install mysql
brew services start mysql

# Create database
mysql -u root -p
CREATE DATABASE harvard_artifacts;
```

**Option 2: TiDB Cloud**
- Sign up at https://tidbcloud.com
- Create a free cluster
- Note connection details (host, port, user, password)

### Environment Configuration

Create a `.env` file in your project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key at: https://www.harvardartmuseums.org/collections/api

## Core Components

### 1. API Data Extraction

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
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages
def fetch_multiple_pages(num_pages=5):
    """Fetch artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        print(f"Fetched page {page}/{num_pages} - Total: {info['totalrecords']}")
    
    return all_artifacts
```

### 2. Database Schema Creation

```python
import mysql.connector
import os

def create_database_schema():
    """Create tables for artifact data"""
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = conn.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            accession_number VARCHAR(255),
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            technique TEXT,
            period VARCHAR(255),
            dated VARCHAR(255),
            url TEXT,
            description TEXT,
            provenance TEXT,
            INDEX idx_culture (culture),
            INDEX idx_century (century),
            INDEX idx_department (department)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(50),
            base_url TEXT,
            primary_image BOOLEAN,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
            INDEX idx_artifact_id (artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_hex VARCHAR(10),
            color_percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
            INDEX idx_artifact_id (artifact_id)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print("Database schema created successfully")
```

### 3. ETL Transform and Load

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform API response into structured dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'accession_number': artifact.get('accessionyear'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'base_url': image.get('baseimageurl'),
                    'primary_image': image.get('primary', False)
                }
                media_records.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_percentage': color.get('percent')
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL"""
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, accession_number, title, culture, century, classification, 
             department, technique, period, dated, url, description, provenance)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, media_type, base_url, primary_image)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percentage)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

### 4. SQL Analytics Queries

```python
def get_artifacts_by_culture():
    """Count artifacts grouped by culture"""
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """
    return execute_query(query)

def get_artifacts_by_century():
    """Distribution of artifacts across centuries"""
    query = """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """
    return execute_query(query)

def get_media_availability():
    """Check image availability for artifacts"""
    query = """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Images' 
                 ELSE 'No Images' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """
    return execute_query(query)

def get_top_colors():
    """Most common colors in artifact collection"""
    query = """
        SELECT color_hex, 
               COUNT(*) as frequency,
               AVG(color_percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY frequency DESC
        LIMIT 20
    """
    return execute_query(query)

def get_department_distribution():
    """Artifacts by department"""
    query = """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
    return execute_query(query)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("Data Engineering & Analytics Application")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    """ETL data collection interface"""
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching artifacts from API..."):
            artifacts = fetch_multiple_pages(num_pages)
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            
        st.success(f"✅ Loaded {len(metadata_df)} artifacts successfully!")
        st.dataframe(metadata_df.head(10))

def show_sql_analytics():
    """SQL query execution interface"""
    st.header("🔍 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": get_artifacts_by_culture,
        "Artifacts by Century": get_artifacts_by_century,
        "Media Availability": get_media_availability,
        "Top Colors": get_top_colors,
        "Department Distribution": get_department_distribution
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = queries[selected_query]()
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1])
            st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """Pre-built visualizations"""
    st.header("📊 Data Visualizations")
    
    col1, col2 = st.columns(2)
    
    with col1:
        st.subheader("Artifacts by Culture")
        df = get_artifacts_by_culture()
        fig = px.bar(df, x='culture', y='artifact_count', 
                     color='artifact_count', color_continuous_scale='Viridis')
        st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        st.subheader("Department Distribution")
        df = get_department_distribution()
        fig = px.pie(df, names='department', values='count')
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Batch Processing Large Collections

```python
def batch_load_artifacts(total_pages=100, batch_size=10):
    """Load artifacts in batches to avoid memory issues"""
    
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        artifacts = []
        for page in range(batch_start, batch_end):
            records, _ = fetch_artifacts(page=page)
            artifacts.extend(records)
        
        # Transform and load batch
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        
        print(f"Completed batch {batch_start}-{batch_end-1}")
```

### Error Handling and Retry Logic

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff retry"""
    
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1} after {wait_time}s - Error: {e}")
            time.sleep(wait_time)
```

### Custom Analytics Query Builder

```python
def build_custom_query(filters):
    """Build dynamic SQL query based on filters"""
    
    query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        query += f" AND culture = '{filters['culture']}'"
    
    if filters.get('century'):
        query += f" AND century = '{filters['century']}'"
    
    if filters.get('department'):
        query += f" AND department = '{filters['department']}'"
    
    return execute_query(query)
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_with_rate_limit(page, delay=1):
    """Add delay between requests"""
    time.sleep(delay)
    return fetch_artifacts(page=page)
```

### Database Connection Issues

```python
def test_database_connection():
    """Verify database connectivity"""
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        print("✅ Database connection successful")
        conn.close()
        return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Missing Data Handling

```python
def safe_extract(artifact, key, default=None):
    """Safely extract values from nested dictionaries"""
    try:
        return artifact.get(key, default)
    except (KeyError, AttributeError):
        return default
```

### Large Dataset Performance

For better performance with large datasets:

```python
# Use batch inserts
def batch_insert(df, table_name, batch_size=1000):
    """Insert data in batches"""
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        # Insert batch logic here
        print(f"Inserted batch {i//batch_size + 1}")
```

## Key Configuration

All configuration should use environment variables:

```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    HARVARD_API_KEY = os.getenv('HARVARD_API_KEY')
    DB_HOST = os.getenv('DB_HOST', 'localhost')
    DB_PORT = int(os.getenv('DB_PORT', 3306))
    DB_USER = os.getenv('DB_USER')
    DB_PASSWORD = os.getenv('DB_PASSWORD')
    DB_NAME = os.getenv('DB_NAME', 'harvard_artifacts')
    
    # API settings
    API_BASE_URL = "https://api.harvardartmuseums.org/object"
    DEFAULT_PAGE_SIZE = 100
    MAX_RETRIES = 3
```

This skill provides a complete foundation for building data engineering pipelines with museum APIs, SQL analytics, and interactive dashboards.
