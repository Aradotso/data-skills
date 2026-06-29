---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics apps using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - show me how to use the Harvard Art Museums API with Python
  - help me create a data analytics app with Streamlit and museum data
  - how to extract and transform Harvard museum artifact data
  - build a SQL database from Harvard Art Museums API
  - create interactive visualizations for museum collection data
  - set up an end-to-end data engineering pipeline with Streamlit
  - analyze Harvard Art Museums data with SQL queries
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Store artifacts in normalized tables (metadata, media, colors)
- **SQL Analytics**: Execute 20+ predefined analytical queries
- **Interactive Dashboards**: Visualize results using Streamlit and Plotly

## Architecture Flow

```
Harvard API → Python ETL → SQL Database → Analytics Queries → Streamlit UI → Plotly Charts
```

## Installation

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

### Required Dependencies

```python
# requirements.txt typical contents
streamlit>=1.28.0
pandas>=2.0.0
requests>=2.31.0
mysql-connector-python>=8.0.33
plotly>=5.17.0
python-dotenv>=1.0.0
```

## Configuration

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'
```

### Database Connection

```python
import mysql.connector
import os

def get_db_connection():
    """Create MySQL database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## ETL Pipeline Implementation

### 1. Extract: Fetch Data from Harvard API

```python
import requests
import time

def fetch_artifacts(api_key, max_pages=5, per_page=100):
    """
    Extract artifact data from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        max_pages: Number of pages to fetch
        per_page: Records per page (max 100)
    
    Returns:
        List of artifact records
    """
    artifacts = []
    base_url = 'https://api.harvardartmuseums.org/object'
    
    for page in range(1, max_pages + 1):
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{max_pages}: {len(data.get('records', []))} records")
            
            # Rate limiting - respect API limits
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts
```

### 2. Transform: Normalize JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into normalized DataFrames
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata_records.append({
            'id': artifact_id,
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline')
        })
        
        # Extract media/images
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact_id,
                'image_id': image.get('imageid'),
                'base_url': image.get('baseimageurl'),
                'width': image.get('width'),
                'height': image.get('height'),
                'format': image.get('format')
            })
        
        # Extract color data
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### 3. Load: Insert Data into SQL Database

```python
def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            description TEXT,
            medium TEXT,
            dimensions VARCHAR(500),
            creditline TEXT
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url TEXT,
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent DECIMAL(5,2),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_data(connection, metadata_df, media_df, colors_df):
    """Batch insert DataFrames into database"""
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, division, 
             dated, description, medium, dimensions, creditline)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, image_id, base_url, width, height, format)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
```

## Complete ETL Execution

```python
import os
from dotenv import load_dotenv

def run_etl_pipeline():
    """Execute full ETL pipeline"""
    load_dotenv()
    
    # Extract
    print("Extracting data from Harvard API...")
    api_key = os.getenv('HARVARD_API_KEY')
    artifacts = fetch_artifacts(api_key, max_pages=10)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    print(f"Metadata: {len(metadata_df)} rows")
    print(f"Media: {len(media_df)} rows")
    print(f"Colors: {len(colors_df)} rows")
    
    # Load
    print("Loading data to database...")
    conn = get_db_connection()
    create_tables(conn)
    load_data(conn, metadata_df, media_df, colors_df)
    conn.close()
    
    print("ETL pipeline completed successfully!")

if __name__ == "__main__":
    run_etl_pipeline()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by culture
ARTIFACTS_BY_CULTURE = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 20
"""

# Query 2: Century distribution
ARTIFACTS_BY_CENTURY = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Department breakdown
ARTIFACTS_BY_DEPARTMENT = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    GROUP BY department
    ORDER BY count DESC
"""

# Query 4: Media availability
ARTIFACTS_WITH_IMAGES = """
    SELECT 
        CASE WHEN image_id IS NOT NULL THEN 'With Images' ELSE 'Without Images' END as status,
        COUNT(DISTINCT m.id) as count
    FROM artifactmetadata m
    LEFT JOIN artifactmedia am ON m.id = am.artifact_id
    GROUP BY status
"""

# Query 5: Color analysis
TOP_COLORS = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percentage
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 6: Classification distribution
CLASSIFICATION_STATS = """
    SELECT classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE classification IS NOT NULL
    GROUP BY classification
    ORDER BY count DESC
    LIMIT 20
