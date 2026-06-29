---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - create an ETL pipeline for Harvard Art Museums data
  - build a Streamlit analytics dashboard for museum artifacts
  - extract and transform Harvard Art Museums API data
  - set up SQL analytics for art collection data
  - implement artifact data pipeline with visualization
  - query and analyze Harvard Art Museums collections
  - create museum artifact database with Python
  - build interactive art data dashboard
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for visualizing museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into SQL databases
- **Database Design**: Structured relational tables (artifactmetadata, artifactmedia, artifactcolors)
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Set up environment variables
echo "HARVARD_API_KEY=your_api_key_here" > .env
echo "DB_HOST=your_database_host" >> .env
echo "DB_USER=your_database_user" >> .env
echo "DB_PASSWORD=your_database_password" >> .env
echo "DB_NAME=harvard_artifacts" >> .env
```

### Get Harvard API Key

1. Visit https://harvardartmuseums.org/collections/api
2. Request an API key
3. Store in `.env` file as `HARVARD_API_KEY`

## Key Commands

### Run the Streamlit Application

```bash
streamlit run app.py
```

### Direct Python Usage

```python
# Run ETL pipeline programmatically
python etl_pipeline.py

# Execute specific analytics queries
python analytics_queries.py
```

## Configuration

### Environment Variables

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

### Database Connection

```python
import mysql.connector

def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Core API Usage Patterns

### Fetching Artifacts from Harvard API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
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
    
    data = response.json()
    return data['records'], data['info']

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=50)
print(f"Total artifacts: {info['totalrecords']}")
print(f"Fetched: {len(artifacts)}")
```

### Paginated Data Collection

```python
def collect_all_artifacts(api_key, max_records=500):
    """
    Collect artifacts with pagination handling
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        try:
            artifacts, info = fetch_artifacts(api_key, page=page, size=size)
            all_artifacts.extend(artifacts)
            
            if len(artifacts) < size:
                break
                
            page += 1
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts[:max_records]
```

## ETL Pipeline Implementation

### Extract: API Data Extraction

```python
def extract_artifact_data(artifact):
    """
    Extract relevant fields from API response
    """
    return {
        'objectid': artifact.get('objectid'),
        'title': artifact.get('title'),
        'culture': artifact.get('culture'),
        'period': artifact.get('period'),
        'century': artifact.get('century'),
        'classification': artifact.get('classification'),
        'department': artifact.get('department'),
        'division': artifact.get('division'),
        'dated': artifact.get('dated'),
        'url': artifact.get('url'),
        'primaryimageurl': artifact.get('primaryimageurl')
    }
```

### Transform: Data Cleaning and Structuring

```python
def transform_artifacts_to_dataframe(artifacts):
    """
    Transform raw API data into structured DataFrame
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Metadata
        metadata_list.append(extract_artifact_data(artifact))
        
        # Media/Images
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media_list.append({
                    'objectid': artifact.get('objectid'),
                    'baseimageurl': image.get('baseimageurl'),
                    'iiifbaseuri': image.get('iiifbaseuri'),
                    'imageid': image.get('imageid')
                })
        
        # Colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                colors_list.append({
                    'objectid': artifact.get('objectid'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into SQL Database

```python
def create_tables(connection):
    """
    Create database schema
    """
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            dated VARCHAR(200),
            url VARCHAR(500),
            primaryimageurl VARCHAR(500)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            imageid INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent DECIMAL(5,2),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()

def load_data_to_sql(df_metadata, df_media, df_colors, connection):
    """
    Batch insert DataFrames into SQL tables
    """
    cursor = connection.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (objectid, title, culture, period, century, classification, 
             department, division, dated, url, primaryimageurl)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (objectid, baseimageurl, iiifbaseuri, imageid)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
```

## Analytics SQL Queries

### Common Analytical Queries

```python
# Query 1: Artifacts by Culture
QUERY_ARTIFACTS_BY_CULTURE = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts by Century
QUERY_ARTIFACTS_BY_CENTURY = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Top Colors Used
QUERY_TOP_COLORS = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 10
"""

# Query 4: Artifacts with Most Images
QUERY_ARTIFACTS_MOST_IMAGES = """
    SELECT m.objectid, m.title, COUNT(med.id) as image_count
    FROM artifactmetadata m
    JOIN artifactmedia med ON m.objectid = med.objectid
    GROUP BY m.objectid, m.title
    ORDER BY image_count DESC
    LIMIT 10
"""

# Query 5: Department Distribution
QUERY_DEPARTMENT_DISTRIBUTION = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""
```

### Executing Queries

```python
def execute_query(connection, query):
    """
    Execute SQL query and return results as DataFrame
    """
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    return pd.DataFrame(results)

# Usage
conn = get_db_connection()
df_results = execute_query(conn, QUERY_ARTIFACTS_BY_CULTURE)
print(df_results)
```

## Streamlit Dashboard Implementation

### Basic Dashboard Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("API Key", type="password", 
                                      value=os.getenv('HARVARD_API_KEY', ''))
    
    # Tabs for different sections
    tab1, tab2, tab3 = st.tabs(["Data Collection", "SQL Analytics", "Visualizations"])
    
    with tab1:
        show_data_collection_tab(api_key)
    
    with tab2:
        show_sql_analytics_tab()
    
    with tab3:
        show_visualizations_tab()

if __name__ == "__main__":
    main()
