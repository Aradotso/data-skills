---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - connect to Harvard Art Museums API
  - create analytics dashboard with Streamlit and SQL
  - extract and transform art collection data
  - visualize museum artifacts with interactive charts
  - set up data engineering pipeline for art collections
  - query and analyze Harvard museum collections
  - build artifact data warehouse with MySQL
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to work with the Harvard Art Museums data engineering and analytics application. The project demonstrates production-grade ETL pipelines that extract artifact data from the Harvard Art Museums API, transform it into relational tables, load it into SQL databases, and provide interactive analytics through Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection app provides:
- **API Integration**: Fetches artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (artifacts, media, colors)
- **SQL Database**: Stores structured data in MySQL/TiDB Cloud with proper foreign keys
- **Analytics Engine**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly charts through Streamlit UI

## Installation

### Prerequisites
```bash
# Python 3.8+
python --version

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies
```python
# requirements.txt
streamlit>=1.20.0
pandas>=1.5.0
requests>=2.28.0
pymysql>=1.0.2
plotly>=5.11.0
python-dotenv>=0.21.0
```

### Environment Setup
```bash
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Run with custom port
streamlit run app.py --server.port 8080

# Run in headless mode (for servers)
streamlit run app.py --server.headless true
```

## Core Architecture

### API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination."""
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
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return None

# Fetch multiple pages with rate limiting
import time

def fetch_multiple_pages(num_pages=5, size=100):
    """Fetch multiple pages with rate limiting."""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=size)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(1)  # Rate limiting: 1 second between requests
        else:
            break
    
    return all_artifacts
```

