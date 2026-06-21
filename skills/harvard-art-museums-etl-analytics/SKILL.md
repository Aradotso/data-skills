---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifact collections
  - extract and transform Harvard museum API data
  - set up Harvard Art Museums data engineering project
  - analyze museum artifacts with SQL queries
  - visualize museum collection data with Streamlit
  - implement batch data ingestion from Harvard API
  - query Harvard artifacts database with analytics
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build complete data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical querying, and interactive visualization using Streamlit. It's ideal for learning real-world data pipeline patterns with cultural heritage data.

## What This Project Does

The Harvard Artifacts Collection application provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract nested JSON, transform to relational structure, load into SQL databases
- **Database Design**: Structured tables for artifact metadata, media, and color information
- **Analytics Engine**: 20+ predefined SQL queries for collection insights
- **Visualization Dashboard**: Interactive Streamlit interface with Plotly charts

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud database
- Harvard Art Museums API key (get from [Harvard API portal](https://www.harvardartmuseums.org/collections/api))

### Setup Steps

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_db_username"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Database Configuration

Create the required database and tables:

```python
import pymysql
import os

# Database connection
connection = pymysql.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME'),
    cursorclass=pymysql.cursors.DictCursor
)

# Create tables
with connection.cursor() as cursor:
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dated VARCHAR(200),
            period VARCHAR(200),
            provenance TEXT,
            description TEXT,
            url VARCHAR(500)
        )
    """)
    
    # Artifact media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            image_url VARCHAR(500),
            base_image_url VARCHAR(500),
            renditionnumber VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
connection.commit()
connection.close()
```

## Key Components

### 1. API Data Extraction

Fetch artifact data with pagination:

```python
import requests
import os
import time

def fetch_artifacts(api_key, page=1, size=100, max_pages=10):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page_num in range(1, max_pages + 1):
        params = {
            'apikey': api_key,
            'page': page_num,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            print(f"Fetched page {page_num}: {len(artifacts)} artifacts")
            
            # Respect rate limits
            time.sleep(0.5)
        else:
            print(f"Error on page {page_num}: {response.status_code}")
            break
    
    return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, max_pages=5)
print(f"Total artifacts collected: {len(artifacts)}")
```

### 2. ETL Transformation

Transform nested JSON to relational structure:

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform raw API data into structured DataFrames
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'technique': artifact.get('technique', ''),
            'medium': artifact.get('medium', ''),
            'dated': artifact.get('dated', ''),
            'period': artifact.get('period', ''),
            'provenance': artifact.get('provenance', ''),
            'description': artifact.get('description', ''),
            'url': artifact.get('url', '')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': img.get('baseimageurl', ''),
                'base_image_url': img.get('iiifbaseuri', ''),
                'renditionnumber': img.get('renditionnumber', '')
            }
            media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            color_records.append(color_data)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

# Usage
metadata_df, media_df, colors_df = transform_artifacts(artifacts)
print(f"Metadata: {len(metadata_df)} rows")
print(f"Media: {len(media_df)} rows")
print(f"Colors: {len(colors_df)} rows")
```

### 3. Data Loading

Batch insert data into SQL database:

```python
from sqlalchemy import create_engine
import os

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load DataFrames into SQL database using batch inserts
    """
    # Create database engine
    db_url = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
    engine = create_engine(db_url)
    
    # Load metadata
    metadata_df.to_sql(
        'artifactmetadata', 
        engine, 
        if_exists='append', 
        index=False,
        method='multi',
        chunksize=500
    )
    print(f"Loaded {len(metadata_df)} metadata records")
    
    # Load media
    if not media_df.empty:
        media_df.to_sql(
            'artifactmedia', 
            engine, 
            if_exists='append', 
            index=False,
            method='multi',
            chunksize=500
        )
        print(f"Loaded {len(media_df)} media records")
    
    # Load colors
    if not colors_df.empty:
        colors_df.to_sql(
            'artifactcolors', 
            engine, 
            if_exists='append', 
            index=False,
            method='multi',
            chunksize=500
        )
        print(f"Loaded {len(colors_df)} color records")

# Usage
load_to_database(metadata_df, media_df, colors_df)
```

### 4. Analytics Queries

Execute analytical SQL queries:

```python
import pandas as pd
from sqlalchemy import create_engine

def execute_analytics_query(query_name):
    """
    Execute predefined analytics queries
    """
    db_url = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
    engine = create_engine(db_url)
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'top_classifications': """
            SELECT classification, COUNT(*) as total
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY total DESC
            LIMIT 15
        """,
        
        'artifacts_with_images': """
            SELECT 
                CASE WHEN m.artifact_id IS NOT NULL THEN 'With Images' ELSE 'No Images' END as status,
                COUNT(DISTINCT a.id) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY status
        """,
        
        'color_distribution': """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """,
        
        'department_statistics': """
            SELECT 
                department,
                COUNT(*) as total_artifacts,
                COUNT(DISTINCT classification) as unique_classifications,
                COUNT(DISTINCT culture) as unique_cultures
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
        """
    }
    
    if query_name in queries:
        df = pd.read_sql(queries[query_name], engine)
        return df
    else:
        raise ValueError(f"Query '{query_name}' not found")

# Usage
culture_data = execute_analytics_query('artifacts_by_culture')
print(culture_data.head(10))
```

### 5. Streamlit Dashboard

Build interactive visualization dashboard:

```python
import streamlit as st
import plotly.express as px
import pandas as pd
from sqlalchemy import create_engine
import os

def create_dashboard():
    """
    Create Streamlit analytics dashboard
    """
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    st.title("🎨 Harvard Art Museums Collection Analytics")
    
    # Database connection
    db_url = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
    engine = create_engine(db_url)
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_options = [
        "Artifacts by Culture",
        "Artifacts by Century",
        "Top Classifications",
        "Department Statistics",
        "Color Distribution",
        "Media Availability"
    ]
    selected_query = st.sidebar.selectbox("Select Analysis", query_options)
    
    # Execute selected query
    if selected_query == "Artifacts by Culture":
        query = """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """
        df = pd.read_sql(query, engine)
        
        st.subheader("Artifact Distribution by Culture")
        st.dataframe(df)
        
        fig = px.bar(df, x='culture', y='count', 
                     title='Top 20 Cultures by Artifact Count',
                     labels={'count': 'Number of Artifacts', 'culture': 'Culture'})
        fig.update_layout(xaxis_tickangle=-45)
        st.plotly_chart(fig, use_container_width=True)
    
    elif selected_query == "Color Distribution":
        query = """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_coverage
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """
        df = pd.read_sql(query, engine)
        
        st.subheader("Color Usage in Artifacts")
        col1, col2 = st.columns(2)
        
        with col1:
            st.dataframe(df)
        
        with col2:
            fig = px.pie(df, values='frequency', names='color',
                        title='Color Distribution')
            st.plotly_chart(fig)
    
    # Summary statistics
    st.sidebar.markdown("---")
    st.sidebar.subheader("Collection Summary")
    
    total_artifacts = pd.read_sql("SELECT COUNT(*) as count FROM artifactmetadata", engine)
    total_images = pd.read_sql("SELECT COUNT(*) as count FROM artifactmedia", engine)
    
    st.sidebar.metric("Total Artifacts", total_artifacts['count'][0])
    st.sidebar.metric("Total Images", total_images['count'][0])

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

### Run ETL Pipeline

```bash
# Execute full ETL pipeline
python etl_pipeline.py --pages 10 --batch-size 100

# With custom parameters
python etl_pipeline.py --pages 20 --batch-size 50 --api-key $HARVARD_API_KEY
```

### Launch Streamlit Dashboard

```bash
streamlit run app.py

# Custom port
streamlit run app.py --server.port 8501
```

## Common Patterns

### Incremental Data Loading

```python
def incremental_load(last_update_time):
    """
    Load only new artifacts since last update
    """
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'updatedafter': last_update_time,
        'hasimage': 1
    }
    
    response = requests.get(
        'https://api.harvardartmuseums.org/object',
        params=params
    )
    
    return response.json().get('records', [])
```

### Error Handling in ETL

```python
def safe_etl_pipeline():
    """
    ETL with comprehensive error handling
    """
    try:
        # Extract
        artifacts = fetch_artifacts(os.getenv('HARVARD_API_KEY'))
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate data
        assert not metadata_df.empty, "No metadata extracted"
        
        # Load
        load_to_database(metadata_df, media_df, colors_df)
        
        print("ETL pipeline completed successfully")
        return True
        
    except requests.RequestException as e:
        print(f"API request failed: {e}")
        return False
    except Exception as e:
        print(f"ETL pipeline error: {e}")
        return False
```

### Query Result Caching

```python
import streamlit as st

@st.cache_data(ttl=3600)
def cached_query(query_string):
    """
    Cache query results for 1 hour
    """
    db_url = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
    engine = create_engine(db_url)
    return pd.read_sql(query_string, engine)
```

## Troubleshooting

### API Rate Limiting

If you encounter 429 errors:

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    session = requests.Session()
    retry = Retry(
        total=5,
        backoff_factor=1,
        status_forcelist=[429, 500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session
```

### Database Connection Issues

```python
from sqlalchemy import create_engine
from sqlalchemy.exc import OperationalError

def test_db_connection():
    try:
        db_url = f"mysql+pymysql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}@{os.getenv('DB_HOST')}/{os.getenv('DB_NAME')}"
        engine = create_engine(db_url)
        connection = engine.connect()
        connection.close()
        print("✓ Database connection successful")
        return True
    except OperationalError as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets

```python
def chunked_etl(total_pages, chunk_size=5):
    """
    Process data in chunks to avoid memory issues
    """
    for start_page in range(1, total_pages + 1, chunk_size):
        end_page = min(start_page + chunk_size - 1, total_pages)
        
        artifacts = fetch_artifacts(
            os.getenv('HARVARD_API_KEY'),
            page=start_page,
            max_pages=chunk_size
        )
        
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        
        print(f"Processed pages {start_page}-{end_page}")
        time.sleep(2)  # Respect rate limits
```

## Configuration Reference

### Environment Variables

```bash
# Required
HARVARD_API_KEY=your_api_key          # Get from Harvard API portal
DB_HOST=localhost                      # Database host
DB_USER=root                           # Database username
DB_PASSWORD=your_password              # Database password
DB_NAME=harvard_artifacts              # Database name

# Optional
API_RATE_LIMIT=1.0                     # Seconds between requests
BATCH_SIZE=100                         # Records per API request
MAX_PAGES=10                           # Maximum pages to fetch
```

This skill enables complete ETL pipeline development for museum collection data with production-ready patterns for API integration, data transformation, and analytics visualization.
