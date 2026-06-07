---
name: harvard-art-museums-etl-pipeline
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums data
  - set up data engineering workflow for museum artifacts
  - create analytics dashboard for Harvard Art Museums API
  - extract and transform Harvard museum collection data
  - build SQL database from Harvard Art Museums API
  - analyze museum artifacts with Python and Streamlit
  - create data pipeline for art collection metadata
  - visualize Harvard museum data with Plotly
---

# Harvard Art Museums ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App provides:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum data into relational SQL databases
- **Database Design**: Multi-table schema with artifact metadata, media, and color information
- **SQL Analytics**: 20+ predefined analytical queries for data insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time analytics

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Get your free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

The project supports MySQL or TiDB Cloud. Configure your database connection:

```python
# Database configuration (in your app or config file)
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

Environment variables:
```bash
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Core API Usage

### Fetching Artifacts from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, size=100, page=1):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Your Harvard API key
        size: Number of records per page (max 100)
        page: Page number for pagination
    
    Returns:
        JSON response with artifact data
    """
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=50, page=1)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages: {data['info']['pages']}")
print(f"Records fetched: {len(data['records'])}")
```

### Pagination Handler

```python
def fetch_all_artifacts(api_key, total_records=500):
    """
    Fetch multiple pages of artifacts with pagination
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < total_records:
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, size=size, page=page)
        
        all_artifacts.extend(data['records'])
        
        if page >= data['info']['pages']:
            break
        
        page += 1
    
    return all_artifacts[:total_records]
```

## ETL Pipeline Implementation

### Extract & Transform

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured DataFrames
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'period': artifact.get('period', 'Unknown'),
            'technique': artifact.get('technique', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': img.get('baseimageurl', ''),
                    'image_id': img.get('imageid', ''),
                    'copyright': img.get('copyright', '')
                }
                media_list.append(media)
        
        # Extract color information
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color_name': color.get('color', ''),
                    'color_hex': color.get('hex', ''),
                    'percentage': color.get('percent', 0)
                }
                colors_list.append(color_data)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### Load to SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema(connection):
    """
    Create tables for artifact data
    """
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            period VARCHAR(200),
            technique TEXT,
            medium TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(1000),
            image_id VARCHAR(100),
            copyright TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color_name VARCHAR(100),
            color_hex VARCHAR(10),
            percentage FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load DataFrames to MySQL database
    """
    try:
        connection = mysql.connector.connect(**db_config)
        create_database_schema(connection)
        
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (artifact_id, title, culture, century, classification, 
                 department, dated, period, technique, medium)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, image_url, image_id, copyright)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color_name, color_hex, percentage)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts to database")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## Streamlit Dashboard Implementation

### Basic App Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv
import os

load_dotenv()

st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🎨",
    layout="wide"
)

st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")
st.markdown("---")

# Sidebar configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", 
                            value=os.getenv('HARVARD_API_KEY', ''), 
                            type="password")
    
    st.header("Database Settings")
    db_host = st.text_input("Host", value=os.getenv('DB_HOST', 'localhost'))
    db_user = st.text_input("User", value=os.getenv('DB_USER', 'root'))
    db_password = st.text_input("Password", type="password")
    db_name = st.text_input("Database", value="harvard_artifacts")

# Main tabs
tab1, tab2, tab3 = st.tabs(["📥 Data Collection", "📊 Analytics", "📈 Visualizations"])

with tab1:
    st.header("Collect Artifact Data")
    
    col1, col2 = st.columns(2)
    with col1:
        num_records = st.number_input("Number of Records", 
                                      min_value=10, max_value=1000, 
                                      value=100, step=10)
    
    if st.button("🚀 Start ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            # Your ETL logic here
            artifacts = fetch_all_artifacts(api_key, num_records)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
            st.success(f"✅ Collected {len(metadata_df)} artifacts")
            st.dataframe(metadata_df.head())
```

### Analytics Queries

```python
def run_analytics_query(connection, query_name):
    """
    Execute predefined analytical queries
    """
    queries = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 15
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century != 'Unknown'
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "media_availability": """
            SELECT 
                CASE WHEN COUNT(m.image_url) > 0 THEN 'Has Media' 
                     ELSE 'No Media' END as media_status,
                COUNT(DISTINCT a.artifact_id) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.artifact_id = m.artifact_id
            GROUP BY media_status
        """,
        
        "top_colors": """
            SELECT color_name, COUNT(*) as usage_count, 
                   AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color_name
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        "department_distribution": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department != 'Unknown'
            GROUP BY department
            ORDER BY artifact_count DESC
        """
    }
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)

# In Streamlit app
with tab2:
    st.header("SQL Analytics")
    
    query_option = st.selectbox(
        "Select Analysis",
        ["artifacts_by_culture", "artifacts_by_century", 
         "media_availability", "top_colors", "department_distribution"]
    )
    
    if st.button("Run Query"):
        connection = mysql.connector.connect(**db_config)
        df = run_analytics_query(connection, query_option)
        connection.close()
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df) > 0 and len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=query_option.replace('_', ' ').title())
            st.plotly_chart(fig, use_container_width=True)
```

