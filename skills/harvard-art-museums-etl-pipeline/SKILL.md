---
name: harvard-art-museums-etl-pipeline
description: Build end-to-end ETL pipelines for Harvard Art Museums data with SQL analytics and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums
  - create a data engineering app with Harvard artifacts
  - fetch and analyze Harvard Art Museums API data
  - set up SQL analytics for museum artifact data
  - build a Streamlit dashboard for art museum collections
  - extract Harvard artifacts data into a database
  - create a museum data pipeline with visualization
  - analyze art museum collection metadata with SQL
---

# Harvard Art Museums ETL Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering applications using the Harvard Art Museums API. The project demonstrates ETL (Extract, Transform, Load) pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data engineering solution that:

- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** collection data using 20+ predefined SQL queries
- **Visualizes** insights through interactive Plotly charts in Streamlit

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api))

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
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

## Database Schema

The project uses three normalized tables:

### artifactmetadata
```sql
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url TEXT
);
```

### artifactmedia
```sql
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    image_width INT,
    image_height INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

### artifactcolors
```sql
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Extract artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size
    }
    
    response = requests.get(base_url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

def collect_all_artifacts(api_key, max_records=1000):
    """Paginate through API to collect multiple records"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get("records", [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
    return all_artifacts[:max_records]
```

### Transform: JSON to Relational Format

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact metadata into DataFrame"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            "artifact_id": artifact.get("id"),
            "title": artifact.get("title"),
            "culture": artifact.get("culture"),
            "century": artifact.get("century"),
            "classification": artifact.get("classification"),
            "department": artifact.get("department"),
            "dated": artifact.get("dated"),
            "accession_number": artifact.get("accessionNumber"),
            "url": artifact.get("url")
        })
    
    return pd.DataFrame(metadata_list)

def transform_media(artifacts):
    """Transform nested media data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        images = artifact.get("images", [])
        
        for image in images:
            media_list.append({
                "artifact_id": artifact_id,
                "image_url": image.get("baseimageurl"),
                "image_width": image.get("width"),
                "image_height": image.get("height")
            })
    
    return pd.DataFrame(media_list)

def transform_colors(artifacts):
    """Transform color data from artifacts"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        colors = artifact.get("colors", [])
        
        for color in colors:
            color_list.append({
                "artifact_id": artifact_id,
                "color_hex": color.get("hex"),
                "color_name": color.get("color"),
                "percentage": color.get("percent")
            })
    
    return pd.DataFrame(color_list)
```

### Load: Batch Insert into SQL

```python
import mysql.connector
from mysql.connector import Error

def create_connection(host, user, password, database):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Error: {e}")
        return None

def load_metadata(connection, df_metadata):
    """Load metadata into SQL with batch insert"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (artifact_id, title, culture, century, classification, department, dated, accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE 
    title=VALUES(title), culture=VALUES(culture)
    """
    
    records = df_metadata.to_records(index=False)
    data = [tuple(record) for record in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

def load_media(connection, df_media):
    """Load media data into SQL"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, image_width, image_height)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df_media.to_records(index=False)
    data = [tuple(record) for record in records]
    
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()
```

## Streamlit Application Structure

```python
import streamlit as st
import plotly.express as px
import os

# Page configuration
st.set_page_config(
    page_title="Harvard Art Museums Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums Data Engineering & Analytics")

# Sidebar for configuration
with st.sidebar:
    st.header("Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv("HARVARD_API_KEY", ""))
    db_host = st.text_input("Database Host", value=os.getenv("DB_HOST", ""))
    db_user = st.text_input("Database User", value=os.getenv("DB_USER", ""))
    db_password = st.text_input("Database Password", type="password",
                               value=os.getenv("DB_PASSWORD", ""))
    db_name = st.text_input("Database Name", value=os.getenv("DB_NAME", ""))

# ETL Section
st.header("📥 ETL Pipeline")

col1, col2, col3 = st.columns(3)

with col1:
    num_records = st.number_input("Records to Fetch", min_value=10, max_value=10000, value=500)

with col2:
    if st.button("Run ETL Pipeline"):
        with st.spinner("Extracting data from API..."):
            artifacts = collect_all_artifacts(api_key, max_records=num_records)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_metadata(artifacts)
            df_media = transform_media(artifacts)
            df_colors = transform_colors(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading into database..."):
            conn = create_connection(db_host, db_user, db_password, db_name)
            load_metadata(conn, df_metadata)
            load_media(conn, df_media)
            load_colors(conn, df_colors)
            conn.close()
            st.success("Data loaded into SQL database")
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Artifacts by Culture
QUERY_ARTIFACTS_BY_CULTURE = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 20
"""

# Query 2: Department Distribution
QUERY_DEPARTMENT_DISTRIBUTION = """
SELECT department, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY count DESC
"""

# Query 3: Artifacts with Images
QUERY_ARTIFACTS_WITH_IMAGES = """
SELECT 
    am.classification,
    COUNT(DISTINCT am.artifact_id) as total_artifacts,
    COUNT(DISTINCT media.artifact_id) as artifacts_with_images,
    ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.artifact_id), 2) as percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.artifact_id = media.artifact_id
WHERE am.classification IS NOT NULL
GROUP BY am.classification
ORDER BY total_artifacts DESC
LIMIT 15
"""

# Query 4: Top Colors Across Collections
QUERY_TOP_COLORS = """
SELECT 
    color_name,
    COUNT(*) as occurrence_count,
    AVG(percentage) as avg_percentage
FROM artifactcolors
WHERE color_name IS NOT NULL
GROUP BY color_name
ORDER BY occurrence_count DESC
LIMIT 10
"""

# Query 5: Century Analysis
QUERY_CENTURY_ANALYSIS = """
SELECT 
    century,
    COUNT(*) as artifact_count,
    COUNT(DISTINCT culture) as unique_cultures
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY artifact_count DESC
LIMIT 15
"""

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

### Visualization Integration

```python
def create_visualization(df, title, x_col, y_col):
    """Create interactive Plotly chart"""
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        labels={x_col: x_col.replace('_', ' ').title(), 
                y_col: y_col.replace('_', ' ').title()},
        color=y_col,
        color_continuous_scale='Viridis'
    )
    
    fig.update_layout(
        xaxis_tickangle=-45,
        height=500
    )
    
    return fig

# In Streamlit app
st.header("📊 Analytics Dashboard")

query_options = {
    "Artifacts by Culture": QUERY_ARTIFACTS_BY_CULTURE,
    "Department Distribution": QUERY_DEPARTMENT_DISTRIBUTION,
    "Artifacts with Images": QUERY_ARTIFACTS_WITH_IMAGES,
    "Top Colors": QUERY_TOP_COLORS,
    "Century Analysis": QUERY_CENTURY_ANALYSIS
}

selected_query = st.selectbox("Select Analysis", list(query_options.keys()))

if st.button("Run Analysis"):
    conn = create_connection(db_host, db_user, db_password, db_name)
    df_result = execute_query(conn, query_options[selected_query])
    conn.close()
    
    st.dataframe(df_result, use_container_width=True)
    
    if len(df_result.columns) >= 2:
        fig = create_visualization(
            df_result,
            selected_query,
            df_result.columns[0],
            df_result.columns[1]
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, page, delay=0.5):
    """Fetch data with rate limiting to avoid API throttling"""
    data = fetch_artifacts(api_key, page=page)
    time.sleep(delay)  # Wait between requests
    return data
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, db_config, max_records=1000):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        artifacts = collect_all_artifacts(api_key, max_records)
        
        # Transform
        df_metadata = transform_metadata(artifacts)
        df_media = transform_media(artifacts)
        df_colors = transform_colors(artifacts)
        
        # Load
        conn = create_connection(**db_config)
        if conn is None:
            raise Exception("Database connection failed")
        
        load_metadata(conn, df_metadata)
        load_media(conn, df_media)
        load_colors(conn, df_colors)
        
        conn.close()
        return True, "ETL completed successfully"
        
    except Exception as e:
        return False, f"ETL failed: {str(e)}"
```

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the last loaded artifact ID"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load(api_key, connection, batch_size=100):
    """Load only new artifacts since last run"""
    last_id = get_last_artifact_id(connection)
    
    # Fetch artifacts with ID > last_id
    # Implementation depends on API filtering capabilities
    pass
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key):
    try:
        response = requests.get(
            f"https://api.harvardartmuseums.org/object?apikey={api_key}&size=1"
        )
        if response.status_code == 200:
            print("✅ API connection successful")
            return True
        else:
            print(f"❌ API returned status code: {response.status_code}")
            return False
    except Exception as e:
        print(f"❌ Connection error: {e}")
        return False
```

### Database Connection Issues

```python
# Test database connectivity
def test_db_connection(host, user, password, database):
    try:
        conn = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database,
            connect_timeout=10
        )
        if conn.is_connected():
            print("✅ Database connection successful")
            conn.close()
            return True
    except Error as e:
        print(f"❌ Database error: {e}")
        return False
```

### Handle Missing Data

```python
def clean_dataframe(df):
    """Clean DataFrame before SQL insert"""
    # Replace None with empty strings for VARCHAR columns
    df = df.fillna('')
    
    # Truncate long strings to fit column limits
    if 'title' in df.columns:
        df['title'] = df['title'].str[:500]
    
    return df
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="root"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Key Configuration

All sensitive configuration should use environment variables:

```python
import os
from dotenv import load_dotenv

load_dotenv()

CONFIG = {
    "api_key": os.getenv("HARVARD_API_KEY"),
    "db_host": os.getenv("DB_HOST"),
    "db_user": os.getenv("DB_USER"),
    "db_password": os.getenv("DB_PASSWORD"),
    "db_name": os.getenv("DB_NAME")
}
```

This skill provides complete coverage for building ETL pipelines with the Harvard Art Museums API, from data extraction through SQL analytics and interactive visualization.