```

### Data Collection Tab

```python
def show_data_collection_tab(api_key):
    st.header("📥 Data Collection from API")
    
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_all_artifacts(api_key, max_records=num_records)
            
            df_meta, df_media, df_colors = transform_artifacts_to_dataframe(artifacts)
            
            st.success(f"Fetched {len(artifacts)} artifacts")
            st.write(f"Metadata rows: {len(df_meta)}")
            st.write(f"Media rows: {len(df_media)}")
            st.write(f"Colors rows: {len(df_colors)}")
            
            # Preview data
            st.subheader("Metadata Preview")
            st.dataframe(df_meta.head())
            
            # Load to database
            if st.button("Load to Database"):
                conn = get_db_connection()
                create_tables(conn)
                load_data_to_sql(df_meta, df_media, df_colors, conn)
                conn.close()
                st.success("Data loaded successfully!")
```

### SQL Analytics Tab

```python
def show_sql_analytics_tab():
    st.header("📊 SQL Analytics Dashboard")
    
    queries = {
        "Artifacts by Culture": QUERY_ARTIFACTS_BY_CULTURE,
        "Artifacts by Century": QUERY_ARTIFACTS_BY_CENTURY,
        "Top Colors Used": QUERY_TOP_COLORS,
        "Artifacts with Most Images": QUERY_ARTIFACTS_MOST_IMAGES,
        "Department Distribution": QUERY_DEPARTMENT_DISTRIBUTION
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df_result = execute_query(conn, queries[selected_query])
        conn.close()
        
        st.subheader("Query Results")
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) >= 2:
            fig = px.bar(df_result, 
                         x=df_result.columns[0], 
                         y=df_result.columns[1],
                         title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

### Visualization Tab

```python
def show_visualizations_tab():
    st.header("📈 Interactive Visualizations")
    
    conn = get_db_connection()
    
    # Culture Distribution
    df_culture = execute_query(conn, QUERY_ARTIFACTS_BY_CULTURE)
    fig1 = px.pie(df_culture, values='artifact_count', names='culture',
                  title='Artifact Distribution by Culture')
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color Analysis
    df_colors = execute_query(conn, QUERY_TOP_COLORS)
    fig2 = px.bar(df_colors, x='color', y='usage_count',
                  title='Most Common Colors in Collection',
                  color='avg_percent')
    st.plotly_chart(fig2, use_container_width=True)
    
    conn.close()
```

## Common Patterns

### Pattern 1: Complete ETL Workflow

```python
def run_complete_etl(api_key, num_records=500):
    """
    End-to-end ETL pipeline execution
    """
    # Extract
    print("Extracting data from API...")
    artifacts = collect_all_artifacts(api_key, max_records=num_records)
    
    # Transform
    print("Transforming data...")
    df_meta, df_media, df_colors = transform_artifacts_to_dataframe(artifacts)
    
    # Load
    print("Loading data to database...")
    conn = get_db_connection()
    create_tables(conn)
    load_data_to_sql(df_meta, df_media, df_colors, conn)
    conn.close()
    
    print(f"ETL complete! Loaded {len(artifacts)} artifacts.")
```

### Pattern 2: Scheduled Data Refresh

```python
import schedule
import time

def scheduled_etl_job():
    api_key = os.getenv('HARVARD_API_KEY')
    run_complete_etl(api_key, num_records=100)

# Run every 24 hours
schedule.every(24).hours.do(scheduled_etl_job)

while True:
    schedule.run_pending()
    time.sleep(3600)
```

### Pattern 3: Incremental Loading

```python
def get_latest_objectid(connection):
    """
    Get the most recent objectid from database
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(api_key):
    """
    Load only new artifacts since last update
    """
    conn = get_db_connection()
    latest_id = get_latest_objectid(conn)
    
    # Fetch new artifacts with objectid > latest_id
    # Implementation depends on API filtering capabilities
    
    conn.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, max_retries=3):
    """
    Fetch with exponential backoff on rate limits
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except HTTPError as e:
            if e.response.status_code == 429:  # Too Many Requests
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def get_db_connection_with_retry(max_attempts=3):
    """
    Establish database connection with retry logic
    """
    for attempt in range(max_attempts):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connect_timeout=10
            )
            return conn
        except mysql.connector.Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_attempts - 1:
                time.sleep(2)
            else:
                raise
```

### Handling Missing Data

```python
def safe_extract(artifact, field, default=None):
    """
    Safely extract field with default value
    """
    return artifact.get(field, default)

def clean_dataframe(df):
    """
    Clean and validate DataFrame before loading
    """
    # Replace None with empty string
    df = df.fillna('')
    
    # Trim long strings
    for col in df.select_dtypes(include=['object']).columns:
        df[col] = df[col].astype(str).str[:500]
    
    return df
```

### Streamlit Performance Optimization

```python
import streamlit as st

@st.cache_data(ttl=3600)
def cached_query(query_text):
    """
    Cache query results for 1 hour
    """
    conn = get_db_connection()
    result = execute_query(conn, query_text)
    conn.close()
    return result

@st.cache_resource
def get_cached_connection():
    """
    Cache database connection
    """
    return get_db_connection()
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement pagination** for large datasets to avoid memory issues
3. **Use batch inserts** for efficient database loading
4. **Cache expensive operations** in Streamlit with `@st.cache_data`
5. **Add error handling** for API requests and database operations
6. **Validate data** before insertion to prevent SQL errors
7. **Close database connections** properly to avoid leaks
8. **Use connection pooling** for production deployments

This skill provides complete coverage for building ETL pipelines and analytics dashboards with the Harvard Art Museums API using Python, SQL, and Streamlit.
