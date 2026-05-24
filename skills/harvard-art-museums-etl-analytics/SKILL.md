---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for museum data
  - create analytics dashboard with Harvard Art Museums API
  - extract and transform art collection data
  - design SQL schema for museum artifacts
  - visualize museum collection analytics
  - fetch data from Harvard Art Museums API
  - create streamlit app for art data
  - analyze artifact metadata with SQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What It Does

This project demonstrates a complete data engineering workflow:

1. **Extract** artifact data from Harvard Art Museums API
2. **Transform** nested JSON into relational database structures
3. **Load** data into MySQL/TiDB Cloud with proper schema design
4. **Analyze** using SQL queries for insights
5. **Visualize** results in an interactive Streamlit dashboard

The application handles pagination, rate limiting, nested data structures, and creates a production-ready analytics interface.

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
export DB_NAME="your_database_name"
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

## Database Schema Design

The project uses a relational schema with three main tables:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    dated VARCHAR(200),
    division VARCHAR(200),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## API Integration Pattern

### Fetching Data with Pagination

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, page: int = 1, size: int = 100) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API with pagination.
    
    Args:
        api_key: Harvard API key from environment
        page: Page number to fetch
        size: Number of records per page (max 100)
    
    Returns:
        Dict containing 'records' list and 'info' metadata
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size,
        "hasimage": 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

def collect_all_artifacts(api_key: str, max_records: int = 1000) -> List[Dict]:
    """
    Collect multiple pages of artifact data.
    """
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
    
    return all_records[:max_records]
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def extract_metadata(records: List[Dict]) -> pd.DataFrame:
    """
    Extract artifact metadata into flat structure.
    """
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'department': record.get('department'),
            'classification': record.get('classification'),
            'dated': record.get('dated'),
            'division': record.get('division'),
            'url': record.get('url')
        })
    
    return pd.DataFrame(metadata)

def extract_media(records: List[Dict]) -> pd.DataFrame:
    """
    Extract nested media information.
    """
    media_list = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for image in images:
            media_list.append({
                'artifact_id': artifact_id,
                'base_url': image.get('baseimageurl'),
                'format': image.get('format'),
                'height': image.get('height'),
                'width': image.get('width')
            })
    
    return pd.DataFrame(media_list)

def extract_colors(records: List[Dict]) -> pd.DataFrame:
    """
    Extract color palette data.
    """
    colors_list = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            colors_list.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return pd.DataFrame(colors_list)
```

### Load into Database

```python
import mysql.connector
from mysql.connector import Error

def get_database_connection():
    """
    Create database connection using environment variables.
    """
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

def batch_insert_metadata(df: pd.DataFrame):
    """
    Batch insert artifact metadata with error handling.
    """
    connection = get_database_connection()
    if not connection:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, period, century, department, classification, dated, division, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    try:
        # Convert DataFrame to list of tuples
        records = df.to_records(index=False).tolist()
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
    finally:
        cursor.close()
        connection.close()

