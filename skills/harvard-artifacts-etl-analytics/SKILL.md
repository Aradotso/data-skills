---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - process Harvard museum collection data
  - set up artifact data pipeline with MySQL
  - visualize museum data with Plotly
  - query Harvard Art Museums API with pagination
  - transform nested JSON museum data to SQL tables
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project provides an end-to-end data engineering solution for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL analytics, and interactive visualization using Streamlit.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational database tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics**: Executes 20+ predefined analytical queries
- **Visualization**: Creates interactive dashboards using Streamlit and Plotly

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

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Database Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema():
    """Initialize the database schema for artifact storage"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    
    cursor = connection.cursor()
    
    # Create database
    cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
    cursor.execute(f"USE {os.getenv('DB_NAME')}")
    
    # Create artifacts metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            medium VARCHAR(500),
            technique VARCHAR(500),
            url VARCHAR(500),
            INDEX idx_culture (culture),
            INDEX idx_century (century),
            INDEX idx_department (department)
        )
    """)
    
    # Create media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(50),
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
            INDEX idx_artifact (artifact_id)
        )
    """)
    
    # Create colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent DECIMAL(5,2),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id) ON DELETE CASCADE,
            INDEX idx_artifact (artifact_id),
            INDEX idx_color (color)
        )
    """)
    
    connection.commit()
    cursor.close()
    connection.close()
```

## API Integration

### Fetching Artifact Data

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response with artifact data
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts with pagination"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            
            # Check if more pages available
            if data['info']['page'] >= data['info']['pages']:
                break
        else:
            break
    
    return all_artifacts
```

## ETL Pipeline

### Extract and Transform

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """
    Transform nested JSON artifact data into normalized DataFrames
    
    Args:
        artifacts: List of artifact dictionaries from API
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'url': artifact.get('url', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact['id'],
                    'media_type': 'image',
                    'baseimageurl': img.get('baseimageurl', '')[:500],
                    'iiifbaseuri': img.get('iiifbaseuri', '')[:500]
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color_obj in artifact['colors']:
                color = {
                    'artifact_id': artifact['id'],
                    'color': color_obj.get('color', '')[:50],
                    'spectrum': color_obj.get('spectrum', '')[:50],
                    'percent': color_obj.get('percent', 0.0)
                }
                colors_list.append(color)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load to Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert DataFrames into MySQL database
    
    Args:
        metadata_df: Artifact metadata DataFrame
        media_df: Media/images DataFrame
        colors_df: Color data DataFrame
    """
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Insert metadata (batch)
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, medium, technique, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    metadata_values = metadata_df.values.tolist()
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media
    if not media_df.empty:
        media_query = """
            INSERT INTO artifactmedia (artifact_id, media_type, baseimageurl, iiifbaseuri)
            VALUES (%s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
    
    connection.commit()
    cursor.close()
    connection.close()
    
    print(f"Inserted {len(metadata_df)} artifacts, {len(media_df)} media, {len(colors_df)} colors")
```

## Analytics Queries

### Sample Analytical Queries

```python
# Common analytics queries for the dataset

ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "top_colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "artifacts_with_media": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as with_media,
            COUNT(DISTINCT a.id) - COUNT(DISTINCT m.artifact_id) as without_media
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
    """,
    
    "culture_color_analysis": """
        SELECT 
            a.culture,
            c.color,
            COUNT(*) as usage_count,
            AVG(c.percent) as avg_percent
        FROM artifactmetadata a
        JOIN artifactcolors c ON a.id = c.artifact_id
        WHERE a.culture IS NOT NULL AND a.culture != ''
        GROUP BY a.culture, c.color
        HAVING COUNT(*) > 5
        ORDER BY a.culture, usage_count DESC
    """
}

def execute_query(query_name):
    """Execute predefined analytical query"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICS_QUERIES.get(query_name)
    df = pd.read_sql(query, connection)
    connection.close()
    
    return df
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
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "Analytics":
        show_analytics()
    elif page == "Visualizations":
        show_visualizations()

def show_data_collection():
    """ETL pipeline interface"""
    st.header("📥 Data Collection & ETL")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 50, 5)
    
    with col2:
        if st.button("Start ETL Pipeline"):
            with st.spinner("Fetching artifacts..."):
                artifacts = fetch_all_artifacts(max_pages=num_pages)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                load_to_database(metadata_df, media_df, colors_df)
                st.success("Data loaded successfully!")
            
            st.balloons()

def show_analytics():
    """Run and display analytical queries"""
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_query(query_name)
            
            st.subheader("Query Results")
            st.dataframe(df, use_container_width=True)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(20),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=f"{query_name.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """Custom visualizations"""
    st.header("📈 Data Visualizations")
    
    # Culture distribution
    df_culture = execute_query("artifacts_by_culture")
    fig1 = px.bar(df_culture, x='culture', y='count', title='Artifacts by Culture')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    df_colors = execute_query("top_colors")
    fig2 = px.pie(df_colors, values='frequency', names='color', title='Color Distribution')
    st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Running the Streamlit App

```bash
# Run the application
streamlit run app.py

# With custom port
streamlit run app.py --server.port 8080

# With custom host
streamlit run app.py --server.address 0.0.0.0
```

## Common Patterns

### Complete ETL Workflow

```python
from dotenv import load_dotenv
import os

# Load environment variables
load_dotenv()

# Complete workflow
def run_etl_pipeline(num_pages=10):
    """Execute complete ETL pipeline"""
    
    # 1. Extract
    print("Step 1: Extracting data from API...")
    artifacts = fetch_all_artifacts(max_pages=num_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # 2. Transform
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
    print(f"Transformed into {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
    
    # 3. Load
    print("Step 3: Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    print("ETL pipeline complete!")
    
    return metadata_df, media_df, colors_df

# Run pipeline
if __name__ == "__main__":
    create_database_schema()
    run_etl_pipeline(num_pages=5)
```

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID already in database"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    connection.close()
    
    return result[0] if result[0] else 0

def incremental_load():
    """Load only new artifacts not in database"""
    last_id = get_last_artifact_id()
    artifacts = fetch_all_artifacts(max_pages=5)
    
    # Filter to only new artifacts
    new_artifacts = [a for a in artifacts if a['id'] > last_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifact_data(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(page, max_retries=3):
    """Fetch with exponential backoff for rate limiting"""
    for attempt in range(max_retries):
        try:
            response = fetch_artifacts(page=page)
            return response
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = (2 ** attempt) * 5  # Exponential backoff
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
def test_database_connection():
    """Verify database connectivity"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            connect_timeout=10
        )
        
        cursor = connection.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        connection.close()
        
        print("✅ Database connection successful")
        return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_extract(artifact, field, default='', max_length=None):
    """Safely extract field with default and length limiting"""
    value = artifact.get(field, default)
    if value is None:
        value = default
    if isinstance(value, str) and max_length:
        value = value[:max_length]
    return value

# Usage in transformation
metadata = {
    'id': artifact.get('id'),
    'title': safe_extract(artifact, 'title', default='Untitled', max_length=500),
    'culture': safe_extract(artifact, 'culture', max_length=255),
    'century': safe_extract(artifact, 'century', max_length=100)
}
```
