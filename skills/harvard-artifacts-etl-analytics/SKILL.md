---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifacts
  - fetch data from Harvard Art Museums API
  - create analytics dashboard with Streamlit
  - set up SQL database for artifact collections
  - visualize museum collection data
  - query Harvard artifacts collection
  - build data engineering pipeline for art museums
  - analyze artifact metadata with SQL
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It implements ETL pipelines to extract artifact data, transform nested JSON into relational tables, load into SQL databases, and visualize analytics through an interactive Streamlit dashboard.

## What It Does

- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational database tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics Queries**: Provides 20+ predefined SQL queries for artifact analysis
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time insights

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

### Environment Variables

Set up your configuration using environment variables or a `.env` file:

```bash
# Harvard Art Museums API
export HARVARD_API_KEY="your_api_key_here"

# Database Configuration
export DB_HOST="your_database_host"
export DB_PORT="3306"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
```

### Database Setup

Create the required database and tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    division VARCHAR(200),
    accession_number VARCHAR(100)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Key Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
artifacts = data.get('records', [])
```

### 2. ETL Pipeline - Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into structured metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'description': artifact.get('description', ''),
            'division': artifact.get('division', 'Unknown'),
            'accession_number': artifact.get('accessionNumber', 'Unknown')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl', ''),
                'media_type': 'image'
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color data"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
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
        print(f"Error connecting to database: {e}")
        return None

def load_metadata(df_metadata):
    """Load artifact metadata to database"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, dated, description, division, accession_number)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    try:
        data_tuples = [tuple(x) for x in df_metadata.to_numpy()]
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Loaded {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error loading metadata: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_media(df_media):
    """Load artifact media to database"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, media_type)
    VALUES (%s, %s, %s)
    """
    
    try:
        data_tuples = [tuple(x) for x in df_media.to_numpy()]
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Loaded {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error loading media: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. Analytics Queries

```python
def execute_analytics_query(query):
    """Execute analytical SQL query and return DataFrame"""
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Error executing query: {e}")
        return None
    finally:
        connection.close()

# Sample Analytics Queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Top Colors Used": """
        SELECT color_hex, SUM(color_percent) as total_usage
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY total_usage DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT m.id, m.title, COUNT(med.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia med ON m.id = med.artifact_id
        GROUP BY m.id, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Classification by Department": """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE department != 'Unknown' AND classification != 'Unknown'
        GROUP BY department, classification
        ORDER BY department, count DESC
    """
}
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for API configuration
    with st.sidebar:
        st.header("⚙️ Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        st.header("📥 Data Collection")
        num_pages = st.slider("Number of pages to fetch", 1, 10, 1)
        page_size = st.selectbox("Records per page", [10, 50, 100], index=2)
        
        if st.button("🔄 Fetch & Load Data"):
            with st.spinner("Fetching data..."):
                all_artifacts = []
                for page in range(1, num_pages + 1):
                    data = fetch_artifacts(api_key, page=page, size=page_size)
                    all_artifacts.extend(data.get('records', []))
                
                # ETL Process
                df_metadata = transform_artifact_metadata(all_artifacts)
                df_media = transform_artifact_media(all_artifacts)
                df_colors = transform_artifact_colors(all_artifacts)
                
                # Load to database
                load_metadata(df_metadata)
                load_media(df_media)
                load_colors(df_colors)
                
                st.success(f"✅ Loaded {len(all_artifacts)} artifacts!")
    
    # Main dashboard area
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("🔍 Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        df_result = execute_analytics_query(query)
        
        if df_result is not None and not df_result.empty:
            st.subheader("📋 Query Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(df_result, 
                            x=df_result.columns[0], 
                            y=df_result.columns[1],
                            title=query_name)
                st.plotly_chart(fig, use_container_width=True)
        else:
            st.warning("No data returned from query")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_full_etl_pipeline(api_key, num_pages=5, page_size=100):
    """Execute complete ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # Extract
    all_artifacts = []
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page, size=page_size)
        all_artifacts.extend(data.get('records', []))
    
    print(f"Extracted {len(all_artifacts)} artifacts")
    
    # Transform
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    
    print(f"Transformed: {len(df_metadata)} metadata, {len(df_media)} media, {len(df_colors)} colors")
    
    # Load
    success = all([
        load_metadata(df_metadata),
        load_media(df_media),
        load_colors(df_colors)
    ])
    
    if success:
        print("✅ ETL Pipeline completed successfully!")
    else:
        print("❌ ETL Pipeline encountered errors")
    
    return success
```

### Batch Processing with Error Handling

```python
def fetch_artifacts_with_retry(api_key, page, size=100, max_retries=3):
    """Fetch with retry logic"""
    import time
    
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page, size)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=1):
    """Fetch with rate limiting between requests"""
    all_data = []
    for page in pages:
        data = fetch_artifacts(api_key, page=page)
        all_data.extend(data.get('records', []))
        time.sleep(delay)  # Wait between requests
    return all_data
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            print("✅ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE()")
            db_name = cursor.fetchone()[0]
            print(f"Connected to database: {db_name}")
            cursor.close()
            connection.close()
            return True
        else:
            print("❌ Failed to connect to database")
            return False
    except Error as e:
        print(f"❌ Database error: {e}")
        return False
```

### Empty or Null Data Handling

```python
def clean_artifact_data(artifacts):
    """Clean and validate artifact data"""
    cleaned = []
    for artifact in artifacts:
        # Skip artifacts without essential fields
        if not artifact.get('id'):
            continue
        
        # Ensure required fields have defaults
        artifact.setdefault('title', 'Untitled')
        artifact.setdefault('culture', 'Unknown')
        artifact.setdefault('century', 'Unknown')
        
        cleaned.append(artifact)
    
    return cleaned
```

## Project Architecture

The application follows this data flow:

1. **Extract**: Harvard API → JSON responses (paginated)
2. **Transform**: JSON → Pandas DataFrames (normalized)
3. **Load**: DataFrames → MySQL tables (batch inserts)
4. **Analyze**: SQL queries → Pandas DataFrames
5. **Visualize**: DataFrames → Plotly charts → Streamlit UI

This architecture demonstrates production-ready data engineering practices including error handling, batch processing, relational database design, and interactive analytics.
