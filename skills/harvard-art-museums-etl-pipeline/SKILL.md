---
name: harvard-art-museums-etl-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for the Harvard Art Museums API
  - show me how to extract and load Harvard museum data into SQL
  - help me create a Streamlit analytics dashboard for art museum data
  - how do I fetch and transform Harvard Art Museums API data
  - guide me through setting up a data engineering pipeline with museum artifacts
  - how to visualize Harvard museum collection data with Plotly
  - help me design SQL schemas for art museum metadata
  - show me how to paginate through Harvard Art Museums API
---

# Harvard Art Museums ETL Pipeline & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering solution for extracting, transforming, and analyzing art museum data from the Harvard Art Museums API. It demonstrates ETL pipeline development, SQL database design, and interactive analytics dashboards using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App:
- **Extracts** artifact data from Harvard Art Museums API with pagination
- **Transforms** nested JSON into normalized relational database schemas
- **Loads** data into MySQL/TiDB Cloud with batch operations
- **Analyzes** data using predefined SQL queries
- **Visualizes** insights through interactive Plotly charts in Streamlit

**Architecture Flow**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Key Dependencies**:
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://docs.harvardartmuseums.org/api/)
2. Request an API key (free)
3. Add to your `.env` file

### Database Setup

```sql
-- Create database
CREATE DATABASE harvard_artifacts;

-- Create artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accessionyear INT,
    technique VARCHAR(300),
    verificationlevel INT
);

-- Create artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    publiccaption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Create artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(100),
    spectrum VARCHAR(100),
    hue VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit app will open in your browser at `http://localhost:8501`.

## Key Components

### 1. API Data Extraction

**Fetching Artifacts with Pagination**:

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')

