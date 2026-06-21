---
name: harvard-art-museums-data-engineering
description: Build end-to-end data pipelines with Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit dashboards
triggers:
  - build a data pipeline for museum artifacts
  - create ETL workflow with Harvard Art Museums API
  - set up analytics dashboard for art collection data
  - analyze museum artifacts with SQL queries
  - build Streamlit app for art museum data
  - create data engineering pipeline for cultural heritage data
  - extract and visualize Harvard Art Museums data
  - design relational database for artifact metadata
---

# Harvard Art Museums Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build production-ready data engineering pipelines using the Harvard Art Museums API. The project demonstrates real-world ETL patterns, relational database design, SQL analytics, and interactive visualization with Streamlit.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **Database Design**: Creates properly structured MySQL/TiDB tables with foreign keys
- **SQL Analytics**: Executes 20+ analytical queries for insights
- **Visualization**: Builds interactive Streamlit dashboards with Plotly charts

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

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to your `.env` file

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)
```

## Project Structure

```
├── app.py                 # Main Streamlit application
├── etl_pipeline.py        # ETL logic for data extraction/transformation
├── database.py            # Database connection and operations
├── queries.py             # SQL analytics queries
├── requirements.txt       # Python dependencies
└── .env                   # Environment configuration
```

## Core Components

### 1. API Data Extraction

```python
import requests
import pandas as pd
from typing import List, Dict
import os

def fetch_artifacts(api_key: str, size: int = 100, page: int = 1) -> Dict:
    """Fetch artifacts from Harvard Art Museums API"""
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

def collect_artifacts(api_key: str, total_records: int = 500) -> List[Dict]:
    """Collect multiple pages of artifact data"""
    artifacts = []
    page = 1
    page_size = 100
    
    while len(artifacts) < total_records:
        data = fetch_artifacts(api_key, size=page_size, page=page)
        records = data.get('records', [])
        
        if not records:
            break
            
        artifacts.extend(records)
        page += 1
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return artifacts[:total_records]
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts: List[Dict]) -> pd.DataFrame:
    """Transform raw API data into metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract media/image information"""
    media_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'objectid': objectid,
                'imageid': img.get('imageid'),
                'baseimageurl': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            })
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts: List[Dict]) -> pd.DataFrame:
    """Extract color information"""
    color_records = []
    
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'objectid': objectid,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

### 3. Database Schema Creation

```python
def create_database_schema(conn):
    """Create normalized database tables"""
    cursor = conn.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            period VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            medium TEXT,
            dated VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            creditline TEXT,
            accessionyear INT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            imageid INT,
            baseimageurl TEXT,
            width INT,
            height INT,
            format VARCHAR(50),
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
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    conn.commit()
    cursor.close()
```

### 4. Batch Data Loading

```python
def load_dataframe_to_table(df: pd.DataFrame, table_name: str, conn):
    """Batch insert DataFrame into SQL table"""
    if df.empty:
        return
    
    cursor = conn.cursor()
    
    # Prepare INSERT statement
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    data = [tuple(row) for row in df.values]
    
    # Execute batch insert
    cursor.executemany(query, data)
    conn.commit()
    cursor.close()
    
    print(f"Loaded {len(df)} records into {table_name}")
```

### 5. Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Top Cultures": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Color Spectrum Analysis": """
        SELECT spectrum, COUNT(*) as occurrence, 
               AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE spectrum IS NOT NULL
        GROUP BY spectrum
        ORDER BY occurrence DESC
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as image_count,
               AVG(width) as avg_width, AVG(height) as avg_height
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY image_count DESC
    """,
    
    "Artifacts with Multiple Images": """
        SELECT a.title, a.culture, COUNT(m.imageid) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.objectid = m.objectid
        GROUP BY a.objectid, a.title, a.culture
        HAVING image_count > 3
        ORDER BY image_count DESC
        LIMIT 20
    """
}

def execute_query(conn, query: str) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    return pd.read_sql(query, conn)
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")

st.title("🎨 Harvard Art Museums - Data Analytics Dashboard")

# Database connection
@st.cache_resource
def get_database_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

conn = get_database_connection()

# Sidebar for query selection
st.sidebar.header("Analytics Queries")
query_name = st.sidebar.selectbox(
    "Select Analysis",
    list(ANALYTICS_QUERIES.keys())
)

# Execute selected query
if st.sidebar.button("Run Query"):
    with st.spinner("Executing query..."):
        query = ANALYTICS_QUERIES[query_name]
        results = execute_query(conn, query)
        
        st.subheader(f"Results: {query_name}")
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) >= 2:
            fig = px.bar(
                results,
                x=results.columns[0],
                y=results.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

# ETL Pipeline Section
st.sidebar.header("ETL Pipeline")
if st.sidebar.button("Run ETL"):
    api_key = os.getenv('HARVARD_API_KEY')
    
    with st.spinner("Fetching artifacts..."):
        artifacts = collect_artifacts(api_key, total_records=100)
        st.success(f"Fetched {len(artifacts)} artifacts")
    
    with st.spinner("Transforming data..."):
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
        st.success("Data transformation complete")
    
    with st.spinner("Loading to database..."):
        load_dataframe_to_table(df_metadata, 'artifactmetadata', conn)
        load_dataframe_to_table(df_media, 'artifactmedia', conn)
        load_dataframe_to_table(df_colors, 'artifactcolors', conn)
        st.success("Data loaded successfully!")
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Collection

```python
def get_latest_objectid(conn) -> int:
    """Get the highest objectid already in database"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] or 0

