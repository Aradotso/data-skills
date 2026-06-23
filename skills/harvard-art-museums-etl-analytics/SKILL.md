---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - show me how to create a data engineering app with Harvard artifacts
  - help me set up a Streamlit analytics dashboard for museum data
  - how to extract and transform Harvard Art Museums data into SQL
  - create a data pipeline for Harvard artifacts collection
  - build an interactive analytics app for museum artifact data
  - help me visualize Harvard Art Museums data with Plotly
  - set up an end-to-end data engineering project with museum API
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Connects to Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts artifact metadata, transforms nested JSON into relational tables, loads into SQL
- **Database Layer**: Structured MySQL/TiDB schema with foreign key relationships
- **Analytics Queries**: 20+ predefined SQL queries for artifact insights
- **Visualization Dashboard**: Interactive Streamlit app with Plotly charts

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

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

### 1. API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### 2. Database Configuration

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

### 3. Database Schema

```sql
-- Artifact Metadata Table
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
    url TEXT,
    description TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    caption TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Extract artifacts from Harvard Art Museums API
    
    Args:
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        dict: API response with artifact records
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(data.get('records', []))
        
        # Check if more pages available
        if page >= data.get('info', {}).get('pages', 0):
            break
    
    return all_artifacts
```

### Transform: Clean and Structure Data

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data into metadata DataFrame"""
    metadata_records = []
    
    for artifact in artifacts:
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'period': artifact.get('period', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'dated': artifact.get('dated', ''),
            'url': artifact.get('url', ''),
            'description': artifact.get('description', '')
        })
    
    return pd.DataFrame(metadata_records)

def transform_media(artifacts):
    """Transform nested media/image data"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_records.append({
                'artifact_id': artifact_id,
                'image_url': image.get('baseimageurl', ''),
                'caption': image.get('caption', ''),
                'media_type': 'image'
            })
    
    return pd.DataFrame(media_records)

