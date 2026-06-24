---
name: harvard-artifacts-etl-analytics
description: Build end-to-end data engineering pipelines for Harvard Art Museums data with ETL, SQL analytics, and Streamlit visualization
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - create a data analytics app with Harvard artifacts
  - set up SQL database for museum artifact data
  - visualize Harvard Art Museums collection data
  - build a Streamlit dashboard for art collection analytics
  - extract and transform Harvard museum API data
  - create analytics queries for artifact metadata
  - design database schema for museum collections
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection app enables:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Structured storage with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts

**Architecture Flow**: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from [Harvard API Portal](https://www.harvardartmuseums.org/collections/api))

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
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

## Configuration

### Database Connection

Create a `.env` file for secure configuration:

```bash
HARVARD_API_KEY=your_api_key
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema

```python
import mysql.connector
from mysql.connector import Error
import os
from dotenv import load_dotenv

load_dotenv()

def create_database_connection():
    """Establish MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT', 3306),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def create_tables(connection):
    """Create normalized database schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            objectnumber VARCHAR(255),
            title TEXT,
            culture VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            century VARCHAR(255),
            dated VARCHAR(255),
            medium TEXT,
            dimensions TEXT,
            creditline TEXT,
            url TEXT,
            INDEX idx_culture (culture),
            INDEX idx_classification (classification),
            INDEX idx_century (century)
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl TEXT,
            primaryimageurl TEXT,
            has_image BOOLEAN,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
            INDEX idx_artifact (artifact_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id),
            INDEX idx_artifact (artifact_id),
            INDEX idx_color (color)
        )
    """)
    
    connection.commit()
    cursor.close()
```

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import time
import os

def fetch_artifacts(api_key, num_pages=5, page_size=100):
    """
    Fetch artifact data from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        num_pages: Number of pages to fetch
        page_size: Records per page (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        try:
            response = requests.get(base_url, params=params)
            response.raise_for_status()
            data = response.json()
            
            artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{num_pages} - {len(data.get('records', []))} records")
            
            # Rate limiting - be respectful to API
            time.sleep(1)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return artifacts
```

### Transform: Process Nested JSON

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """
    Transform nested JSON into normalized dataframes
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'objectnumber': artifact.get('objectnumber'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media info
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'has_image': 1 if artifact.get('primaryimageurl') else 0
        }
        media_records.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    metadata_df = pd.DataFrame(metadata_records)
    media_df = pd.DataFrame(media_records)
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### Load: Batch Insert to SQL

```python
def load_to_database(connection, metadata_df, media_df, colors_df):
    """Batch load dataframes to SQL database"""
    cursor = connection.cursor()
    
    # Load metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, objectnumber, title, culture, classification, department, 
         century, dated, medium, dimensions, creditline, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Load media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, primaryimageurl, has_image)
        VALUES (%s, %s, %s, %s)
    """
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Load colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

## SQL Analytics Queries

### Example Analytical Queries

```python
def get_analytics_queries():
    """Collection of predefined analytical queries"""
    return {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 20
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
        
        "Image Availability": """
            SELECT has_image, COUNT(*) as count
            FROM artifactmedia
            GROUP BY has_image
        """,
        
        "Top Colors Used": """
            SELECT color, COUNT(*) as frequency, 
                   AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """,
        
        "Artifacts by Classification": """
            SELECT classification, COUNT(*) as count,
                   GROUP_CONCAT(DISTINCT culture SEPARATOR ', ') as cultures
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 20
        """,
        
        "Color Spectrum Analysis": """
            SELECT spectrum, color, COUNT(*) as usage_count
            FROM artifactcolors
            GROUP BY spectrum, color
            ORDER BY spectrum, usage_count DESC
        """
    }

def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, connection)
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("*End-to-end Data Engineering & Analytics Pipeline*")
    
    # Sidebar for ETL operations
    st.sidebar.header("ETL Operations")
    
    if st.sidebar.button("🔄 Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            artifacts = fetch_artifacts(api_key, num_pages=5)
            st.sidebar.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.sidebar.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            conn = create_database_connection()
            create_tables(conn)
            load_to_database(conn, metadata_df, media_df, colors_df)
            conn.close()
            st.sidebar.success("✅ ETL Complete!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    conn = create_database_connection()
    if conn:
        queries = get_analytics_queries()
        
        selected_query = st.selectbox(
            "Select Analysis",
            list(queries.keys())
        )
        
        if st.button("Run Query"):
            df = execute_query(conn, queries[selected_query])
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(20),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)
        
        conn.close()

if __name__ == "__main__":
    main()
```

### Run the Dashboard

```bash
streamlit run app.py
```

## Common Patterns

### Full ETL Workflow

```python
from dotenv import load_dotenv
import os

load_dotenv()

# Step 1: Extract
api_key = os.getenv('HARVARD_API_KEY')
raw_artifacts = fetch_artifacts(api_key, num_pages=10, page_size=100)

# Step 2: Transform
metadata_df, media_df, colors_df = transform_artifacts(raw_artifacts)

# Step 3: Load
connection = create_database_connection()
create_tables(connection)
load_to_database(connection, metadata_df, media_df, colors_df)

# Step 4: Analyze
result = execute_query(connection, get_analytics_queries()["Artifacts by Culture"])
print(result)

connection.close()
```

### Incremental Data Loading

```python
def incremental_load(connection, last_id=0):
    """Load only new artifacts since last_id"""
    api_key = os.getenv('HARVARD_API_KEY')
    
    # Fetch artifacts with ID filter
    params = {
        'apikey': api_key,
        'q': f'id:>{last_id}',
        'size': 100
    }
    
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params=params
    )
    new_artifacts = response.json().get('records', [])
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(connection, metadata_df, media_df, colors_df)
        return max(artifact['id'] for artifact in new_artifacts)
    
    return last_id
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_minute=60):
    """Decorator to enforce rate limiting"""
    min_interval = 60.0 / calls_per_minute
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_minute=50)
def fetch_page(page_num, api_key):
    # API call code here
    pass
```

### Database Connection Pooling

```python
from mysql.connector import pooling

def create_connection_pool():
    """Create connection pool for better performance"""
    return pooling.MySQLConnectionPool(
        pool_name="harvard_pool",
        pool_size=5,
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

pool = create_connection_pool()
connection = pool.get_connection()
```

### Handling Missing Data

```python
def safe_transform(artifact):
    """Transform with null handling"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Unknown'),
        'culture': artifact.get('culture', 'Unknown'),
        'century': artifact.get('century', 'Unknown'),
        # Use .get() with defaults for all fields
    }
```

### Memory Optimization for Large Datasets

```python
def batch_process(artifacts, batch_size=1000):
    """Process artifacts in batches to avoid memory issues"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i + batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        
        conn = create_database_connection()
        load_to_database(conn, metadata_df, media_df, colors_df)
        conn.close()
        
        print(f"Processed batch {i//batch_size + 1}")
```
