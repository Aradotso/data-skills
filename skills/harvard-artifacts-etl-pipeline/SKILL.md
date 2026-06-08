---
name: harvard-artifacts-etl-pipeline
description: Build ETL pipelines and analytics apps using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline with Harvard artifacts data
  - create analytics dashboard with Streamlit and museum data
  - structure Harvard API data into SQL tables
  - visualize museum collection data with Plotly
  - set up data engineering pipeline for art museum collections
  - query and analyze Harvard Art Museums artifacts
  - transform nested JSON art data into relational database
---

# Harvard Artifacts ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates end-to-end data engineering and analytics using the Harvard Art Museums API. It extracts artifact data, transforms it into structured formats, loads it into SQL databases, and visualizes insights through interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetches artifact metadata, media, and color data from Harvard Art Museums API
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational SQL tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Plotly-based charts and Streamlit dashboards

Architecture: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

**Dependencies** (typical `requirements.txt`):
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Store credentials in `.env`:
```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Connection

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

def get_db_connection():
    """Establish database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        page: Page number (pagination)
        size: Number of records per page (max 100)
    
    Returns:
        dict: API response with artifact data
    """
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Fetch first 100 artifacts
data = fetch_artifacts(page=1, size=100)
artifacts = data['records']
total_records = data['info']['totalrecords']
print(f"Total artifacts available: {total_records}")
```

### 2. ETL Pipeline

```python
import pandas as pd

def extract_artifact_metadata(artifacts):
    """Extract metadata from artifact records"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_artifact_media(artifacts):
    """Extract media/image data from artifacts"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format'),
                'copyright': img.get('copyright')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_artifact_colors(artifacts):
    """Extract color data from artifacts"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent'),
                'css3': color.get('css3')
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### 3. Load Data into SQL

```python
def create_tables(connection):
    """Create database tables with proper schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            artifact_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            technique TEXT,
            medium TEXT,
            dimensions TEXT,
            creditline TEXT,
            url VARCHAR(512)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_id INT,
            base_url VARCHAR(512),
            width INT,
            height INT,
            format VARCHAR(50),
            copyright TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            css3 VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_dataframe_to_sql(df, table_name, connection):
    """Batch insert DataFrame into SQL table"""
    cursor = connection.cursor()
    
    # Prepare insert query
    columns = ','.join(df.columns)
    placeholders = ','.join(['%s'] * len(df.columns))
    query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data = [tuple(row) for row in df.values]
    cursor.executemany(query, data)
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(df)} records into {table_name}")

# Complete ETL workflow
connection = get_db_connection()
create_tables(connection)

artifacts = fetch_artifacts(page=1, size=100)['records']

df_metadata = extract_artifact_metadata(artifacts)
df_media = extract_artifact_media(artifacts)
df_colors = extract_artifact_colors(artifacts)

load_dataframe_to_sql(df_metadata, 'artifactmetadata', connection)
load_dataframe_to_sql(df_media, 'artifactmedia', connection)
load_dataframe_to_sql(df_colors, 'artifactcolors', connection)

connection.close()
```

### 4. SQL Analytics Queries

```python
def run_analytics_query(query, connection):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)

# Example queries
queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Media Availability": """
        SELECT 
            CASE 
                WHEN EXISTS (SELECT 1 FROM artifactmedia WHERE artifact_id = am.artifact_id)
                THEN 'Has Images'
                ELSE 'No Images'
            END as media_status,
            COUNT(*) as count
        FROM artifactmetadata am
        GROUP BY media_status
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

# Execute query
connection = get_db_connection()
df_result = run_analytics_query(queries["Artifacts by Culture"], connection)
print(df_result)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
query_name = st.sidebar.selectbox(
    "Select Analytics Query",
    list(queries.keys())
)

# Execute selected query
if st.button("Run Query"):
    with st.spinner("Executing query..."):
        connection = get_db_connection()
        df = run_analytics_query(queries[query_name], connection)
        connection.close()
        
        # Display results
        st.subheader(f"Results: {query_name}")
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(
                df, 
                x=df.columns[0], 
                y=df.columns[1],
                title=query_name,
                labels={df.columns[0]: df.columns[0].title(), 
                       df.columns[1]: df.columns[1].title()}
            )
            st.plotly_chart(fig, use_container_width=True)

# Data collection section
st.sidebar.header("📥 Data Collection")
num_pages = st.sidebar.number_input("Pages to fetch", min_value=1, max_value=10, value=1)

if st.sidebar.button("Collect Data"):
    progress_bar = st.sidebar.progress(0)
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page, size=100)
        artifacts = data['records']
        
        # Process and load
        connection = get_db_connection()
        df_meta = extract_artifact_metadata(artifacts)
        df_media = extract_artifact_media(artifacts)
        df_colors = extract_artifact_colors(artifacts)
        
        load_dataframe_to_sql(df_meta, 'artifactmetadata', connection)
        load_dataframe_to_sql(df_media, 'artifactmedia', connection)
        load_dataframe_to_sql(df_colors, 'artifactcolors', connection)
        
        connection.close()
        progress_bar.progress(page / num_pages)
    
    st.sidebar.success(f"Collected {num_pages * 100} artifacts!")
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Paginated Data Collection

```python
def collect_all_artifacts(max_pages=10):
    """Collect artifacts with pagination and rate limiting"""
    import time
    
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(page=page, size=100)
            all_artifacts.extend(data['records'])
            
            # Rate limiting
            time.sleep(0.5)
            
            print(f"Fetched page {page}/{max_pages}")
            
        except requests.exceptions.RequestException as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(artifacts):
    """ETL with error handling"""
    try:
        df_metadata = extract_artifact_metadata(artifacts)
        df_media = extract_artifact_media(artifacts)
        df_colors = extract_artifact_colors(artifacts)
        
        connection = get_db_connection()
        
        load_dataframe_to_sql(df_metadata, 'artifactmetadata', connection)
        load_dataframe_to_sql(df_media, 'artifactmedia', connection)
        load_dataframe_to_sql(df_colors, 'artifactcolors', connection)
        
        connection.close()
        return True
        
    except Exception as e:
        print(f"ETL Pipeline Error: {e}")
        if 'connection' in locals():
            connection.rollback()
            connection.close()
        return False
```

## Troubleshooting

**API Rate Limiting**
- Harvard API limits requests; add delays between calls
- Use `time.sleep(0.5)` between pagination requests

**Database Connection Issues**
- Verify credentials in `.env` file
- Check if MySQL/TiDB service is running
- Ensure database exists: `CREATE DATABASE harvard_artifacts;`

**Foreign Key Constraints**
- Load `artifactmetadata` table first (parent table)
- Then load `artifactmedia` and `artifactcolors` (child tables)

**Missing Data in Queries**
- Some artifacts may have NULL values for culture, period, etc.
- Use `WHERE column IS NOT NULL` to filter

**Streamlit Performance**
- Cache database connections with `@st.cache_resource`
- Cache query results with `@st.cache_data`

```python
@st.cache_resource
def get_cached_connection():
    return get_db_connection()

@st.cache_data(ttl=3600)
def cached_query(query):
    conn = get_cached_connection()
    return pd.read_sql(query, conn)
```

This skill enables AI agents to help developers build complete data engineering pipelines using museum API data, from extraction through visualization.
