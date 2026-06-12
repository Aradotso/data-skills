---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with museum artifact data
  - set up Harvard API data engineering workflow
  - implement artifact collection data pipeline
  - build museum data visualization with Streamlit
  - query and analyze Harvard Art Museums collections
  - create ETL process for art museum metadata
  - develop artifact analytics application
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Art Museums ETL Analytics App is a complete data pipeline that:
- Fetches artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and color data
- Visualizes insights through interactive Streamlit dashboards with Plotly charts

**Architecture**: API → ETL (Python) → SQL Database → Analytics Queries → Streamlit Visualization

## Installation

### Prerequisites
```bash
python 3.8+
pip
MySQL or TiDB Cloud account
Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)
```

### Setup Steps
```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Configure environment variables (create .env file)
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_database_host
# DB_USER=your_database_user
# DB_PASSWORD=your_database_password
# DB_NAME=harvard_artifacts
```

### Requirements
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Database Schema

### Core Tables

**artifactmetadata**
```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    description TEXT,
    url VARCHAR(500)
);
```

**artifactmedia**
```sql
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    image_width INT,
    image_height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

**artifactcolors**
```sql
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'fields': 'id,title,culture,century,classification,department,dated,medium,description,url,images,colors'
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

def fetch_all_artifacts(max_records=1000):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    import time
    
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=size)
        
        records = data.get('records', [])
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting: 1 second between requests
        time.sleep(1)
        
        if len(all_artifacts) >= max_records:
            all_artifacts = all_artifacts[:max_records]
            break
    
    return all_artifacts
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform artifacts into normalized dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'description': artifact.get('description'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for image in images:
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl'),
                'image_width': image.get('width'),
                'image_height': image.get('height')
            }
            media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_rec = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            }
            color_records.append(color_rec)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df_metadata):
    """
    Load metadata into database with batch insert
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, dated, medium, description, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    records = df_metadata.to_records(index=False)
    data = [tuple(row) for row in records]
    
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return len(data)

def load_media(df_media):
    """
    Load media records
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, image_width, image_height)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df_media.to_records(index=False)
    data = [tuple(row) for row in records]
    
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return len(data)

def load_colors(df_colors):
    """
    Load color records
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
    VALUES (%s, %s, %s)
    """
    
    records = df_colors.to_records(index=False)
    data = [tuple(row) for row in records]
    
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return len(data)
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Top 10 cultures by artifact count
QUERY_TOP_CULTURES = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century distribution
QUERY_CENTURY_DISTRIBUTION = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Department-wise artifact classification
QUERY_DEPT_CLASSIFICATION = """
SELECT department, classification, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC;
"""

# Query 4: Artifacts with images vs without
QUERY_IMAGE_AVAILABILITY = """
SELECT 
    CASE 
        WHEN media_count > 0 THEN 'With Images'
        ELSE 'Without Images'
    END as image_status,
    COUNT(*) as artifact_count
FROM (
    SELECT m.id, COUNT(am.media_id) as media_count
    FROM artifactmetadata m
    LEFT JOIN artifactmedia am ON m.id = am.artifact_id
    GROUP BY m.id
) as subquery
GROUP BY image_status;
"""

# Query 5: Top 10 most common colors in artifacts
QUERY_TOP_COLORS = """
SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
FROM artifactcolors
WHERE color_hex IS NOT NULL
GROUP BY color_hex
ORDER BY usage_count DESC
LIMIT 10;
"""

# Query 6: Artifacts by medium
QUERY_BY_MEDIUM = """
SELECT medium, COUNT(*) as count
FROM artifactmetadata
WHERE medium IS NOT NULL
GROUP BY medium
ORDER BY count DESC
LIMIT 15;
"""

def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard Implementation

### Main Application

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    """
    ETL pipeline interface
    """
    st.header("📥 Data Collection & ETL Pipeline")
    
    num_records = st.number_input(
        "Number of records to fetch",
        min_value=10,
        max_value=5000,
        value=100
    )
    
    if st.button("Start ETL Process"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_all_artifacts(max_records=num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading into database..."):
            meta_count = load_metadata(df_metadata)
            media_count = load_media(df_media)
            color_count = load_colors(df_colors)
            
            st.success(f"""
            Loaded successfully:
            - {meta_count} metadata records
            - {media_count} media records
            - {color_count} color records
            """)

def show_sql_analytics():
    """
    SQL query execution interface
    """
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top Cultures": QUERY_TOP_CULTURES,
        "Century Distribution": QUERY_CENTURY_DISTRIBUTION,
        "Department Classification": QUERY_DEPT_CLASSIFICATION,
        "Image Availability": QUERY_IMAGE_AVAILABILITY,
        "Top Colors": QUERY_TOP_COLORS,
        "Medium Distribution": QUERY_BY_MEDIUM
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        query = queries[selected_query]
        
        with st.spinner("Executing query..."):
            df = execute_query(query)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)

def show_visualizations():
    """
    Pre-built visualizations
    """
    st.header("📈 Data Visualizations")
    
    # Culture distribution
    st.subheader("Artifact Distribution by Culture")
    df_cultures = execute_query(QUERY_TOP_CULTURES)
    fig1 = px.bar(df_cultures, x='culture', y='artifact_count', 
                  title="Top 10 Cultures")
    st.plotly_chart(fig1, use_container_width=True)
    
    # Color analysis
    st.subheader("Color Usage in Artifacts")
    df_colors = execute_query(QUERY_TOP_COLORS)
    fig2 = px.scatter(df_colors, x='usage_count', y='avg_percent',
                      color='color_hex', size='usage_count',
                      title="Color Distribution")
    st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

### Running the Dashboard

```bash
# Start Streamlit application
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(num_records=1000):
    """
    Execute full ETL pipeline
    """
    print("Starting ETL process...")
    
    # Extract
    print("1. Extracting data from API...")
    artifacts = fetch_all_artifacts(max_records=num_records)
    print(f"   Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("2. Transforming data...")
    df_metadata, df_media, df_colors = transform_artifacts(artifacts)
    print(f"   Transformed into {len(df_metadata)} metadata, "
          f"{len(df_media)} media, {len(df_colors)} color records")
    
    # Load
    print("3. Loading into database...")
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    print("   Load complete!")
    
    return df_metadata, df_media, df_colors
```

### Custom Query Execution

```python
def run_custom_analysis(custom_query):
    """
    Execute custom SQL query with visualization
    """
    df = execute_query(custom_query)
    
    # Automatic chart selection based on data
    if len(df.columns) == 2 and df[df.columns[1]].dtype in ['int64', 'float64']:
        fig = px.bar(df, x=df.columns[0], y=df.columns[1])
    else:
        fig = px.scatter(df)
    
    return df, fig
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for API requests
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retry():
    session = requests.Session()
    retry = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Large Dataset Memory Issues
```python
# Use chunked processing for large datasets
def load_data_chunked(df, table_name, chunk_size=1000):
    for i in range(0, len(df), chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        load_chunk(chunk, table_name)
        print(f"Loaded chunk {i//chunk_size + 1}")
```

### Missing API Key
```python
# Validate environment setup
def validate_config():
    required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
    missing = [var for var in required_vars if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing environment variables: {', '.join(missing)}")
    
    return True
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement rate limiting** when fetching from APIs (1 request/second recommended)
3. **Use batch inserts** for database operations to improve performance
4. **Handle NULL values** in SQL queries with proper WHERE clauses
5. **Create indexes** on frequently queried columns (culture, century, department)
6. **Log ETL operations** for debugging and monitoring
7. **Validate data** before loading into database
8. **Use transactions** for data integrity in multi-table inserts
