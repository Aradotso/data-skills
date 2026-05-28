---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - create a data engineering app with Harvard Art Museums API
  - set up analytics dashboard for artifact collections
  - implement SQL analytics for museum artifacts
  - visualize Harvard Art Museums data with Streamlit
  - extract and transform museum API data into SQL
  - create interactive art collection analytics
  - build a data pipeline for cultural heritage data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in building end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App enables:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Executes 20+ predefined analytical queries on artifact collections
- **Interactive Visualization**: Displays results through Streamlit dashboards with Plotly charts

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud database
- Harvard Art Museums API key (get from: https://harvardartmuseums.org/collections/api)

### Setup

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

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Database Configuration

Create a database configuration module:

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

def get_db_connection():
    """Establish database connection"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return connection

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            technique VARCHAR(500),
            division VARCHAR(200)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(100),
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
```

### API Configuration

```python
import requests
import os

class HarvardAPIClient:
    """Client for Harvard Art Museums API"""
    
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org"
        
    def fetch_objects(self, page=1, size=100):
        """Fetch objects with pagination"""
        url = f"{self.base_url}/object"
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_object_by_id(self, object_id):
        """Fetch single object details"""
        url = f"{self.base_url}/object/{object_id}"
        params = {'apikey': self.api_key}
        
        response = requests.get(url, params=params)
        response.raise_for_status()
        return response.json()
```

## ETL Pipeline Implementation

### Extract Phase

```python
import time

def extract_artifacts(api_client, num_pages=10):
    """Extract artifact data from API with rate limiting"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        try:
            data = api_client.fetch_objects(page=page, size=100)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Extracted page {page}: {len(artifacts)} artifacts")
            
            # Rate limiting
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error extracting page {page}: {e}")
            continue
    
    return all_artifacts
```

### Transform Phase

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into structured dataframes"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
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
            'technique': artifact.get('technique', '')[:500],
            'division': artifact.get('division', '')[:200]
        }
        metadata_records.append(metadata)
        
        # Extract media information
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'primary',
                'baseimageurl': primary_image[:500],
                'iiifbaseuri': artifact.get('iiifbaseuri', '')[:500]
            }
            media_records.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load Phase

```python
def load_to_database(metadata_df, media_df, colors_df, connection):
    """Load transformed data into SQL database"""
    cursor = connection.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        sql = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, dated, technique, division)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        cursor.execute(sql, tuple(row))
    
    # Load media
    for _, row in media_df.iterrows():
        sql = """
            INSERT INTO artifactmedia 
            (artifact_id, media_type, baseimageurl, iiifbaseuri)
            VALUES (%s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    # Load colors
    for _, row in colors_df.iterrows():
        sql = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

## SQL Analytics Queries

### Predefined Analytical Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'With Image' 
                 ELSE 'Without Image' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY image_status
    """,
    
    "Top Colors in Collection": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 20
    """
}

def execute_query(connection, query_name):
    """Execute analytical query and return results"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)
```

## Streamlit Application

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
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        etl_page()
    elif page == "SQL Analytics":
        analytics_page()
    elif page == "Visualizations":
        visualization_page()

def etl_page():
    """ETL pipeline interface"""
    st.header("📥 ETL Pipeline")
    
    num_pages = st.number_input("Number of pages to extract", 1, 50, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            api_client = HarvardAPIClient()
            artifacts = extract_artifacts(api_client, num_pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
            
            # Show preview
            st.subheader("Metadata Preview")
            st.dataframe(metadata_df.head())
        
        with st.spinner("Loading to database..."):
            conn = get_db_connection()
            load_to_database(metadata_df, media_df, colors_df, conn)
            conn.close()
            st.success("Data loaded to database")

def analytics_page():
    """SQL analytics interface"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Execute Query"):
        conn = get_db_connection()
        df = execute_query(conn, query_name)
        conn.close()
        
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

def visualization_page():
    """Custom visualizations"""
    st.header("📈 Data Visualizations")
    
    conn = get_db_connection()
    
    # Example: Culture distribution
    df = execute_query(conn, "Artifacts by Culture")
    fig = px.bar(df, x='culture', y='count',
                 title='Top 20 Cultures in Collection',
                 labels={'count': 'Number of Artifacts'})
    st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing with Progress Tracking

```python
def batch_etl_with_progress(total_pages):
    """Run ETL with progress tracking"""
    progress_bar = st.progress(0)
    status_text = st.empty()
    
    for i in range(total_pages):
        status_text.text(f"Processing page {i+1}/{total_pages}")
        # Process page
        progress_bar.progress((i + 1) / total_pages)
    
    status_text.text("ETL Complete!")
```

### Error Handling in ETL

```python
def safe_extract(api_client, page):
    """Extract with error handling and retry logic"""
    max_retries = 3
    
    for attempt in range(max_retries):
        try:
            return api_client.fetch_objects(page=page)
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                st.error(f"Failed to fetch page {page}: {e}")
                return None
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
# Increase delay between requests
time.sleep(1.0)  # Instead of 0.5

# Reduce batch size
data = api_client.fetch_objects(page=page, size=50)  # Instead of 100
```

### Database Connection Issues

```python
# Test connection
try:
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    cursor.close()
    conn.close()
    print("Database connection successful")
except Exception as e:
    print(f"Database connection failed: {e}")
```

### Memory Issues with Large Datasets

```python
# Process in smaller chunks
def chunked_load(df, chunk_size=1000):
    """Load dataframe in chunks"""
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        load_chunk_to_db(chunk)
```

### Missing or Null Data

```python
# Handle missing values during transform
metadata = {
    'id': artifact.get('id', 0),
    'title': (artifact.get('title') or 'Untitled')[:500],
    'culture': (artifact.get('culture') or 'Unknown')[:200]
}
```

## Running the Application

```bash
# Standard run
streamlit run app.py

# Custom port
streamlit run app.py --server.port 8080

# Debug mode
streamlit run app.py --logger.level=debug
```

This skill enables AI coding agents to help developers build comprehensive data engineering pipelines for cultural heritage data, combining API integration, ETL processes, SQL analytics, and interactive visualization.