### ETL Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into normalized DataFrames."""
    artifacts_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Artifact metadata
        artifact_record = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        }
        artifacts_list.append(artifact_record)
        
        # Media/Images
        if artifact.get('images'):
            for img in artifact['images']:
                media_record = {
                    'objectid': artifact.get('objectid'),
                    'imageid': img.get('imageid'),
                    'baseimageurl': img.get('baseimageurl'),
                    'iiifbaseuri': img.get('iiifbaseuri'),
                    'publiccaption': img.get('publiccaption'),
                    'width': img.get('width'),
                    'height': img.get('height')
                }
                media_list.append(media_record)
        
        # Colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_record = {
                    'objectid': artifact.get('objectid'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                }
                colors_list.append(color_record)
    
    return (
        pd.DataFrame(artifacts_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Database Schema

```sql
-- Create artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    objectid INT PRIMARY KEY,
    title TEXT,
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium TEXT,
    dimensions TEXT,
    creditline TEXT,
    accessionyear INT,
    INDEX idx_culture (culture),
    INDEX idx_department (department),
    INDEX idx_century (century)
);

-- Create media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    imageid INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    publiccaption TEXT,
    width INT,
    height INT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid) ON DELETE CASCADE,
    INDEX idx_objectid (objectid)
);

-- Create colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(100),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid) ON DELETE CASCADE,
    INDEX idx_objectid (objectid),
    INDEX idx_color (color)
);
```

### Database Loading

```python
import pymysql
from pymysql import Error

def get_db_connection():
    """Create database connection."""
    try:
        connection = pymysql.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            port=int(os.getenv('DB_PORT', 3306)),
            charset='utf8mb4'
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_artifacts_to_db(artifacts_df, media_df, colors_df):
    """Batch insert DataFrames into database."""
    conn = get_db_connection()
    if not conn:
        return False
    
    try:
        cursor = conn.cursor()
        
        # Insert artifacts (replace duplicates)
        for _, row in artifacts_df.iterrows():
            cursor.execute("""
                REPLACE INTO artifactmetadata 
                (objectid, title, culture, period, century, classification, 
                 department, dated, medium, dimensions, creditline, accessionyear)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (objectid, imageid, baseimageurl, iiifbaseuri, publiccaption, width, height)
                VALUES (%s, %s, %s, %s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (objectid, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(artifacts_df)} artifacts, {len(media_df)} media, {len(colors_df)} colors")
        return True
        
    except Error as e:
        print(f"Database load error: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

### Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Department": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Most Common Cultures": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts with Media": """
        SELECT 
            COUNT(DISTINCT a.objectid) as artifacts_with_media,
            COUNT(*) as total_media_items,
            AVG(m.width) as avg_width,
            AVG(m.height) as avg_height
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.objectid = m.objectid
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Artifacts by Classification": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 15
    """
}

def execute_query(query):
    """Execute SQL query and return DataFrame."""
    conn = get_db_connection()
    if not conn:
        return None
    
    try:
        df = pd.read_sql(query, conn)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        conn.close()
```

### Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Function",
        ["Data Collection", "Analytics Dashboard", "Database Stats"]
    )
    
    if page == "Data Collection":
        st.header("📥 Data Collection from API")
        
        num_pages = st.number_input("Number of pages to fetch", 1, 20, 5)
        page_size = st.slider("Records per page", 10, 100, 100)
        
        if st.button("Fetch and Load Data"):
            with st.spinner("Fetching from API..."):
                raw_data = fetch_multiple_pages(num_pages, page_size)
                
            with st.spinner("Transforming data..."):
                artifacts_df, media_df, colors_df = transform_artifacts(raw_data)
                
            st.write(f"✅ Transformed {len(artifacts_df)} artifacts")
            st.dataframe(artifacts_df.head())
            
            with st.spinner("Loading to database..."):
                success = load_artifacts_to_db(artifacts_df, media_df, colors_df)
                
            if success:
                st.success("✅ Data loaded successfully!")
    
    elif page == "Analytics Dashboard":
        st.header("📊 SQL Analytics")
        
        query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Run Analysis"):
            query = ANALYTICS_QUERIES[query_name]
            
            with st.spinner("Executing query..."):
                df = execute_query(query)
            
            if df is not None and not df.empty:
                st.dataframe(df)
                
                # Auto-generate visualization
                if len(df.columns) == 2:
                    fig = px.bar(
                        df, 
                        x=df.columns[0], 
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Database Stats":
        st.header("📈 Database Statistics")
        
        stats_query = """
            SELECT 
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
                (SELECT COUNT(*) FROM artifactmedia) as total_media,
                (SELECT COUNT(*) FROM artifactcolors) as total_colors
        """
        
        df = execute_query(stats_query)
        if df is not None:
            col1, col2, col3 = st.columns(3)
            col1.metric("Total Artifacts", df['total_artifacts'][0])
            col2.metric("Total Media Items", df['total_media'][0])
            col3.metric("Total Color Records", df['total_colors'][0])

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline
```python
def run_full_etl_pipeline(num_pages=5):
    """Run complete ETL pipeline."""
    print("Starting ETL pipeline...")
    
    # Extract
    print("1. Extracting data from API...")
    raw_data = fetch_multiple_pages(num_pages)
    
    # Transform
    print("2. Transforming data...")
    artifacts_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("3. Loading to database...")
    success = load_artifacts_to_db(artifacts_df, media_df, colors_df)
    
    if success:
        print("✅ ETL pipeline completed successfully")
    else:
        print("❌ ETL pipeline failed")
    
    return success
```

### Error Handling Pattern
```python
def safe_api_fetch(page, max_retries=3):
    """Fetch with retry logic."""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(page=page)
            return data
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                return None
```

## Troubleshooting

### API Key Issues
```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')
if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")
```

### Database Connection Errors
```python
# Test database connection
def test_db_connection():
    try:
        conn = get_db_connection()
        if conn:
            print("✅ Database connection successful")
            cursor = conn.cursor()
            cursor.execute("SELECT VERSION()")
            version = cursor.fetchone()
            print(f"Database version: {version[0]}")
            conn.close()
            return True
    except Exception as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Memory Management for Large Datasets
```python
def batch_load_artifacts(artifacts_df, batch_size=1000):
    """Load large datasets in batches."""
    total_rows = len(artifacts_df)
    
    for i in range(0, total_rows, batch_size):
        batch = artifacts_df.iloc[i:i+batch_size]
        load_artifacts_to_db(batch, pd.DataFrame(), pd.DataFrame())
        print(f"Loaded batch {i//batch_size + 1} ({i+len(batch)}/{total_rows})")
```

### Rate Limiting
```python
# Respect API rate limits (default: 2500 requests/day)
import time
from functools import wraps

def rate_limit(delay=1):
    """Decorator to add delay between API calls."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            time.sleep(delay)
            return result
        return wrapper
    return decorator

@rate_limit(delay=1)
def fetch_with_rate_limit(page):
    return fetch_artifacts(page=page)
```

## Advanced Usage

### Custom Query Execution
```python
def run_custom_query(sql_query):
    """Execute custom SQL query with validation."""
    # Basic SQL injection prevention
    dangerous_keywords = ['DROP', 'DELETE', 'TRUNCATE', 'ALTER']
    if any(keyword in sql_query.upper() for keyword in dangerous_keywords):
        raise ValueError("Query contains dangerous keywords")
    
    return execute_query(sql_query)
```

### Export Results
```python
def export_query_results(query_name, output_format='csv'):
    """Export query results to file."""
    df = execute_query(ANALYTICS_QUERIES[query_name])
    
    if df is not None:
        filename = f"{query_name.replace(' ', '_').lower()}.{output_format}"
        
        if output_format == 'csv':
            df.to_csv(filename, index=False)
        elif output_format == 'json':
            df.to_json(filename, orient='records', indent=2)
        
        print(f"Results exported to {filename}")
        return filename
```

This skill enables AI agents to build, configure, and extend the Harvard Art Museums ETL and analytics application with proper data engineering practices.
