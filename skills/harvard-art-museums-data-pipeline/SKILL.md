---
name: harvard-art-museums-data-pipeline
description: ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build a data pipeline from Harvard Art Museums API
  - create ETL pipeline for museum artifact data
  - fetch and analyze Harvard museum collections
  - set up artifact analytics dashboard with Streamlit
  - extract museum data into SQL database
  - visualize art museum collection analytics
  - implement Harvard API data engineering workflow
  - analyze artifact metadata with SQL queries
---

# Harvard Art Museums Data Pipeline & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates:

- **API Integration**: Paginated data collection from Harvard Art Museums
- **ETL Pipeline**: Transform nested JSON into relational database schema
- **SQL Analytics**: 20+ predefined analytical queries
- **Interactive Dashboard**: Streamlit-based visualization with Plotly

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

## Installation

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

## Database Schema

The pipeline creates three relational tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    provenance TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Media/images table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Color analysis table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Harvard Art Museums API Integration

### API Authentication

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination support"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Example usage
data = fetch_artifacts(page=1, size=50)
records = data.get('records', [])
total_records = data.get('info', {}).get('totalrecords', 0)
```

### Handling Pagination

```python
def fetch_all_artifacts(max_records=1000):
    """Fetch artifacts across multiple pages"""
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        try:
            data = fetch_artifacts(page=page, size=size)
            records = data.get('records', [])
            
            if not records:
                break
                
            all_records.extend(records)
            page += 1
            
            # Rate limiting
            time.sleep(0.5)
            
        except requests.exceptions.RequestException as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_records[:max_records]
```

## ETL Pipeline Implementation

### Extract & Transform

```python
import pandas as pd