def fetch_artifacts(num_pages=5, page_size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': API_KEY,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data['records'])
            print(f"Fetched page {page}/{num_pages}")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### 2. Data Transformation

**Transform Nested JSON to DataFrames**:

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform raw API data into normalized dataframes"""
    
    # Main metadata
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'verificationlevel': artifact.get('verificationlevel')
        })
        
        # Extract media (images)
        if artifact.get('images'):
            for image in artifact['images']:
                media_records.append({
                    'artifact_id': artifact['id'],
                    'iiifbaseuri': image.get('iiifbaseuri'),
                    'baseimageurl': image.get('baseimageurl'),
                    'publiccaption': image.get('publiccaption')
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact['id'],
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

**Batch Insert with MySQL**:

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=os.getenv('DB_PORT'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_to_database(df_metadata, df_media, df_colors):
    """Load dataframes to MySQL database"""
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        for _, row in df_metadata.iterrows():
            query = """
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 division, department, dated, medium, dimensions, 
                 creditline, accessionyear, technique, verificationlevel)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(query, tuple(row))
        
        # Load media
        for _, row in df_media.iterrows():
            query = """
                INSERT INTO artifactmedia 
                (artifact_id, iiifbaseuri, baseimageurl, publiccaption)
                VALUES (%s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        # Load colors
        for _, row in df_colors.iterrows():
            query = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.execute(query, tuple(row))
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. SQL Analytics Queries

**Example Analytical Queries**:

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
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
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Color Usage": """
        SELECT color, COUNT(*) as usage_count, 
               AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(media.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Classification by Century": """
        SELECT century, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND classification IS NOT NULL
        GROUP BY century, classification
        ORDER BY century, count DESC
    """
}

def execute_query(query):
    """Execute SQL query and return results"""
    connection = create_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        connection.close()
```

### 5. Streamlit Dashboard

**Building the Interactive UI**:

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for ETL controls
    with st.sidebar:
        st.header("ETL Pipeline")
        
        num_pages = st.number_input("Pages to fetch", 1, 50, 5)
        
        if st.button("🔄 Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_artifacts(num_pages=num_pages)
                st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                df_meta, df_media, df_colors = transform_artifacts(artifacts)
                st.success("Data transformed")
            
            with st.spinner("Loading to database..."):
                success = load_to_database(df_meta, df_media, df_colors)
                if success:
                    st.success("✅ ETL Complete!")
    
    # Main analytics section
    st.header("📊 Analytics Queries")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        with st.spinner("Executing query..."):
            df_result = execute_query(query)
        
        if df_result is not None and not df_result.empty:
            st.subheader("Results")
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                x_col = df_result.columns[0]
                y_col = df_result.columns[1]
                
                fig = px.bar(
                    df_result.head(20),
                    x=x_col,
                    y=y_col,
                    title=query_name,
                    color=y_col,
                    color_continuous_scale='viridis'
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(num_pages, delay=0.5):
    """Fetch data with rate limiting to respect API limits"""
    artifacts = []
    for page in range(1, num_pages + 1):
        response = requests.get(url, params=params)
        if response.status_code == 200:
            artifacts.extend(response.json()['records'])
        time.sleep(delay)  # Prevent hitting rate limits
    return artifacts
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default=None):
    """Safely extract nested values"""
    value = dictionary.get(key, default)
    return value if value not in [None, '', 'null'] else default

# Usage
culture = safe_get(artifact, 'culture', 'Unknown')
```

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID in database"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    result = execute_query(query)
    return result['max_id'][0] if result is not None else 0

def fetch_new_artifacts_only():
    """Fetch only artifacts newer than what we have"""
    last_id = get_last_artifact_id()
    # Fetch artifacts with id > last_id
    params = {'after': last_id}
    # Implementation depends on API capabilities
```

## Troubleshooting

### API Connection Issues

**Problem**: `requests.exceptions.ConnectionError`

**Solution**: Check internet connection and API key validity:
```python
def test_api_connection():
    test_url = f"https://api.harvardartmuseums.org/object?apikey={API_KEY}&size=1"
    try:
        response = requests.get(test_url, timeout=10)
        return response.status_code == 200
    except Exception as e:
        print(f"API test failed: {e}")
        return False
```

### Database Connection Errors

**Problem**: `mysql.connector.errors.DatabaseError`

**Solution**: Verify credentials and network access:
```python
def test_db_connection():
    try:
        connection = create_db_connection()
        if connection and connection.is_connected():
            print("✅ Database connected")
            connection.close()
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Memory Issues with Large Datasets

**Problem**: Out of memory when processing thousands of artifacts

**Solution**: Use chunked processing:
```python
def process_in_chunks(artifacts, chunk_size=100):
    """Process data in smaller batches"""
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        df_meta, df_media, df_colors = transform_artifacts(chunk)
        load_to_database(df_meta, df_media, df_colors)
```

### Duplicate Key Errors

**Problem**: Trying to insert artifacts that already exist

**Solution**: Use `ON DUPLICATE KEY UPDATE` or check existence:
```python
# Already handled in the INSERT query above with:
# ON DUPLICATE KEY UPDATE title=VALUES(title)
```

### Empty Visualization

**Problem**: Chart not rendering or showing empty

**Solution**: Validate data before plotting:
```python
if df_result is not None and not df_result.empty and len(df_result) > 0:
    # Only create visualization if we have data
    fig = px.bar(df_result, ...)
else:
    st.warning("No data available for visualization")
```

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(table, filters, aggregations):
    """Dynamically build SQL queries"""
    query = f"SELECT {', '.join(aggregations)} FROM {table}"
    
    if filters:
        where_clauses = [f"{k} = '{v}'" for k, v in filters.items()]
        query += f" WHERE {' AND '.join(where_clauses)}"
    
    return query

# Usage in Streamlit
culture_filter = st.text_input("Filter by culture")
if culture_filter:
    custom_query = build_custom_query(
        'artifactmetadata',
        {'culture': culture_filter},
        ['classification', 'COUNT(*) as count']
    )
    result = execute_query(custom_query)
```

This skill enables AI agents to guide developers through building production-ready ETL pipelines for museum data, from API integration through interactive analytics dashboards.
