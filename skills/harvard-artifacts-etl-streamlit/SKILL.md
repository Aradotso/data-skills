---
name: harvard-artifacts-etl-streamlit
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - connect to Harvard Art Museums API
  - create a data engineering app with Streamlit
  - set up artifact collection analytics
  - analyze museum artifacts with SQL
  - build a cultural heritage data pipeline
  - visualize museum collection data
  - create an art museum analytics dashboard
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build and work with an end-to-end data engineering application that collects artifact data from the Harvard Art Museums API, performs ETL operations, stores data in SQL databases, and creates interactive analytics dashboards using Streamlit.

## What This Project Does

The Harvard Artifacts Collection app demonstrates a complete data pipeline:

1. **Extract**: Fetches artifact data from Harvard Art Museums API with pagination
2. **Transform**: Processes nested JSON into relational database schema
3. **Load**: Batch inserts into MySQL/TiDB Cloud with foreign key relationships
4. **Analyze**: Runs 20+ predefined SQL analytical queries
5. **Visualize**: Displays results in interactive Streamlit dashboards with Plotly charts

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Install core packages if requirements.txt not available
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Getting Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to `.env` file

## Database Schema

The project uses three main tables with relationships:

```sql
-- Main artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    classification VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    description TEXT,
    provenance TEXT,
    commentary TEXT,
    url VARCHAR(500),
    total_unique_pages INT,
    total_page_views INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Media/images associated with artifacts
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url VARCHAR(500),
    image_copyright TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Color analysis for artifacts
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    color_percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Core ETL Pipeline Pattern

### 1. Extract Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

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
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=50)

print(f"Total records: {info['totalrecords']}")
print(f"Total pages: {info['pages']}")
```

### 2. Transform Nested JSON to Relational Format

```python
import pandas as pd

def transform_artifacts(artifacts_json):
    """
    Transform nested JSON into separate DataFrames for each table
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts_json:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'classification': artifact.get('classification', '')[:200],
            'period': artifact.get('period', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'description': artifact.get('description'),
            'provenance': artifact.get('provenance'),
            'commentary': artifact.get('commentary'),
            'url': artifact.get('url', '')[:500],
            'total_unique_pages': artifact.get('totaluniquepageviews', 0),
            'total_page_views': artifact.get('totalpageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'base_image_url': artifact.get('primaryimageurl', '')[:500],
                'image_copyright': artifact.get('imagepermissionlevel')
            }
            media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_name': color.get('color'),
                    'color_percentage': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Usage
df_metadata, df_media, df_colors = transform_artifacts(artifacts)
```

### 3. Load Data into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def batch_insert_metadata(df_metadata):
    """Batch insert artifact metadata"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, century, department, division, dated, 
         classification, period, technique, medium, description, 
         provenance, commentary, url, total_unique_pages, total_page_views)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    # Convert DataFrame to list of tuples
    data = df_metadata.to_records(index=False).tolist()
    
    cursor.executemany(insert_query, data)
    conn.commit()
    
    rows_inserted = cursor.rowcount
    cursor.close()
    conn.close()
    
    return rows_inserted

def batch_insert_media(df_media):
    """Batch insert media records"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, base_image_url, image_copyright)
        VALUES (%s, %s, %s)
    """
    
    data = df_media.to_records(index=False).tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    rows_inserted = cursor.rowcount
    cursor.close()
    conn.close()
    
    return rows_inserted
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_top_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10;
"""

# Query 2: Artifacts with images by department
query_dept_images = """
    SELECT am.department, COUNT(DISTINCT am.id) as artifact_count
    FROM artifactmetadata am
    INNER JOIN artifactmedia img ON am.id = img.artifact_id
    GROUP BY am.department
    ORDER BY artifact_count DESC;
"""

# Query 3: Most common colors across artifacts
query_top_colors = """
    SELECT color_name, COUNT(*) as usage_count,
           AVG(color_percentage) as avg_percentage
    FROM artifactcolors
    WHERE color_name IS NOT NULL
    GROUP BY color_name
    ORDER BY usage_count DESC
    LIMIT 15;
"""