def batch_insert_media(df: pd.DataFrame):
    """
    Batch insert media records.
    """
    connection = get_database_connection()
    if not connection:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, base_url, format, height, width)
        VALUES (%s, %s, %s, %s, %s)
    """
    
    try:
        records = df.to_records(index=False).tolist()
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()
        connection.close()
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for API configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Data Collection Section
    st.header("1️⃣ Data Collection")
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=5000, value=100)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data from API..."):
            records = collect_all_artifacts(api_key, max_records=num_records)
            
            # Extract data
            metadata_df = extract_metadata(records)
            media_df = extract_media(records)
            colors_df = extract_colors(records)
            
            # Load into database
            batch_insert_metadata(metadata_df)
            batch_insert_media(media_df)
            batch_insert_colors(colors_df)
            
            st.success(f"✅ Loaded {len(records)} artifacts successfully!")
    
    # Analytics Section
    st.header("2️⃣ SQL Analytics")
    
    analytics_queries = {
        "Top 10 Cultures by Artifact Count": """
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
        "Department Distribution": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        "Most Common Colors": """
            SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color_name IS NOT NULL
            GROUP BY color_name
            ORDER BY frequency DESC
            LIMIT 10
        """,
        "Media Format Distribution": """
            SELECT format, COUNT(*) as count
            FROM artifactmedia
            WHERE format IS NOT NULL
            GROUP BY format
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(analytics_queries.keys()))
    
    if st.button("Run Query"):
        query = analytics_queries[selected_query]
        result_df = execute_query(query)
        
        if not result_df.empty:
            st.dataframe(result_df)
            
            # Auto-generate visualization
            if len(result_df.columns) >= 2:
                fig = px.bar(result_df, 
                            x=result_df.columns[0], 
                            y=result_df.columns[1],
                            title=selected_query)
                st.plotly_chart(fig)

def execute_query(query: str) -> pd.DataFrame:
    """
    Execute SQL query and return results as DataFrame.
    """
    connection = get_database_connection()
    if not connection:
        return pd.DataFrame()
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        connection.close()

if __name__ == "__main__":
    main()
```

## Common Analytics Queries

### Temporal Analysis

```sql
-- Artifact distribution by time period
SELECT 
    period,
    COUNT(*) as artifact_count,
    COUNT(DISTINCT culture) as unique_cultures
FROM artifactmetadata
WHERE period IS NOT NULL
GROUP BY period
ORDER BY artifact_count DESC;
```

### Color Analysis

```sql
-- Dominant colors across collections
SELECT 
    color_name,
    COUNT(DISTINCT artifact_id) as artifacts_with_color,
    AVG(percentage) as avg_dominance
FROM artifactcolors
GROUP BY color_name
HAVING COUNT(DISTINCT artifact_id) > 10
ORDER BY artifacts_with_color DESC;
```

### Media Availability

```sql
-- Artifacts with multiple images
SELECT 
    a.title,
    a.culture,
    COUNT(m.media_id) as image_count
FROM artifactmetadata a
JOIN artifactmedia m ON a.id = m.artifact_id
GROUP BY a.id, a.title, a.culture
HAVING image_count > 5
ORDER BY image_count DESC;
```

### Cross-Department Analysis

```sql
-- Classification distribution by department
SELECT 
    department,
    classification,
    COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC;
```

## Configuration Management

```python
from dotenv import load_dotenv
import os

# Load environment variables
load_dotenv()

class Config:
    """Centralized configuration management."""
    
    # API Configuration
    HARVARD_API_KEY = os.getenv('HARVARD_API_KEY')
    API_BASE_URL = "https://api.harvardartmuseums.org/object"
    
    # Database Configuration
    DB_HOST = os.getenv('DB_HOST', 'localhost')
    DB_PORT = int(os.getenv('DB_PORT', 3306))
    DB_USER = os.getenv('DB_USER')
    DB_PASSWORD = os.getenv('DB_PASSWORD')
    DB_NAME = os.getenv('DB_NAME', 'harvard_artifacts')
    
    # ETL Configuration
    BATCH_SIZE = int(os.getenv('BATCH_SIZE', 100))
    MAX_RECORDS = int(os.getenv('MAX_RECORDS', 1000))
    
    @classmethod
    def validate(cls):
        """Validate required configuration."""
        required = ['HARVARD_API_KEY', 'DB_USER', 'DB_PASSWORD']
        missing = [key for key in required if not getattr(cls, key)]
        
        if missing:
            raise ValueError(f"Missing required config: {', '.join(missing)}")
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limited(max_per_second=10):
    """Decorator to rate limit API calls."""
    min_interval = 1.0 / max_per_second
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

@rate_limited(max_per_second=5)
def fetch_artifacts_rate_limited(api_key: str, page: int = 1):
    return fetch_artifacts(api_key, page)
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

def get_pooled_connection():
    """Get connection from pool."""
    return connection_pool.get_connection()
```

### Handling Missing Data

```python
def clean_record(record: Dict) -> Dict:
    """Clean and normalize record data."""
    return {
        'id': record.get('id'),
        'title': record.get('title', 'Untitled'),
        'culture': record.get('culture') or 'Unknown',
        'period': record.get('period') or 'Unknown',
        'century': record.get('century') or 'Unknown',
        'department': record.get('department') or 'Uncategorized',
        'classification': record.get('classification') or 'Unclassified',
        'dated': record.get('dated'),
        'division': record.get('division'),
        'url': record.get('url')
    }
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Run with custom port
streamlit run app.py --server.port 8080

# Run with specific config
streamlit run app.py --server.address localhost
```