def transform_artifact_metadata(records):
    """Transform API records into metadata dataframe"""
    metadata_list = []
    
    for record in records:
        metadata = {
            'objectid': record.get('objectid'),
            'title': record.get('title', ''),
            'culture': record.get('culture', ''),
            'century': record.get('century', ''),
            'classification': record.get('classification', ''),
            'department': record.get('department', ''),
            'dated': record.get('dated', ''),
            'description': record.get('description', ''),
            'provenance': record.get('provenance', ''),
            'totalpageviews': record.get('totalpageviews', 0),
            'totaluniquepageviews': record.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(records):
    """Extract media/image data"""
    media_list = []
    
    for record in records:
        objectid = record.get('objectid')
        images = record.get('images', [])
        
        for image in images:
            media = {
                'objectid': objectid,
                'baseimageurl': image.get('baseimageurl', ''),
                'iiifbaseuri': image.get('iiifbaseuri', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(records):
    """Extract color analysis data"""
    colors_list = []
    
    for record in records:
        objectid = record.get('objectid')
        colors = record.get('colors', [])
        
        for color in colors:
            color_data = {
                'objectid': objectid,
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return pd.DataFrame(colors_list)
```

### Load to SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata_to_db(df_metadata, connection):
    """Batch insert metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, culture, century, classification, department, 
     dated, description, provenance, totalpageviews, totaluniquepageviews)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media_to_db(df_media, connection):
    """Insert media records"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (objectid, baseimageurl, iiifbaseuri)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} media records")

def load_colors_to_db(df_colors, connection):
    """Insert color analysis records"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_colors.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} color records")
```

## Complete ETL Workflow

```python
def run_etl_pipeline(num_records=500):
    """Execute complete ETL pipeline"""
    print("Starting ETL Pipeline...")
    
    # Extract
    print("Extracting data from Harvard API...")
    records = fetch_all_artifacts(max_records=num_records)
    print(f"Extracted {len(records)} records")
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(records)
    df_media = transform_artifact_media(records)
    df_colors = transform_artifact_colors(records)
    
    # Load
    print("Loading data to database...")
    connection = create_database_connection()
    
    if connection:
        load_metadata_to_db(df_metadata, connection)
        load_media_to_db(df_media, connection)
        load_colors_to_db(df_colors, connection)
        connection.close()
        print("ETL Pipeline completed successfully!")
    else:
        print("Failed to connect to database")
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by culture
QUERY_ARTIFACTS_BY_CULTURE = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 15
"""

# Query 2: Most viewed artifacts
QUERY_TOP_VIEWED = """
SELECT objectid, title, totalpageviews, culture, century
FROM artifactmetadata
ORDER BY totalpageviews DESC
LIMIT 20
"""

# Query 3: Color distribution
QUERY_COLOR_DISTRIBUTION = """
SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY count DESC
LIMIT 10
"""

# Query 4: Artifacts by department and century
QUERY_DEPT_CENTURY = """
SELECT department, century, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND century IS NOT NULL
GROUP BY department, century
ORDER BY department, count DESC
"""

# Query 5: Media availability analysis
QUERY_MEDIA_AVAILABILITY = """
SELECT 
    COUNT(DISTINCT am.objectid) as total_artifacts,
    COUNT(DISTINCT media.objectid) as artifacts_with_images,
    ROUND(COUNT(DISTINCT media.objectid) * 100.0 / COUNT(DISTINCT am.objectid), 2) as image_percentage
FROM artifactmetadata am
LEFT JOIN artifactmedia media ON am.objectid = media.objectid
"""

def execute_query(query, connection):
    """Execute SQL query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Basic App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualizations_page()

def show_data_collection_page():
    """Data collection interface"""
    st.header("📥 Data Collection from Harvard API")
    
    num_records = st.number_input(
        "Number of records to collect",
        min_value=10,
        max_value=5000,
        value=500,
        step=50
    )
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            run_etl_pipeline(num_records)
            st.success(f"Successfully collected and loaded {num_records} artifacts!")

def show_analytics_page():
    """SQL analytics interface"""
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": QUERY_ARTIFACTS_BY_CULTURE,
        "Top Viewed Artifacts": QUERY_TOP_VIEWED,
        "Color Distribution": QUERY_COLOR_DISTRIBUTION,
        "Department & Century Analysis": QUERY_DEPT_CENTURY,
        "Media Availability": QUERY_MEDIA_AVAILABILITY
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        connection = create_database_connection()
        if connection:
            df_results = execute_query(queries[selected_query], connection)
            
            st.subheader("Query Results")
            st.dataframe(df_results)
            
            # Auto-visualization
            if len(df_results.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(
                    df_results.head(15),
                    x=df_results.columns[0],
                    y=df_results.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)
            
            connection.close()

def show_visualizations_page():
    """Custom visualizations page"""
    st.header("📈 Custom Visualizations")
    
    connection = create_database_connection()
    
    if connection:
        # Example: Culture distribution pie chart
        df_culture = execute_query(QUERY_ARTIFACTS_BY_CULTURE, connection)
        
        col1, col2 = st.columns(2)
        
        with col1:
            fig_pie = px.pie(
                df_culture.head(10),
                values='artifact_count',
                names='culture',
                title='Top 10 Cultures by Artifact Count'
            )
            st.plotly_chart(fig_pie)
        
        with col2:
            fig_bar = px.bar(
                df_culture.head(10),
                x='culture',
                y='artifact_count',
                title='Artifact Distribution by Culture'
            )
            st.plotly_chart(fig_bar)
        
        connection.close()

if __name__ == "__main__":
    main()
```

### Running the Streamlit App

```bash
streamlit run app.py
```

## Configuration

### Environment Variables

Create a `.env` file:

```bash
# Harvard API
HARVARD_API_KEY=your_harvard_api_key

# Database Configuration
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Loading Environment Variables

```python
from dotenv import load_dotenv
import os

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(num_records=500):
    """ETL with comprehensive error handling"""
    try:
        # Extract
        records = fetch_all_artifacts(max_records=num_records)
        
        if not records:
            raise ValueError("No records fetched from API")
        
        # Transform
        df_metadata = transform_artifact_metadata(records)
        df_media = transform_artifact_media(records)
        df_colors = transform_artifact_colors(records)
        
        # Validate data
        if df_metadata.empty:
            raise ValueError("Metadata transformation resulted in empty DataFrame")
        
        # Load
        connection = create_database_connection()
        
        if not connection:
            raise ConnectionError("Failed to connect to database")
        
        try:
            load_metadata_to_db(df_metadata, connection)
            load_media_to_db(df_media, connection)
            load_colors_to_db(df_colors, connection)
        finally:
            connection.close()
        
        return True
        
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return False
    except mysql.connector.Error as e:
        print(f"Database Error: {e}")
        return False
    except Exception as e:
        print(f"Unexpected Error: {e}")
        return False
```

### Incremental Data Loading

```python
def get_existing_object_ids(connection):
    """Get already loaded artifact IDs"""
    cursor = connection.cursor()
    cursor.execute("SELECT objectid FROM artifactmetadata")
    return {row[0] for row in cursor.fetchall()}

def incremental_etl(num_records=500):
    """Only load new artifacts"""
    connection = create_database_connection()
    existing_ids = get_existing_object_ids(connection)
    
    records = fetch_all_artifacts(max_records=num_records)
    new_records = [r for r in records if r.get('objectid') not in existing_ids]
    
    print(f"Found {len(new_records)} new artifacts to load")
    
    if new_records:
        df_metadata = transform_artifact_metadata(new_records)
        load_metadata_to_db(df_metadata, connection)
    
    connection.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limit(calls_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            wait_time = min_interval - elapsed
            
            if wait_time > 0:
                time.sleep(wait_time)
            
            result = func(*args, **kwargs)
            last_called[0] = time.time()
            return result
        return wrapper
    return decorator

@rate_limit(calls_per_second=2)
def fetch_artifacts_rate_limited(page=1, size=100):
    return fetch_artifacts(page, size)
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

def get_connection_from_pool():
    """Get connection from pool"""
    return connection_pool.get_connection()
```

### Handling Missing Data

```python
def safe_get(record, key, default=''):
    """Safely extract nested values"""
    try:
        value = record.get(key, default)
        return value if value is not None else default
    except:
        return default

def transform_with_defaults(records):
    """Transform with comprehensive null handling"""
    metadata_list = []
    
    for record in records:
        metadata = {
            'objectid': safe_get(record, 'objectid', 0),
            'title': safe_get(record, 'title', 'Untitled')[:500],
            'culture': safe_get(record, 'culture', 'Unknown')[:200],
            'century': safe_get(record, 'century', 'Unknown')[:100],
            'totalpageviews': int(safe_get(record, 'totalpageviews', 0))
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards using the Harvard Art Museums API, SQL databases, and Streamlit visualization.