# Query 4: Artifacts by century
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY count DESC;
"""

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard Pattern

```python
import streamlit as st
import plotly.express as px

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums - Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")

queries = {
    "Top Cultures": query_top_cultures,
    "Artifacts by Department": query_dept_images,
    "Most Common Colors": query_top_colors,
    "Artifacts by Century": query_by_century
}

selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute and display results
if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        df_result = execute_query(queries[selected_query])
        
        # Display table
        st.subheader("Query Results")
        st.dataframe(df_result, use_container_width=True)
        
        # Auto-generate visualization
        if len(df_result) > 0:
            st.subheader("Visualization")
            
            x_col = df_result.columns[0]
            y_col = df_result.columns[1]
            
            fig = px.bar(
                df_result,
                x=x_col,
                y=y_col,
                title=f"{selected_query} Analysis",
                color=y_col,
                color_continuous_scale='Viridis'
            )
            
            st.plotly_chart(fig, use_container_width=True)
```

## Complete ETL Workflow Example

```python
import streamlit as st
import time

def run_full_etl_pipeline(num_pages=5):
    """
    Complete ETL pipeline workflow
    """
    api_key = os.getenv('HARVARD_API_KEY')
    
    progress_bar = st.progress(0)
    status_text = st.empty()
    
    all_metadata = []
    all_media = []
    all_colors = []
    
    # Extract
    status_text.text("Extracting data from API...")
    for page in range(1, num_pages + 1):
        artifacts, info = fetch_artifacts(api_key, page=page, size=100)
        
        # Transform
        df_meta, df_med, df_col = transform_artifacts(artifacts)
        
        all_metadata.append(df_meta)
        all_media.append(df_med)
        all_colors.append(df_col)
        
        progress = page / num_pages
        progress_bar.progress(progress)
        time.sleep(0.5)  # Rate limiting
    
    # Combine all pages
    final_metadata = pd.concat(all_metadata, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load
    status_text.text("Loading data into database...")
    
    rows_meta = batch_insert_metadata(final_metadata)
    rows_media = batch_insert_media(final_media)
    rows_colors = batch_insert_colors(final_colors)
    
    progress_bar.progress(1.0)
    status_text.text("ETL Complete!")
    
    return {
        'metadata_rows': rows_meta,
        'media_rows': rows_media,
        'colors_rows': rows_colors
    }

# Streamlit UI for ETL
st.header("Data Collection")

num_pages = st.slider("Number of pages to collect", 1, 20, 5)

if st.button("Start ETL Pipeline"):
    results = run_full_etl_pipeline(num_pages)
    st.success(f"""
        ETL Pipeline Completed!
        - Metadata: {results['metadata_rows']} rows
        - Media: {results['media_rows']} rows
        - Colors: {results['colors_rows']} rows
    """)
```

## Common Patterns

### Error Handling for API Calls

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            time.sleep(wait_time)
```

### Data Validation

```python
def validate_artifact_data(df):
    """Validate transformed data before loading"""
    # Check for required fields
    required_cols = ['id', 'title']
    missing = [col for col in required_cols if col not in df.columns]
    
    if missing:
        raise ValueError(f"Missing required columns: {missing}")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        df = df.drop_duplicates(subset=['id'])
    
    # Truncate long text fields
    text_fields = ['title', 'culture', 'department']
    for field in text_fields:
        if field in df.columns:
            df[field] = df[field].astype(str).str[:500]
    
    return df
```

## Troubleshooting

### API Rate Limiting
If you hit rate limits, add delays between requests:
```python
time.sleep(1)  # 1 second between API calls
```

### Database Connection Issues
Verify connection parameters:
```python
try:
    conn = get_db_connection()
    conn.close()
    print("Database connection successful")
except Error as e:
    print(f"Database error: {e}")
```

### Missing Data Fields
Handle None/null values during transformation:
```python
artifact.get('field_name', 'default_value')
```

### Streamlit Caching
Use caching for expensive operations:
```python
@st.cache_data(ttl=3600)
def load_analytics_data():
    return execute_query(query)
```

This skill provides comprehensive guidance for building end-to-end data engineering pipelines with museum API data, SQL analytics, and interactive visualizations.
