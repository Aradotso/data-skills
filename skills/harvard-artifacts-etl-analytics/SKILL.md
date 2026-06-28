---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts data
  - create analytics dashboard with Streamlit and SQL
  - process Harvard museum API data into SQL database
  - analyze art collection data with Python and MySQL
  - set up data engineering pipeline for museum artifacts
  - visualize Harvard Art Museums data with Plotly
  - transform nested JSON art data into relational tables
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL analytics, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **SQL Analytics**: Execute predefined analytical queries on artifact metadata, media, and color data
- **Interactive Dashboards**: Visualize query results using Streamlit and Plotly

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

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    mediacount INT,
    imagecount INT,
    primaryimageurl TEXT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key API Integration Patterns

### Fetching Data from Harvard Art Museums API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
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

# Usage
data = fetch_artifacts(page=1, size=50)
artifacts = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### Handling Pagination

```python
def fetch_all_artifacts(max_records=1000, batch_size=100):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(page=page, size=batch_size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Respect API rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:max_records]
```

## ETL Pipeline Implementation

### Extract Phase

```python
import pandas as pd

def extract_artifact_metadata(artifacts):
    """Extract metadata from API response"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)
```

### Transform Phase

```python
def transform_media_data(artifacts):
    """Transform nested media data into flat structure"""
    media_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        media = {
            'objectid': objectid,
            'mediacount': len(images),
            'imagecount': len(images),
            'primaryimageurl': artifact.get('primaryimageurl', '')
        }
        media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_color_data(artifacts):
    """Extract color information from artifacts"""
    color_list = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_list.append(color_record)
    
    return pd.DataFrame(color_list)
```

### Load Phase

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
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

def load_metadata(df_metadata):
    """Load metadata into SQL database"""
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, period, century, department, classification, dated, accessionyear)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    # Batch insert
    data_tuples = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data_tuples)
    
    connection.commit()
    cursor.close()
    connection.close()
    
    return True
```

## SQL Analytics Patterns

### Sample Analytical Queries

```python
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
    """,
    
    "media_availability": """
        SELECT 
            CASE 
                WHEN imagecount > 0 THEN 'Has Images'
                ELSE 'No Images'
            END as image_status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY image_status
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(query_name):
    """Execute predefined analytical query"""
    connection = create_db_connection()
    if not connection:
        return None
    
    query = ANALYTICS_QUERIES.get(query_name)
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "ETL Pipeline", "Analytics Dashboard"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "ETL Pipeline":
        show_etl_pipeline()
    elif page == "Analytics Dashboard":
        show_analytics_dashboard()

def show_data_collection():
    st.header("Data Collection from API")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching data from Harvard API..."):
            artifacts = fetch_all_artifacts(max_records=num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
            st.session_state['artifacts'] = artifacts
            
            # Display sample
            st.dataframe(pd.DataFrame(artifacts[:10]))

def show_etl_pipeline():
    st.header("ETL Pipeline Execution")
    
    if 'artifacts' not in st.session_state:
        st.warning("Please fetch data first")
        return
    
    artifacts = st.session_state['artifacts']
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            # Extract
            df_metadata = extract_artifact_metadata(artifacts)
            df_media = transform_media_data(artifacts)
            df_colors = transform_color_data(artifacts)
            
            # Load
            load_metadata(df_metadata)
            st.success("ETL pipeline completed successfully")
            
            # Show statistics
            col1, col2, col3 = st.columns(3)
            col1.metric("Metadata Records", len(df_metadata))
            col2.metric("Media Records", len(df_media))
            col3.metric("Color Records", len(df_colors))

def show_analytics_dashboard():
    st.header("Analytics Dashboard")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        df_result = execute_query(query_name)
        
        if df_result is not None and not df_result.empty:
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) == 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=f"Analysis: {query_name.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Troubleshooting

### API Rate Limiting
```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff on rate limits"""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 429:  # Too Many Requests
            wait_time = 2 ** attempt
            time.sleep(wait_time)
            continue
        
        return response
    
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = create_db_connection()
        if connection and connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
        else:
            print("Failed to connect to database")
            return False
    except Error as e:
        print(f"Error: {e}")
        return False
```

### Handling Missing Data
```python
def clean_artifact_data(artifacts):
    """Clean and validate artifact data"""
    cleaned = []
    
    for artifact in artifacts:
        # Skip artifacts without essential fields
        if not artifact.get('objectid'):
            continue
        
        # Handle None values
        cleaned_artifact = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title') or 'Untitled',
            'culture': artifact.get('culture') or 'Unknown',
            'century': artifact.get('century') or 'Unknown'
        }
        cleaned.append(cleaned_artifact)
    
    return cleaned
```

This skill enables AI agents to guide developers through building complete data engineering pipelines using the Harvard Art Museums API, from data extraction to interactive analytics dashboards.