def incremental_etl(api_key: str, conn):
    """Only fetch new artifacts"""
    latest_id = get_latest_objectid(conn)
    
    # Fetch with filter
    url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': 100,
        'q': f'objectid:>{latest_id}'
    }
    
    response = requests.get(url, params=params)
    new_artifacts = response.json().get('records', [])
    
    # Transform and load only new records
    if new_artifacts:
        df = transform_artifact_metadata(new_artifacts)
        load_dataframe_to_table(df, 'artifactmetadata', conn)
```

### Pattern 2: Error Handling for API Calls

```python
import time
from requests.exceptions import RequestException

def robust_fetch_artifacts(api_key: str, max_retries: int = 3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            response = requests.get(
                "https://api.harvardartmuseums.org/object",
                params={'apikey': api_key, 'size': 100},
                timeout=10
            )
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # Exponential backoff
                time.sleep(wait_time)
            else:
                raise Exception(f"Failed after {max_retries} attempts: {e}")
```

### Pattern 3: Data Quality Validation

```python
def validate_artifacts(df: pd.DataFrame) -> pd.DataFrame:
    """Clean and validate artifact data"""
    # Remove duplicates
    df = df.drop_duplicates(subset=['objectid'])
    
    # Fill missing values
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Undated')
    
    # Data type conversions
    df['accessionyear'] = pd.to_numeric(df['accessionyear'], errors='coerce')
    
    # Filter invalid records
    df = df[df['objectid'].notna()]
    
    return df
```

## Troubleshooting

### Issue: API Rate Limiting

```python
# Add delays between requests
import time

for page in range(1, 10):
    data = fetch_artifacts(api_key, page=page)
    time.sleep(1)  # 1 second delay
```

### Issue: Database Connection Timeout

```python
# Use connection pooling
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

conn = db_pool.get_connection()
```

### Issue: Large Dataset Memory Errors

```python
# Process in chunks
def chunked_etl(api_key: str, conn, chunk_size: int = 100):
    """Process data in manageable chunks"""
    page = 1
    
    while True:
        artifacts = fetch_artifacts(api_key, size=chunk_size, page=page)
        
        if not artifacts:
            break
        
        # Transform and load chunk
        df = transform_artifact_metadata(artifacts)
        load_dataframe_to_table(df, 'artifactmetadata', conn)
        
        page += 1
```

### Issue: NULL Values in Foreign Keys

```python
# Ensure parent records exist before inserting children
def safe_load_media(df_media: pd.DataFrame, conn):
    """Only load media for existing artifacts"""
    cursor = conn.cursor()
    cursor.execute("SELECT objectid FROM artifactmetadata")
    valid_ids = {row[0] for row in cursor.fetchall()}
    cursor.close()
    
    # Filter media records
    df_filtered = df_media[df_media['objectid'].isin(valid_ids)]
    load_dataframe_to_table(df_filtered, 'artifactmedia', conn)
```

## Key Commands

```bash
# Run full ETL pipeline
python -c "from etl_pipeline import run_full_etl; run_full_etl()"

# Launch dashboard
streamlit run app.py

# Run specific query
python -c "from queries import execute_query; execute_query('SELECT * FROM artifactmetadata LIMIT 10')"
```

This skill provides everything needed to build production-grade data pipelines for cultural heritage data using modern Python data engineering tools.
