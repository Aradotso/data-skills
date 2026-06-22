---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - how do I set up a data pipeline with the Harvard Art Museums API
  - help me build an ETL workflow for museum artifact data
  - show me how to create analytics dashboards with Streamlit and SQL
  - I want to extract and analyze Harvard museum collection data
  - guide me through building a data engineering app with museum APIs
  - how can I visualize artifact data from Harvard Art Museums
  - help me design SQL schemas for art museum collections
  - show me patterns for API-to-database ETL pipelines
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering App provides a complete data pipeline that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** data using predefined SQL queries
- **Visualizes** results through interactive Streamlit dashboards with Plotly charts

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

```bash
# Required Python packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Register at https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Add to `.env` file

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;

-- Create artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    creditline TEXT,
    description TEXT,
    provenance TEXT,
    technique TEXT
);

-- Create artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Key Components and Patterns

### 1. API Data Extraction

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
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages
def fetch_all_artifacts(total_records=500):
    """Fetch artifacts across multiple pages"""
    all_artifacts = []
    page_size = 100
    pages_needed = (total_records // page_size) + 1
    
    for page in range(1, pages_needed + 1):
        data = fetch_artifacts(page=page, size=page_size)
        all_artifacts.extend(data.get('records', []))
        
        if len(all_artifacts) >= total_records:
            break
    
    return all_artifacts[:total_records]
```

### 2. ETL Transform Pattern

```python
import pandas as pd

def transform_artifact_data(raw_artifacts):
    """Transform API response into normalized dataframes"""
    
    # Extract metadata
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Metadata transformation
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'creditline': artifact.get('creditline'),
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'technique': artifact.get('technique')
        }
        metadata_records.append(metadata)
        
        # Media transformation
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl'),
                'iiifbaseuri': artifact.get('iiifbaseuri'),
                'primaryimageurl': artifact.get('primaryimageurl')
            }
            media_records.append(media)
        
        # Color transformation
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Database Loading Pattern

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_data_to_db(metadata_df, media_df, colors_df):
    """Batch insert data into database"""
    conn = get_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, classification, department, 
         division, dated, creditline, description, provenance, technique)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, percent)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Image' 
                 ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY image_status
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_analytics_query(query_name):
    """Execute an analytical query and return results"""
    conn = get_db_connection()
    if not conn:
        return None
    
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
        return None
    
    try:
        df = pd.read_sql(query, conn)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        conn.close()
```

### 5. Streamlit Dashboard Pattern

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    st.sidebar.header("Navigation")
    page = st.sidebar.radio("Select Page", 
                            ["ETL Pipeline", "SQL Analytics", "Visualizations"])
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualizations_page()

def show_etl_page():
    st.header("📥 ETL Pipeline")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_all_artifacts(num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            success = load_data_to_db(metadata_df, media_df, colors_df)
            if success:
                st.success("Data loaded to database!")
            else:
                st.error("Failed to load data")

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_analytics_query(query_name)
            
            if df is not None and not df.empty:
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=query_name)
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No data returned")

def show_visualizations_page():
    st.header("📈 Data Visualizations")
    
    # Custom visualization examples
    df = execute_analytics_query("Top Cultures by Artifact Count")
    
    if df is not None:
        col1, col2 = st.columns(2)
        
        with col1:
            fig1 = px.bar(df, x='culture', y='artifact_count',
                         title="Artifacts by Culture")
            st.plotly_chart(fig1, use_container_width=True)
        
        with col2:
            fig2 = px.pie(df, names='culture', values='artifact_count',
                         title="Culture Distribution")
            st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Workflows

### Complete ETL Pipeline Execution

```python
# Full pipeline from API to database
def run_complete_pipeline(num_records=500):
    # Step 1: Extract
    print("Step 1: Extracting data from API...")
    artifacts = fetch_all_artifacts(num_records)
    
    # Step 2: Transform
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
    
    # Step 3: Load
    print("Step 3: Loading data to database...")
    success = load_data_to_db(metadata_df, media_df, colors_df)
    
    if success:
        print(f"✅ Pipeline completed: {num_records} artifacts processed")
    else:
        print("❌ Pipeline failed")
    
    return success
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff on rate limit"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except Exception as e:
            if "429" in str(e):  # Rate limit error
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        if conn and conn.is_connected():
            print("✅ Database connection successful")
            cursor = conn.cursor()
            cursor.execute("SELECT DATABASE()")
            db_name = cursor.fetchone()[0]
            print(f"Connected to database: {db_name}")
            conn.close()
            return True
        else:
            print("❌ Failed to connect to database")
            return False
    except Error as e:
        print(f"❌ Connection error: {e}")
        return False
```

### Handling Missing Data

```python
def safe_transform(value, default=None):
    """Safely handle None values in transformation"""
    return value if value is not None else default

# Usage in transformation
metadata = {
    'id': artifact.get('id'),
    'title': safe_transform(artifact.get('title'), 'Unknown'),
    'culture': safe_transform(artifact.get('culture'), 'Unknown'),
    # ... other fields
}
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement pagination** for large datasets to avoid memory issues
3. **Use batch inserts** instead of row-by-row for better performance
4. **Handle API rate limits** with exponential backoff
5. **Normalize data** into separate tables for efficient querying
6. **Create indexes** on foreign keys and frequently queried columns
7. **Log ETL operations** for debugging and monitoring
8. **Validate data** before loading to database

This skill provides comprehensive guidance for building production-ready data engineering pipelines with museum APIs, SQL databases, and interactive analytics dashboards.
