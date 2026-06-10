---
name: harvard-art-museum-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data pipeline for museum artifact data
  - connect to Harvard Art Museums API and store in database
  - build analytics dashboard for art museum collection data
  - extract and analyze Harvard museum artifacts with SQL
  - set up Streamlit app for museum data visualization
  - query Harvard Art Museums API and load into MySQL
  - create data engineering project with museum API
---

# Harvard Art Museum ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum artifact data into relational databases
- **SQL Storage**: Store structured data in MySQL/TiDB Cloud with proper schema design
- **Analytics Queries**: 20+ predefined SQL queries for artifact analysis
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies**:
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
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to environment variables

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

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

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    provenance TEXT,
    description TEXT,
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    mediatype VARCHAR(100),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
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
        data = response.json()
        
        return {
            'records': data.get('records', []),
            'total_pages': data.get('info', {}).get('pages', 0),
            'total_records': data.get('info', {}).get('totalrecords', 0)
        }
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None
```

### Batch Data Collection

```python
import time

def collect_artifacts_batch(num_pages=5, delay=1):
    """
    Collect multiple pages of artifacts with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        result = fetch_artifacts(page=page, size=100)
        
        if result and result['records']:
            all_artifacts.extend(result['records'])
            print(f"Collected {len(result['records'])} artifacts")
        
        # Rate limiting - be respectful to the API
        time.sleep(delay)
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """
    Transform raw API response into structured dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'provenance': artifact.get('provenance'),
            'description': artifact.get('description'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media
        for media in artifact.get('images', []):
            media_record = {
                'artifact_id': artifact.get('id'),
                'mediatype': media.get('mediatype'),
                'baseimageurl': media.get('baseimageurl'),
                'iiifbaseuri': media.get('iiifbaseuri'),
                'publiccaption': media.get('publiccaption')
            }
            media_list.append(media_record)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            colors_list.append(color_record)
    
    return {
        'metadata': pd.DataFrame(metadata_list),
        'media': pd.DataFrame(media_list),
        'colors': pd.DataFrame(colors_list)
    }
```

### Load to Database

```python
def load_to_database(dataframes, conn):
    """
    Load transformed dataframes into MySQL database
    """
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in dataframes['metadata'].iterrows():
        sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, 
         dated, period, technique, medium, dimensions, provenance, description, 
         url, totalpageviews, totaluniquepageviews)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE 
        title=VALUES(title), culture=VALUES(culture)
        """
        cursor.execute(sql, tuple(row))
    
    # Load media
    for _, row in dataframes['media'].iterrows():
        sql = """
        INSERT INTO artifactmedia 
        (artifact_id, mediatype, baseimageurl, iiifbaseuri, publiccaption)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    # Load colors
    for _, row in dataframes['colors'].iterrows():
        sql = """
        INSERT INTO artifactcolors 
        (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.execute(sql, tuple(row))
    
    conn.commit()
    cursor.close()
    print("Data loaded successfully!")
```

## Analytics Query Examples

### Top Cultures by Artifact Count

```python
def query_top_cultures(conn, limit=10):
    """
    Query top cultures by number of artifacts
    """
    query = f"""
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT {limit}
    """
    return pd.read_sql(query, conn)
```

### Artifacts by Century

```python
def query_artifacts_by_century(conn):
    """
    Analyze artifact distribution across centuries
    """
    query = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
    """
    return pd.read_sql(query, conn)
```

### Most Popular Colors

```python
def query_popular_colors(conn, limit=10):
    """
    Find most common colors in the collection
    """
    query = f"""
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT {limit}
    """
    return pd.read_sql(query, conn)
```

### Department Statistics

```python
def query_department_stats(conn):
    """
    Get comprehensive statistics by department
    """
    query = """
    SELECT 
        department,
        COUNT(*) as total_artifacts,
        SUM(totalpageviews) as total_views,
        AVG(totalpageviews) as avg_views
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY total_artifacts DESC
    """
    return pd.read_sql(query, conn)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(
    page_title="Harvard Art Museum Analytics",
    page_icon="🎨",
    layout="wide"
)

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar navigation
page = st.sidebar.selectbox(
    "Choose a section",
    ["Data Collection", "Analytics", "Visualizations"]
)

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

conn = get_db_connection()
```

### Data Collection Page

```python
if page == "Data Collection":
    st.header("📥 Collect Artifact Data")
    
    num_pages = st.number_input("Number of pages to fetch", 
                                 min_value=1, max_value=20, value=5)
    
    if st.button("Start Collection"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_artifacts_batch(num_pages=num_pages)
            st.success(f"Collected {len(artifacts)} artifacts")
            
            # Transform and load
            dataframes = transform_artifacts(artifacts)
            load_to_database(dataframes, conn)
            st.success("Data loaded to database!")
```

### Analytics Page with Visualizations

```python
elif page == "Analytics":
    st.header("📊 SQL Analytics")
    
    query_options = {
        "Top 10 Cultures": query_top_cultures,
        "Artifacts by Century": query_artifacts_by_century,
        "Popular Colors": query_popular_colors,
        "Department Statistics": query_department_stats
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        df = query_options[selected_query](conn)
        
        # Display data
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(num_pages=5):
    """
    ETL pipeline with comprehensive error handling
    """
    try:
        # Extract
        artifacts = collect_artifacts_batch(num_pages)
        if not artifacts:
            raise ValueError("No artifacts collected")
        
        # Transform
        dataframes = transform_artifacts(artifacts)
        
        # Validate
        if dataframes['metadata'].empty:
            raise ValueError("No metadata to load")
        
        # Load
        conn = get_db_connection()
        load_to_database(dataframes, conn)
        conn.close()
        
        return True
    except Exception as e:
        print(f"ETL Pipeline Error: {e}")
        return False
```

### Incremental Data Loading

```python
def get_last_artifact_id(conn):
    """
    Get the last loaded artifact ID to support incremental loads
    """
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load(conn):
    """
    Load only new artifacts since last run
    """
    last_id = get_last_artifact_id(conn)
    # Fetch artifacts with ID > last_id
    # Implementation depends on API filtering capabilities
```

## Troubleshooting

### API Rate Limiting

```python
# Add exponential backoff
import time

def fetch_with_retry(page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

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

def get_pooled_connection():
    return db_pool.get_connection()
```

### Memory Management for Large Datasets

```python
def batch_load_large_dataset(artifacts, batch_size=100):
    """
    Load large datasets in batches to avoid memory issues
    """
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        dataframes = transform_artifacts(batch)
        load_to_database(dataframes, conn)
        print(f"Loaded batch {i//batch_size + 1}")
```

This skill enables AI agents to help developers build complete data engineering solutions using the Harvard Art Museums API, from ETL pipelines to interactive analytics dashboards.