"""
```

### Execute Queries

```python
def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

def run_analytics():
    """Run all analytical queries"""
    conn = get_db_connection()
    
    results = {
        'by_culture': execute_query(conn, ARTIFACTS_BY_CULTURE),
        'by_century': execute_query(conn, ARTIFACTS_BY_CENTURY),
        'by_department': execute_query(conn, ARTIFACTS_BY_DEPARTMENT),
        'with_images': execute_query(conn, ARTIFACTS_WITH_IMAGES),
        'top_colors': execute_query(conn, TOP_COLORS)
    }
    
    conn.close()
    return results
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Data Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["ETL Pipeline", "SQL Analytics", "Data Visualization"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Data Visualization":
        show_visualization_page()

def show_etl_page():
    """ETL Pipeline UI"""
    st.header("Extract, Transform, Load Pipeline")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
        per_page = st.number_input("Records per page", 10, 100, 100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            try:
                run_etl_pipeline()
                st.success("ETL pipeline completed successfully!")
            except Exception as e:
                st.error(f"Error: {str(e)}")

def show_analytics_page():
    """SQL Analytics UI"""
    st.header("SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": ARTIFACTS_BY_CULTURE,
        "Artifacts by Century": ARTIFACTS_BY_CENTURY,
        "Artifacts by Department": ARTIFACTS_BY_DEPARTMENT,
        "Media Availability": ARTIFACTS_WITH_IMAGES,
        "Top Colors": TOP_COLORS
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        conn = get_db_connection()
        df = execute_query(conn, queries[selected_query])
        conn.close()
        
        st.dataframe(df, use_container_width=True)
        
        # Auto-generate visualization
        if len(df.columns) == 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

def show_visualization_page():
    """Interactive Visualizations"""
    st.header("Data Visualizations")
    
    conn = get_db_connection()
    
    # Culture distribution
    st.subheader("Artifacts by Culture")
    df_culture = execute_query(conn, ARTIFACTS_BY_CULTURE)
    fig1 = px.bar(df_culture, x='culture', y='count', 
                  title='Top 20 Cultures by Artifact Count')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Century timeline
    st.subheader("Artifacts by Century")
    df_century = execute_query(conn, ARTIFACTS_BY_CENTURY)
    fig2 = px.bar(df_century, x='century', y='count',
                  title='Artifact Distribution Across Centuries')
    st.plotly_chart(fig2, use_container_width=True)
    
    # Color analysis
    st.subheader("Color Usage Analysis")
    df_colors = execute_query(conn, TOP_COLORS)
    fig3 = px.bar(df_colors, x='color', y='usage_count',
                  title='Most Common Colors in Artifacts')
    st.plotly_chart(fig3, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

### Run Streamlit App

```bash
streamlit run app.py
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def incremental_load(connection, last_sync_date):
    """Load only new artifacts since last sync"""
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'lastmodified': f'>{last_sync_date}'
    }
    
    response = requests.get(BASE_URL, params=params)
    new_artifacts = response.json().get('records', [])
    
    # Transform and load only new records
    metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
    load_data(connection, metadata_df, media_df, colors_df)
```

### Pattern 2: Data Quality Checks

```python
def validate_data(df):
    """Validate DataFrame before loading"""
    checks = {
        'null_ids': df['id'].isnull().sum(),
        'duplicate_ids': df['id'].duplicated().sum(),
        'empty_titles': df['title'].isnull().sum()
    }
    
    for check, count in checks.items():
        if count > 0:
            st.warning(f"{check}: {count} issues found")
    
    return all(count == 0 for count in checks.values())
```

### Pattern 3: Parameterized Queries

```python
def get_artifacts_by_filter(connection, culture=None, century=None, department=None):
    """Flexible filtering with dynamic WHERE clause"""
    query = "SELECT * FROM artifactmetadata WHERE 1=1"
    params = []
    
    if culture:
        query += " AND culture = %s"
        params.append(culture)
    
    if century:
        query += " AND century = %s"
        params.append(century)
    
    if department:
        query += " AND department = %s"
        params.append(department)
    
    cursor = connection.cursor()
    cursor.execute(query, params)
    results = cursor.fetchall()
    cursor.close()
    
    return results
```

## Troubleshooting

### API Rate Limiting

```python
# Add exponential backoff
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def get_session_with_retries():
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues

```python
# Connection pooling for better performance
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_pooled_connection():
    return db_pool.get_connection()
```

### Memory Management for Large Datasets

```python
def batch_process_artifacts(api_key, total_pages, batch_size=5):
    """Process data in batches to avoid memory issues"""
    conn = get_db_connection()
    create_tables(conn)
    
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        # Process batch
        artifacts = fetch_artifacts(api_key, max_pages=batch_end-batch_start, 
                                   start_page=batch_start)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_data(conn, metadata_df, media_df, colors_df)
        
        print(f"Processed batch {batch_start}-{batch_end-1}")
        
        # Clear memory
        del artifacts, metadata_df, media_df, colors_df
    
    conn.close()
```

### Handling Missing Data

```python
def safe_transform(artifact):
    """Handle missing fields gracefully"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Unknown'),
        'culture': artifact.get('culture', 'Unknown'),
        'century': artifact.get('century', 'Unknown'),
        'classification': artifact.get('classification', 'Uncategorized'),
        'department': artifact.get('department', 'General'),
        'dated': artifact.get('dated', 'Undated'),
        'description': artifact.get('description', '')[:1000]  # Truncate long text
    }
```

This skill provides comprehensive guidance for building ETL pipelines and analytics applications using the Harvard Art Museums API with Python, SQL, and Streamlit.
