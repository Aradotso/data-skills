---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering project with Harvard Art Museums data
  - build analytics dashboard for museum artifacts
  - extract and visualize Harvard Art Museums collection data
  - set up SQL database for museum artifact data
  - analyze Harvard Art Museums API data with Python
  - create Streamlit app for museum collection analytics
  - build end-to-end data pipeline for art museum data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application that demonstrates how to extract data from the Harvard Art Museums API, transform it into structured formats, load it into SQL databases, and visualize insights using Streamlit dashboards.

## What It Does

The Harvard Art Museums ETL Analytics app enables you to:

- **Extract** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transform** nested JSON responses into normalized relational tables
- **Load** structured data into MySQL/TiDB Cloud databases
- **Analyze** museum collections using predefined SQL queries
- **Visualize** query results through interactive Plotly charts in Streamlit

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud account
- Harvard Art Museums API key (get from https://harvardartmuseums.org/collections/api)

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

### Database Setup

Create the required database and tables:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    baseimageurl VARCHAR(500),
    format VARCHAR(100),
    description TEXT,
    technique VARCHAR(255),
    renditionnumber VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will launch in your browser at `http://localhost:8501`

## Key Components

### 1. API Data Extraction

Extract artifact data with pagination handling:

```python
import requests
import os

def fetch_artifacts(api_key, num_pages=10, page_size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_pages=5)
print(f"Fetched {len(artifacts)} artifacts")
```

### 2. ETL Pipeline

Transform nested JSON into relational format:

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform artifact data into normalized tables"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        for media in artifact.get('images', []):
            media_record = {
                'artifact_id': artifact.get('id'),
                'media_id': media.get('imageid'),
                'baseimageurl': media.get('baseimageurl'),
                'format': media.get('format'),
                'description': media.get('description'),
                'technique': media.get('technique'),
                'renditionnumber': media.get('renditionnumber')
            }
            media_records.append(media_record)
        
        # Extract color information
        for color in artifact.get('colors', []):
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return {
        'metadata': pd.DataFrame(metadata_records),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }

# Usage
transformed_data = transform_artifacts(artifacts)
print(f"Metadata records: {len(transformed_data['metadata'])}")
print(f"Media records: {len(transformed_data['media'])}")
print(f"Color records: {len(transformed_data['colors'])}")
```

### 3. Load Data into SQL

Batch insert data into database:

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(dataframes, db_config):
    """Load transformed data into MySQL database"""
    try:
        connection = mysql.connector.connect(
            host=db_config['host'],
            user=db_config['user'],
            password=db_config['password'],
            database=db_config['database']
        )
        
        if connection.is_connected():
            cursor = connection.cursor()
            
            # Load metadata
            metadata_df = dataframes['metadata']
            for _, row in metadata_df.iterrows():
                insert_query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, century, department, classification, 
                 dated, accessionyear, technique, medium, dimensions, creditline, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
                """
                cursor.execute(insert_query, tuple(row))
            
            # Load media
            media_df = dataframes['media']
            for _, row in media_df.iterrows():
                insert_query = """
                INSERT INTO artifactmedia 
                (artifact_id, media_id, baseimageurl, format, description, technique, renditionnumber)
                VALUES (%s, %s, %s, %s, %s, %s, %s)
                """
                cursor.execute(insert_query, tuple(row))
            
            # Load colors
            colors_df = dataframes['colors']
            for _, row in colors_df.iterrows():
                insert_query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
                """
                cursor.execute(insert_query, tuple(row))
            
            connection.commit()
            print("Data loaded successfully")
            
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
load_to_database(transformed_data, db_config)
```

### 4. Analytics Queries

Example analytical queries:

```python
# Query 1: Top 10 cultures by artifact count
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 3: Most common colors
query_colors = """
SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
FROM artifactcolors
GROUP BY color
ORDER BY frequency DESC
LIMIT 10;
"""

# Query 4: Department distribution
query_department = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
GROUP BY department
ORDER BY artifact_count DESC;
"""

# Query 5: Artifacts with images by classification
query_media = """
SELECT m.classification, COUNT(DISTINCT m.id) as artifacts_with_images
FROM artifactmetadata m
INNER JOIN artifactmedia am ON m.id = am.artifact_id
GROUP BY m.classification
ORDER BY artifacts_with_images DESC;
"""
```

### 5. Streamlit Dashboard

Create an interactive dashboard:

```python
import streamlit as st
import plotly.express as px

st.title("Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Configuration")
api_key = st.sidebar.text_input("API Key", type="password", value=os.getenv('HARVARD_API_KEY', ''))

# Database connection
@st.cache_resource
def get_database_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Query execution
def execute_query(query):
    conn = get_database_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Analytics section
st.header("SQL Analytics")

queries = {
    "Top Cultures": query_culture,
    "Artifacts by Century": query_century,
    "Most Common Colors": query_colors,
    "Department Distribution": query_department,
    "Media Availability": query_media
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    with st.spinner("Executing query..."):
        result_df = execute_query(queries[selected_query])
        
        st.subheader("Results")
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) == 2:
            fig = px.bar(
                result_df, 
                x=result_df.columns[0], 
                y=result_df.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Pattern 1: Incremental Data Loading

```python
def get_latest_artifact_id(connection):
    """Get the most recent artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': last_id
    }
    response = requests.get(base_url, params=params)
    return response.json().get('records', [])
```

### Pattern 2: Error Handling and Retry Logic

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff retry"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
            time.sleep(wait_time)
```

### Pattern 3: Data Quality Validation

```python
def validate_artifact_data(df):
    """Validate artifact metadata before loading"""
    validation_results = {
        'total_records': len(df),
        'missing_ids': df['id'].isna().sum(),
        'missing_titles': df['title'].isna().sum(),
        'invalid_years': df[df['accessionyear'] > 2024].shape[0],
        'duplicate_ids': df['id'].duplicated().sum()
    }
    
    # Flag issues
    issues = []
    if validation_results['missing_ids'] > 0:
        issues.append("Found records with missing IDs")
    if validation_results['duplicate_ids'] > 0:
        issues.append("Found duplicate artifact IDs")
    
    return validation_results, issues
```

## Configuration

### Environment Variables

```bash
# Required
HARVARD_API_KEY=your_api_key_from_harvard_museums
DB_HOST=your_mysql_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts

# Optional
DB_PORT=3306
API_PAGE_SIZE=100
API_MAX_PAGES=10
CACHE_TTL=3600
```

### Configuration File (config.yaml)

```yaml
api:
  base_url: "https://api.harvardartmuseums.org/object"
  page_size: 100
  max_pages: 10
  rate_limit: 1  # requests per second

database:
  pool_size: 5
  charset: "utf8mb4"
  
analytics:
  default_limit: 100
  cache_enabled: true
  
visualization:
  theme: "plotly_white"
  height: 500
```

## Troubleshooting

### Issue: API Rate Limiting

**Solution:** Implement rate limiting in your requests:

```python
import time

class RateLimiter:
    def __init__(self, calls_per_second=1):
        self.calls_per_second = calls_per_second
        self.last_call = 0
    
    def wait(self):
        elapsed = time.time() - self.last_call
        wait_time = 1.0 / self.calls_per_second - elapsed
        if wait_time > 0:
            time.sleep(wait_time)
        self.last_call = time.time()

limiter = RateLimiter(calls_per_second=1)
for page in range(1, 11):
    limiter.wait()
    response = requests.get(url, params=params)
```

### Issue: Database Connection Timeout

**Solution:** Use connection pooling:

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

### Issue: Memory Error with Large Datasets

**Solution:** Use batch processing:

```python
def batch_insert(df, table_name, batch_size=1000):
    """Insert data in batches to manage memory"""
    connection = get_database_connection()
    cursor = connection.cursor()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i + batch_size]
        # Insert batch
        for _, row in batch.iterrows():
            # Insert logic here
            pass
        connection.commit()
        print(f"Inserted batch {i//batch_size + 1}")
    
    cursor.close()
    connection.close()
```

### Issue: Missing or Null Values

**Solution:** Handle nulls during transformation:

```python
def safe_get(dictionary, key, default=None):
    """Safely get value from dictionary with default"""
    value = dictionary.get(key, default)
    return value if value is not None else default

metadata = {
    'id': artifact.get('id', 0),
    'title': safe_get(artifact, 'title', 'Unknown'),
    'culture': safe_get(artifact, 'culture', 'Not Specified'),
    'accessionyear': safe_get(artifact, 'accessionyear', 0)
}
```

## Advanced Usage

### Custom Analytics Query Builder

```python
class QueryBuilder:
    def __init__(self, table):
        self.table = table
        self.filters = []
        self.group_by = None
        self.order_by = None
        self.limit_val = None
    
    def filter(self, column, operator, value):
        self.filters.append(f"{column} {operator} '{value}'")
        return self
    
    def group(self, column):
        self.group_by = column
        return self
    
    def order(self, column, direction='DESC'):
        self.order_by = f"{column} {direction}"
        return self
    
    def limit(self, n):
        self.limit_val = n
        return self
    
    def build(self):
        query = f"SELECT * FROM {self.table}"
        if self.filters:
            query += " WHERE " + " AND ".join(self.filters)
        if self.group_by:
            query += f" GROUP BY {self.group_by}"
        if self.order_by:
            query += f" ORDER BY {self.order_by}"
        if self.limit_val:
            query += f" LIMIT {self.limit_val}"
        return query

# Usage
query = (QueryBuilder('artifactmetadata')
         .filter('century', '=', '19th century')
         .group('culture')
         .order('COUNT(*)', 'DESC')
         .limit(10)
         .build())
```

This skill provides comprehensive guidance for building ETL pipelines and analytics applications using the Harvard Art Museums API with Python, SQL, and Streamlit.