def transform_colors(artifacts):
    """Transform color data"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_records)
```

### Load: Insert into SQL Database

```python
def load_to_sql(df, table_name, connection):
    """
    Load DataFrame to SQL table using batch insert
    
    Args:
        df: pandas DataFrame
        table_name: Target SQL table name
        connection: MySQL connection object
    """
    cursor = connection.cursor()
    
    # Prepare column names and placeholders
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    connection.commit()
    cursor.close()
    
    print(f"Loaded {len(df)} records into {table_name}")

def run_etl_pipeline():
    """Complete ETL pipeline execution"""
    # Extract
    print("Extracting data from API...")
    artifacts = fetch_all_artifacts(max_pages=5)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Loading data to SQL...")
    connection = get_db_connection()
    
    try:
        load_to_sql(metadata_df, 'artifactmetadata', connection)
        load_to_sql(media_df, 'artifactmedia', connection)
        load_to_sql(colors_df, 'artifactcolors', connection)
    finally:
        connection.close()
    
    print("ETL pipeline completed successfully!")
```

## Analytics Queries

### Sample SQL Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10;
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC;
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC;
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15;
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_items
        FROM artifactmedia m;
    """,
    
    "Artifacts with Multiple Images": """
        SELECT artifact_id, COUNT(*) as image_count
        FROM artifactmedia
        GROUP BY artifact_id
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20;
    """,
    
    "Color Spectrum Distribution": """
        SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_coverage
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY count DESC;
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    connection = get_db_connection()
    cursor = connection.cursor(dictionary=True)
    
    query = ANALYTICS_QUERIES.get(query_name)
    cursor.execute(query)
    results = cursor.fetchall()
    
    cursor.close()
    connection.close()
    
    return pd.DataFrame(results)
```

## Streamlit Dashboard

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
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a section",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualization_page()

def show_etl_page():
    """ETL Pipeline interface"""
    st.header("📥 ETL Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        max_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
    
    with col2:
        st.write("")
        st.write("")
        run_button = st.button("🚀 Run ETL Pipeline", type="primary")
    
    if run_button:
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline()
                st.success("✅ ETL pipeline completed successfully!")
            except Exception as e:
                st.error(f"❌ Error: {str(e)}")

def show_analytics_page():
    """SQL Analytics interface"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select a query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Show the SQL query
    with st.expander("View SQL Query"):
        st.code(ANALYTICS_QUERIES[query_name], language="sql")
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            results_df = execute_query(query_name)
            
            st.subheader("Query Results")
            st.dataframe(results_df, use_container_width=True)
            
            # Auto-generate visualization
            if len(results_df) > 0 and len(results_df.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualization_page():
    """Custom visualizations"""
    st.header("📈 Data Visualizations")
    
    viz_type = st.selectbox(
        "Choose visualization",
        ["Culture Distribution", "Color Analysis", "Department Overview"]
    )
    
    if viz_type == "Culture Distribution":
        df = execute_query("Artifacts by Culture")
        fig = px.pie(
            df,
            values=df.columns[1],
            names=df.columns[0],
            title="Artifacts Distribution by Culture"
        )
        st.plotly_chart(fig, use_container_width=True)
    
    elif viz_type == "Color Analysis":
        df = execute_query("Top Colors Used")
        fig = px.bar(
            df,
            x='color',
            y='frequency',
            color='avg_percent',
            title="Most Common Colors in Artifacts"
        )
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Running the Dashboard

```bash
streamlit run app.py
```

The dashboard will be available at `http://localhost:8501`

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline():
    """ETL with comprehensive error handling"""
    try:
        # Extract
        artifacts = fetch_all_artifacts(max_pages=5)
        if not artifacts:
            raise ValueError("No artifacts fetched from API")
        
        # Transform
        metadata_df = transform_metadata(artifacts)
        media_df = transform_media(artifacts)
        colors_df = transform_colors(artifacts)
        
        # Validate data
        if metadata_df.empty:
            raise ValueError("Metadata transformation resulted in empty DataFrame")
        
        # Load
        connection = get_db_connection()
        try:
            load_to_sql(metadata_df, 'artifactmetadata', connection)
            load_to_sql(media_df, 'artifactmedia', connection)
            load_to_sql(colors_df, 'artifactcolors', connection)
        finally:
            connection.close()
        
        return True, "ETL completed successfully"
        
    except requests.exceptions.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

### Incremental Data Loading

```python
def incremental_load():
    """Load only new artifacts since last update"""
    connection = get_db_connection()
    cursor = connection.cursor()
    
    # Get max ID from existing data
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    max_id = cursor.fetchone()[0] or 0
    
    # Fetch only newer artifacts
    artifacts = fetch_all_artifacts(max_pages=5)
    new_artifacts = [a for a in artifacts if a.get('id', 0) > max_id]
    
    if new_artifacts:
        metadata_df = transform_metadata(new_artifacts)
        load_to_sql(metadata_df, 'artifactmetadata', connection)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
    
    cursor.close()
    connection.close()
```

### Custom Query Builder

```python
def build_custom_query(filters):
    """
    Build dynamic SQL query based on filters
    
    Args:
        filters: dict with keys like 'culture', 'century', 'department'
    """
    base_query = "SELECT * FROM artifactmetadata WHERE 1=1"
    params = []
    
    if filters.get('culture'):
        base_query += " AND culture = %s"
        params.append(filters['culture'])
    
    if filters.get('century'):
        base_query += " AND century = %s"
        params.append(filters['century'])
    
    if filters.get('department'):
        base_query += " AND department = %s"
        params.append(filters['department'])
    
    connection = get_db_connection()
    cursor = connection.cursor(dictionary=True)
    cursor.execute(base_query, params)
    results = cursor.fetchall()
    
    cursor.close()
    connection.close()
    
    return pd.DataFrame(results)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(page, size=100, delay=0.5):
    """Fetch with delay to respect rate limits"""
    time.sleep(delay)
    return fetch_artifacts(page, size)
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = get_db_connection()
        cursor = connection.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        connection.close()
        return True, "Connection successful"
    except Exception as e:
        return False, f"Connection failed: {str(e)}"
```

### Empty or Missing Data

```python
def validate_artifact_data(artifact):
    """Validate artifact has required fields"""
    required_fields = ['id', 'title']
    return all(artifact.get(field) for field in required_fields)

def safe_transform(artifacts):
    """Transform only valid artifacts"""
    valid_artifacts = [a for a in artifacts if validate_artifact_data(a)]
    print(f"Valid artifacts: {len(valid_artifacts)}/{len(artifacts)}")
    return transform_metadata(valid_artifacts)
```

### Duplicate Key Errors

```python
def upsert_to_sql(df, table_name, connection):
    """Insert or update on duplicate key"""
    cursor = connection.cursor()
    
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    # Create update clause for all columns except id
    update_cols = [c for c in df.columns if c != 'id']
    update_clause = ', '.join([f"{col}=VALUES({col})" for col in update_cols])
    
    query = f"""
        INSERT INTO {table_name} ({columns}) 
        VALUES ({placeholders})
        ON DUPLICATE KEY UPDATE {update_clause}
    """
    
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(query, data_tuples)
    connection.commit()
    cursor.close()
```

This skill provides complete guidance for building ETL pipelines and analytics dashboards with the Harvard Art Museums API, covering all aspects from data extraction to interactive visualization.
