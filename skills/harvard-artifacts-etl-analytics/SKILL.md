---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data with SQL storage and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to create a data engineering project with museum artifacts
  - help me set up a Streamlit analytics dashboard for API data
  - how do I extract and transform Harvard museum data into SQL
  - build an end-to-end data pipeline with API, SQL, and visualization
  - create analytics queries for Harvard Art Museums collection
  - set up a museum artifacts data engineering application
  - integrate Harvard API with SQL database and Streamlit
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases (MySQL/TiDB), and building interactive analytics dashboards with Streamlit and Plotly.

The application handles:
- API pagination and rate limiting
- Nested JSON to relational table transformation
- Batch SQL inserts for performance
- 20+ predefined analytical queries
- Interactive visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

### Environment Variables

Create a `.env` file or set environment variables:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Obtaining Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Store in environment variable or Streamlit secrets

### Streamlit Secrets

For deployment, use `.streamlit/secrets.toml`:

```toml
[api]
harvard_key = "your_api_key"

[database]
host = "your_host"
user = "your_user"
password = "your_password"
database = "harvard_artifacts"
port = 3306
```

## Database Schema

The project uses three main tables:

### artifactmetadata

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    dated VARCHAR(200),
    period VARCHAR(200),
    provenance TEXT
);
```

### artifactmedia

```sql
CREATE TABLE artifactmedia (
    artifact_id INT,
    has_images BOOLEAN,
    primary_image_url VARCHAR(1000),
    image_count INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### artifactcolors

```sql
CREATE TABLE artifactcolors (
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Code Patterns

### API Data Extraction

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
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
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {info['totalrecords']}")
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'provenance': artifact.get('provenance')
        }
        metadata_list.append(metadata)
        
        # Extract media info
        media = {
            'artifact_id': artifact.get('id'),
            'has_images': artifact.get('primaryimageurl') is not None,
            'primary_image_url': artifact.get('primaryimageurl'),
            'image_count': len(artifact.get('images', []))
        }
        media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            colors_list.append({
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Usage
metadata_df, media_df, colors_df = transform_artifacts(artifacts)
```

### Database Connection and Loading

```python
def get_db_connection():
    """
    Create MySQL database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        port=int(os.getenv('DB_PORT', 3306))
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Batch insert dataframes into SQL tables
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         division, technique, medium, dimensions, dated, period, provenance)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, has_images, primary_image_url, image_count)
        VALUES (%s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE image_count=VALUES(image_count)
    """
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors 
        (artifact_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(colors_query, colors_df.values.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
```

### Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=5, records_per_page=100):
    """
    Execute full ETL pipeline
    """
    api_key = os.getenv('HARVARD_API_KEY')
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        artifacts, info = fetch_artifacts(api_key, page=page, size=records_per_page)
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        all_metadata.append(metadata_df)
        all_media.append(media_df)
        all_colors.append(colors_df)
    
    # Concatenate all dataframes
    final_metadata = pd.concat(all_metadata, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load
    load_to_database(final_metadata, final_media, final_colors)
    
    print(f"ETL Complete: {len(final_metadata)} artifacts loaded")
```

## Streamlit Application

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")

# Sidebar for API key and database config
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                             value=st.secrets.get("api", {}).get("harvard_key", ""))
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            run_etl_pipeline(num_pages=5)
        st.success("ETL Pipeline completed successfully!")

# Main content
tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔍 Query Builder", "📈 Visualizations"])

with tab1:
    st.header("Predefined Analytics Queries")
    # Query selection and execution
```

### Analytics Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors Across Collection": """
        SELECT color_hex, 
               COUNT(DISTINCT artifact_id) as artifact_count,
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Media Coverage Analysis": """
        SELECT has_images, COUNT(*) as count
        FROM artifactmedia
        GROUP BY has_images
    """,
    
    "Artifacts with Most Images": """
        SELECT am.title, am.culture, ame.image_count
        FROM artifactmetadata am
        JOIN artifactmedia ame ON am.id = ame.artifact_id
        WHERE ame.image_count > 0
        ORDER BY ame.image_count DESC
        LIMIT 20
    """
}

def execute_query(query_name):
    """
    Execute selected analytics query
    """
    conn = get_db_connection()
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# In Streamlit app
query_choice = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))

if st.button("Run Query"):
    results_df = execute_query(query_choice)
    
    st.dataframe(results_df)
    
    # Auto-visualization
    if len(results_df.columns) == 2:
        fig = px.bar(results_df, 
                     x=results_df.columns[0], 
                     y=results_df.columns[1],
                     title=query_choice)
        st.plotly_chart(fig, use_container_width=True)
```

### Visualization Patterns

```python
def create_culture_chart(df):
    """
    Create interactive bar chart for culture distribution
    """
    fig = px.bar(df, 
                 x='culture', 
                 y='artifact_count',
                 title='Artifact Distribution by Culture',
                 color='artifact_count',
                 color_continuous_scale='Viridis')
    
    fig.update_layout(xaxis_tickangle=-45)
    return fig

def create_color_palette_chart(df):
    """
    Visualize color distribution with actual colors
    """
    fig = px.bar(df,
                 x='color_hex',
                 y='artifact_count',
                 title='Most Common Colors in Collection',
                 color='color_hex',
                 color_discrete_map={row['color_hex']: row['color_hex'] 
                                     for _, row in df.iterrows()})
    return fig

# Usage in Streamlit
st.plotly_chart(create_culture_chart(results_df), use_container_width=True)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Workflows

### 1. Initial Data Collection

```python
# Collect first 500 artifacts (5 pages × 100 records)
run_etl_pipeline(num_pages=5, records_per_page=100)
```

### 2. Incremental Updates

```python
def incremental_load(start_page, end_page):
    """
    Load additional pages without duplicating existing data
    """
    for page in range(start_page, end_page + 1):
        artifacts, _ = fetch_artifacts(
            os.getenv('HARVARD_API_KEY'), 
            page=page
        )
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
```

### 3. Custom Analytics Query

```python
def run_custom_query(sql_query):
    """
    Execute user-provided SQL query
    """
    conn = get_db_connection()
    try:
        df = pd.read_sql(sql_query, conn)
        return df
    except Exception as e:
        st.error(f"Query error: {e}")
        return None
    finally:
        conn.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """
    Fetch with exponential backoff on rate limits
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if "429" in str(e):  # Rate limit error
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """
    Verify database connectivity
    """
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        return result[0] == 1
    except Exception as e:
        print(f"Connection failed: {e}")
        return False
```

### Missing Data Handling

```python
def clean_artifact_data(artifact):
    """
    Handle missing or null fields gracefully
    """
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Untitled'),
        'culture': artifact.get('culture', 'Unknown'),
        'century': artifact.get('century', 'Unknown'),
        # Use .get() with defaults for all fields
    }
```

## Advanced Usage

### Parallel Data Loading

```python
from concurrent.futures import ThreadPoolExecutor

def parallel_etl(num_pages, workers=4):
    """
    Fetch multiple pages concurrently
    """
    with ThreadPoolExecutor(max_workers=workers) as executor:
        futures = [
            executor.submit(fetch_artifacts, os.getenv('HARVARD_API_KEY'), page)
            for page in range(1, num_pages + 1)
        ]
        
        results = [f.result() for f in futures]
    
    # Process all results
    all_artifacts = [artifact for artifacts, _ in results for artifact in artifacts]
    return transform_artifacts(all_artifacts)
```

### Data Quality Checks

```python
def validate_data_quality():
    """
    Run data quality checks on loaded data
    """
    conn = get_db_connection()
    
    checks = {
        "Total Artifacts": "SELECT COUNT(*) FROM artifactmetadata",
        "Artifacts Without Images": """
            SELECT COUNT(*) FROM artifactmedia WHERE has_images = 0
        """,
        "Missing Cultures": """
            SELECT COUNT(*) FROM artifactmetadata WHERE culture IS NULL
        """
    }
    
    results = {}
    for check_name, query in checks.items():
        df = pd.read_sql(query, conn)
        results[check_name] = df.iloc[0, 0]
    
    conn.close()
    return results
```

This skill provides everything needed to build, run, and extend the Harvard Artifacts ETL Analytics application.