### Visualization Patterns

```python
import plotly.express as px
import plotly.graph_objects as go

def create_culture_distribution_chart(df):
    """
    Create interactive bar chart for culture distribution
    """
    fig = px.bar(
        df,
        x='culture',
        y='count',
        title='Artifact Distribution by Culture',
        labels={'culture': 'Culture', 'count': 'Number of Artifacts'},
        color='count',
        color_continuous_scale='Viridis'
    )
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_pie_chart(df):
    """
    Create pie chart for color distribution
    """
    fig = px.pie(
        df,
        values='usage_count',
        names='color_name',
        title='Top Colors in Artifact Collection'
    )
    return fig

def create_timeline_chart(df):
    """
    Create timeline visualization for artifacts by century
    """
    fig = go.Figure(data=[
        go.Bar(x=df['century'], y=df['count'],
               marker_color='indianred')
    ])
    fig.update_layout(
        title='Artifacts Across Centuries',
        xaxis_title='Century',
        yaxis_title='Count'
    )
    return fig

# Usage in Streamlit
with tab3:
    st.header("Data Visualizations")
    
    viz_type = st.radio(
        "Visualization Type",
        ["Culture Distribution", "Color Analysis", "Century Timeline"]
    )
    
    if viz_type == "Culture Distribution":
        df = run_analytics_query(connection, "artifacts_by_culture")
        fig = create_culture_distribution_chart(df)
        st.plotly_chart(fig, use_container_width=True)
```

## Complete ETL Workflow

```python
def complete_etl_pipeline(api_key, db_config, num_records=500):
    """
    Run complete ETL pipeline from API to Database
    """
    print("Step 1: Extracting data from Harvard API...")
    raw_artifacts = fetch_all_artifacts(api_key, num_records)
    
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)
    
    print("Step 3: Loading to database...")
    load_to_database(metadata_df, media_df, colors_df, db_config)
    
    print("✅ ETL Pipeline completed successfully!")
    
    return {
        'metadata_records': len(metadata_df),
        'media_records': len(media_df),
        'color_records': len(colors_df)
    }

# Execute pipeline
if __name__ == "__main__":
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME'),
        'port': int(os.getenv('DB_PORT', 3306))
    }
    
    api_key = os.getenv('HARVARD_API_KEY')
    
    stats = complete_etl_pipeline(api_key, db_config, num_records=200)
    print(f"Pipeline Stats: {stats}")
```

## Running the Application

```bash
# Run the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(df, table_name, connection, batch_size=1000):
    """
    Insert DataFrame to SQL in batches for better performance
    """
    cursor = connection.cursor()
    
    for start in range(0, len(df), batch_size):
        batch = df.iloc[start:start + batch_size]
        # Insert batch
        for _, row in batch.iterrows():
            # Your insert logic
            pass
        connection.commit()
        print(f"Inserted batch {start//batch_size + 1}")
    
    cursor.close()
```

### Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key, size=100, page=1, max_retries=3):
    """
    Fetch artifacts with retry logic for failed requests
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, size, page)
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                print(f"Request failed. Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Data Validation

```python
def validate_artifact_data(df):
    """
    Validate artifact data before loading to database
    """
    # Check for required columns
    required_cols = ['artifact_id', 'title']
    missing_cols = set(required_cols) - set(df.columns)
    if missing_cols:
        raise ValueError(f"Missing columns: {missing_cols}")
    
    # Check for null artifact_ids
    if df['artifact_id'].isnull().any():
        raise ValueError("Found null artifact_ids")
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['artifact_id'])
    
    return df
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_with_rate_limit(api_key, delay=1.0):
    """
    Add delay between API requests to avoid rate limiting
    """
    time.sleep(delay)
    return fetch_artifacts(api_key)
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    """
    Test database connectivity
    """
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✅ Database connection successful")
            db_info = connection.get_server_info()
            print(f"MySQL Server version: {db_info}")
            connection.close()
            return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def process_in_chunks(api_key, total_records, chunk_size=100):
    """
    Process large datasets in chunks to manage memory
    """
    for offset in range(0, total_records, chunk_size):
        page = (offset // 100) + 1
        chunk = fetch_artifacts(api_key, size=chunk_size, page=page)
        
        # Process chunk immediately
        metadata_df, media_df, colors_df = transform_artifacts(chunk['records'])
        load_to_database(metadata_df, media_df, colors_df, db_config)
        
        print(f"Processed records {offset} to {offset + chunk_size}")
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API.
