---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to create analytics dashboards with museum data
  - help me set up a data engineering project with Streamlit
  - how to extract and transform Harvard artifacts data
  - build a SQL analytics pipeline for art museum collections
  - create visualizations from Harvard Art Museums API
  - set up ETL for cultural heritage data
  - help me query and visualize museum artifact metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries on museum collections
- Visualizes results through interactive Plotly charts in Streamlit

## Architecture Flow

```
Harvard Art Museums API → ETL Pipeline → SQL Database → Analytics Queries → Streamlit Dashboard
```

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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Getting Harvard API Key

1. Register at https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Add to `.env` file

### Database Setup

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Extraction

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
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

def collect_all_artifacts(max_pages=10):
    """
    Collect multiple pages of artifacts with rate limiting
    """
    import time
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(0.5)  # Rate limiting
        else:
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured DataFrames
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
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        if artifact.get('images'):
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl'),
                    'media_type': 'image'
                }
                media_list.append(media)
        
        # Extract color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                }
                colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed data into MySQL database
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            query = """
            INSERT IGNORE INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            query = """
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            query = """
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database Error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. Analytical SQL Queries

```python
# Sample analytical queries for insights

ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Most Common Colors": """
        SELECT color, 
               COUNT(*) as frequency,
               ROUND(AVG(percentage), 2) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Artifacts with Most Images": """
        SELECT a.id, a.title, COUNT(m.media_id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Culture and Classification Matrix": """
        SELECT culture, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND classification IS NOT NULL
        GROUP BY culture, classification
        ORDER BY count DESC
        LIMIT 30
    """
}

def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = pd.read_sql(query, connection)
        return df
        
    except Error as e:
        print(f"Query Error: {e}")
        return pd.DataFrame()
    finally:
        if connection.is_connected():
            connection.close()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums - ETL Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    # Execute query
    if st.sidebar.button("Run Analysis"):
        with st.spinner("Executing query..."):
            query = ANALYTICAL_QUERIES[query_name]
            df = execute_query(query)
            
            if not df.empty:
                st.subheader(f"Results: {query_name}")
                
                # Display table
                st.dataframe(df, use_container_width=True)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
                
                # Download option
                csv = df.to_csv(index=False)
                st.download_button(
                    label="Download CSV",
                    data=csv,
                    file_name=f"{query_name.replace(' ', '_')}.csv",
                    mime="text/csv"
                )
            else:
                st.error("No results returned")
    
    # ETL Pipeline Section
    st.sidebar.header("ETL Operations")
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            artifacts = collect_all_artifacts(max_pages=5)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            load_to_database(metadata_df, media_df, colors_df)
            st.success(f"✅ Loaded {len(metadata_df)} artifacts")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_latest_artifact_id():
    """Get the highest artifact ID already in database"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    connection.close()
    return result[0] if result[0] else 0

def incremental_etl():
    """Only fetch new artifacts"""
    latest_id = get_latest_artifact_id()
    # Fetch artifacts with ID > latest_id
    # Transform and load
```

### Pattern 2: Error Handling in ETL

```python
def safe_etl_pipeline(max_pages=10):
    """ETL with comprehensive error handling"""
    failed_records = []
    
    for page in range(1, max_pages + 1):
        try:
            artifacts = fetch_artifacts(page=page)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            load_to_database(metadata_df, media_df, colors_df)
        except Exception as e:
            failed_records.append({'page': page, 'error': str(e)})
            continue
    
    return failed_records
```

### Pattern 3: Visualization Helper

```python
def create_visualization(df, chart_type='bar'):
    """Generate appropriate visualization based on data"""
    if len(df.columns) == 2:
        x_col, y_col = df.columns[0], df.columns[1]
        
        if chart_type == 'bar':
            fig = px.bar(df, x=x_col, y=y_col)
        elif chart_type == 'pie':
            fig = px.pie(df, names=x_col, values=y_col)
        
        return fig
    return None
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(max_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / max_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        return wrapper
    return decorator

@rate_limit(max_per_second=2)
def fetch_artifacts_limited(page=1):
    return fetch_artifacts(page)
```

### Database Connection Pooling

```python
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return connection_pool.get_connection()
```

### Handling Missing Data

```python
def safe_transform(artifact):
    """Transform with null-safe operations"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Untitled'),
        'culture': artifact.get('culture', 'Unknown'),
        'century': artifact.get('century', 'Unknown'),
        # Handle nested fields
        'primary_image': (
            artifact['images'][0]['baseimageurl'] 
            if artifact.get('images') else None
        )
    }
```

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(filters):
    """Build SQL query from user-defined filters"""
    query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        query += f" AND culture = '{filters['culture']}'"
    if filters.get('century'):
        query += f" AND century = '{filters['century']}'"
    if filters.get('department'):
        query += f" AND department = '{filters['department']}'"
    
    return query
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards with the Harvard Art Museums API using Python, SQL, and Streamlit.
