---
name: harvard-artifacts-etl-analytics
description: ETL pipeline and analytics application for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - integrate Harvard Art Museums API with SQL database
  - create artifact data analytics dashboard
  - extract and transform museum collection data
  - set up Streamlit app for art museum analytics
  - query Harvard artifacts collection with SQL
  - visualize museum artifact metadata
  - process nested JSON from Harvard API
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill helps you build end-to-end ETL pipelines and analytics applications using the Harvard Art Museums API. The project demonstrates real-world data engineering patterns including API integration, data transformation, SQL database design, and interactive visualization with Streamlit.

## What This Project Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** data using predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

## Architecture Flow

```
Harvard API → ETL Pipeline → SQL Database → Analytics Queries → Streamlit Dashboard
```

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### Environment Variables

Set up your credentials using environment variables:

```bash
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Database Setup

Create the necessary tables:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    objectnumber VARCHAR(255),
    description TEXT,
    dimensions VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Core ETL Pipeline Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size
    }
    
    response = requests.get(base_url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data.get("records", [])
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform raw API data into structured metadata"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            "id": artifact.get("id"),
            "title": artifact.get("title"),
            "culture": artifact.get("culture"),
            "century": artifact.get("century"),
            "dated": artifact.get("dated"),
            "department": artifact.get("department"),
            "classification": artifact.get("classification"),
            "objectnumber": artifact.get("objectnumber"),
            "description": artifact.get("description"),
            "dimensions": artifact.get("dimensions")
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_media_data(artifacts):
    """Extract nested media information"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        images = artifact.get("images", [])
        
        for img in images:
            media_list.append({
                "artifact_id": artifact_id,
                "image_url": img.get("baseimageurl"),
                "media_type": img.get("format", "image")
            })
    
    return pd.DataFrame(media_list)

def extract_color_data(artifacts):
    """Extract color information from artifacts"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        colors = artifact.get("colors", [])
        
        for color in colors:
            color_list.append({
                "artifact_id": artifact_id,
                "color_hex": color.get("hex"),
                "color_percent": color.get("percent")
            })
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv("DB_HOST"),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
            database=os.getenv("DB_NAME")
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata_to_db(df_metadata):
    """Batch insert metadata into database"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, dated, department, classification, 
     objectnumber, description, dimensions)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(row) for row in df_metadata.values]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        print(f"Inserted {cursor.rowcount} records")
        return True
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## Streamlit Dashboard Components

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    st.title("🏛️ Harvard Art Museums Collection Analytics")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Home", "Data Collection", "Analytics Dashboard", "Visualizations"]
    )
    
    if menu == "Data Collection":
        show_data_collection()
    elif menu == "Analytics Dashboard":
        show_analytics()
    elif menu == "Visualizations":
        show_visualizations()
    else:
        show_home()

if __name__ == "__main__":
    main()
```

### Data Collection Interface

```python
def show_data_collection():
    st.header("📥 Collect Artifact Data")
    
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv("HARVARD_API_KEY", ""))
    num_pages = st.number_input("Number of Pages to Fetch", min_value=1, 
                                max_value=100, value=5)
    page_size = st.selectbox("Items per Page", [10, 50, 100], index=1)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching and processing data..."):
            progress_bar = st.progress(0)
            
            all_artifacts = []
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(api_key, page=page, size=page_size)
                all_artifacts.extend(data.get("records", []))
                progress_bar.progress(page / num_pages)
            
            # Transform
            df_metadata = transform_artifact_metadata(all_artifacts)
            df_media = extract_media_data(all_artifacts)
            df_colors = extract_color_data(all_artifacts)
            
            # Load
            load_metadata_to_db(df_metadata)
            st.success(f"✅ Loaded {len(df_metadata)} artifacts into database")
```

### Analytics Queries

```python
def show_analytics():
    st.header("📊 SQL Analytics Dashboard")
    
    query_options = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY century
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        "Artifacts with Images": """
            SELECT 
                CASE WHEN media_id IS NOT NULL THEN 'Has Images' 
                     ELSE 'No Images' END as media_status,
                COUNT(DISTINCT a.id) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY media_status
        """,
        "Top 10 Most Common Colors": """
            SELECT color_hex, COUNT(*) as frequency, 
                   AVG(color_percent) as avg_percent
            FROM artifactcolors
            GROUP BY color_hex
            ORDER BY frequency DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Analysis Query", list(query_options.keys()))
    
    if st.button("Run Query"):
        df_result = execute_query(query_options[selected_query])
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, x=df_result.columns[0], 
                        y=df_result.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = get_db_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Common Patterns

### Pagination Handler

```python
def fetch_all_pages(api_key, max_pages=10):
    """Fetch multiple pages with rate limiting"""
    import time
    
    all_records = []
    for page in range(1, max_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_records.extend(data.get("records", []))
        
        # Rate limiting
        time.sleep(0.5)
    
    return all_records
```

### Error Handling for ETL

```python
def safe_etl_pipeline(api_key, pages):
    """ETL with comprehensive error handling"""
    try:
        # Extract
        artifacts = fetch_all_pages(api_key, pages)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        df_metadata = transform_artifact_metadata(artifacts)
        
        # Validate
        if df_metadata.empty:
            raise ValueError("Transformation resulted in empty dataset")
        
        # Load
        success = load_metadata_to_db(df_metadata)
        return success, len(df_metadata)
        
    except requests.exceptions.RequestException as e:
        st.error(f"API Error: {e}")
        return False, 0
    except Exception as e:
        st.error(f"ETL Error: {e}")
        return False, 0
```

## Troubleshooting

### API Rate Limiting
If you encounter `429 Too Many Requests`:
```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retries():
    session = requests.Session()
    retry = Retry(total=3, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues
```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        if conn and conn.is_connected():
            print("✅ Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Handling Missing Data
```python
def clean_artifact_data(df):
    """Handle missing values in artifact data"""
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Undated')
    df['description'] = df['description'].fillna('')
    return df
```

This skill enables AI agents to help developers build complete data engineering pipelines from API to visualization using the Harvard Art Museums collection as a real-world dataset.
