---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts
  - extract and load Harvard museum data to SQL
  - visualize Harvard Art Museums collection data
  - set up data engineering pipeline for museum artifacts
  - query and analyze Harvard art collection database
  - transform Harvard API data into relational tables
  - build Streamlit app for museum data analytics
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill provides expertise in building end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App enables:

- **API Integration**: Collect artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational SQL tables
- **Database Design**: Store artifacts, media, and color data in normalized SQL schema
- **SQL Analytics**: Run 20+ predefined analytical queries for insights
- **Interactive Visualization**: Build Streamlit dashboards with Plotly charts

**Architecture Flow**: API → ETL → SQL (MySQL/TiDB) → Analytics → Streamlit Visualization

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Project Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install requirements
pip install -r requirements.txt

# Create environment configuration
cat > .env << EOF
HARVARD_API_KEY=your_api_key_here
MYSQL_HOST=your_mysql_host
MYSQL_USER=your_mysql_user
MYSQL_PASSWORD=your_mysql_password
MYSQL_DATABASE=harvard_artifacts
EOF
```

### Get Harvard API Key

1. Register at https://www.harvardartmuseums.org/collections/api
2. Get your API key from the dashboard
3. Add to `.env` file as `HARVARD_API_KEY`

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch first 100 artifacts
artifacts, info = fetch_artifacts(page=1, size=100)
print(f"Total records available: {info['totalrecords']}")
print(f"Fetched {len(artifacts)} artifacts")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> pd.DataFrame:
    """Transform raw API data into structured DataFrame"""
    transformed = []
    
    for artifact in raw_data:
        record = {
            'object_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'division': artifact.get('division'),
            'creditline': artifact.get('creditline')
        }
        transformed.append(record)
    
    return pd.DataFrame(transformed)

def transform_media(raw_data: List[Dict]) -> pd.DataFrame:
    """Extract media/image information"""
    media_records = []
    
    for artifact in raw_data:
        object_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'object_id': object_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'width': img.get('width'),
                'height': img.get('height'),
                'format': img.get('format')
            })
    
    return pd.DataFrame(media_records)

def transform_colors(raw_data: List[Dict]) -> pd.DataFrame:
    """Extract color palette information"""
    color_records = []
    
    for artifact in raw_data:
        object_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'object_id': object_id,
                'color_hex': color.get('hex'),
                'color_name': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return pd.DataFrame(color_records)
```

### 3. Database Schema and Loading

```python
def create_database_schema(connection):
    """Create normalized database schema"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            period VARCHAR(200),
            medium TEXT,
            dimensions VARCHAR(500),
            division VARCHAR(200),
            creditline TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url TEXT,
            width INT,
            height INT,
            format VARCHAR(50),
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color_hex VARCHAR(10),
            color_name VARCHAR(100),
            percentage FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_to_database(df: pd.DataFrame, table_name: str, connection):
    """Batch insert DataFrame into SQL table"""
    cursor = connection.cursor()
    
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"""
        INSERT INTO {table_name} ({columns}) 
        VALUES ({placeholders})
        ON DUPLICATE KEY UPDATE {columns}={columns}
    """
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data_tuples)
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(df)} records into {table_name}")
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "media_availability": """
        SELECT 
            CASE WHEN m.object_id IS NOT NULL THEN 'With Media' 
                 ELSE 'Without Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.object_id = m.object_id
        GROUP BY media_status
    """,
    
    "top_colors": """
        SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "classification_summary": """
        SELECT classification, COUNT(*) as count, 
               GROUP_CONCAT(DISTINCT culture) as cultures
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

def execute_analytics_query(query_name: str, connection):
    """Execute analytical query and return results"""
    cursor = connection.cursor(dictionary=True)
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        raise ValueError(f"Query '{query_name}' not found")
    
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("---")
    
    # Database connection
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_options = list(ANALYTICS_QUERIES.keys())
    selected_query = st.sidebar.selectbox("Select Query", query_options)
    
    # Execute and display
    if st.sidebar.button("Run Query"):
        with st.spinner("Executing query..."):
            df = execute_analytics_query(selected_query, conn)
            
            st.subheader(f"Results: {selected_query.replace('_', ' ').title()}")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=f"{selected_query.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit app
streamlit run app.py

# Access dashboard at
# http://localhost:8501
```

## Common Patterns

### Full ETL Workflow

```python
import os
from dotenv import load_dotenv
import mysql.connector

load_dotenv()

def run_full_etl_pipeline(num_pages=5):
    """Execute complete ETL pipeline"""
    
    # 1. Extract
    all_artifacts = []
    for page in range(1, num_pages + 1):
        artifacts, info = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(artifacts)
        print(f"Fetched page {page}/{num_pages}")
    
    # 2. Transform
    metadata_df = transform_artifacts(all_artifacts)
    media_df = transform_media(all_artifacts)
    colors_df = transform_colors(all_artifacts)
    
    # 3. Load
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    create_database_schema(conn)
    
    load_to_database(metadata_df, 'artifactmetadata', conn)
    load_to_database(media_df, 'artifactmedia', conn)
    load_to_database(colors_df, 'artifactcolors', conn)
    
    conn.close()
    print("ETL pipeline completed successfully!")

# Run pipeline
run_full_etl_pipeline(num_pages=10)
```

### Incremental Data Loading

```python
def get_last_loaded_id(connection):
    """Get the last loaded artifact ID"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(object_id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def incremental_load():
    """Load only new artifacts since last run"""
    conn = mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )
    
    last_id = get_last_loaded_id(conn)
    
    # Fetch artifacts with ID greater than last_id
    # Process and load new data
    
    conn.close()
```

## Configuration

### Environment Variables

```bash
# Required
HARVARD_API_KEY=your_api_key_from_harvard
MYSQL_HOST=localhost_or_cloud_host
MYSQL_USER=your_database_user
MYSQL_PASSWORD=your_database_password
MYSQL_DATABASE=harvard_artifacts

# Optional
API_PAGE_SIZE=100
API_RATE_LIMIT=5  # requests per second
```

### TiDB Cloud Setup (Alternative to MySQL)

```python
# TiDB Cloud connection
conn = mysql.connector.connect(
    host=os.getenv('TIDB_HOST'),
    port=int(os.getenv('TIDB_PORT', 4000)),
    user=os.getenv('TIDB_USER'),
    password=os.getenv('TIDB_PASSWORD'),
    database=os.getenv('TIDB_DATABASE'),
    ssl_ca='/path/to/ca.pem'  # For secure connection
)
```

## Troubleshooting

### API Rate Limiting

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(page, size, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            artifacts, info = fetch_artifacts(page, size)
            return artifacts, info
        except RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Errors

```python
def get_connection_with_retry():
    """Robust database connection"""
    max_attempts = 3
    for attempt in range(max_attempts):
        try:
            conn = mysql.connector.connect(
                host=os.getenv('MYSQL_HOST'),
                user=os.getenv('MYSQL_USER'),
                password=os.getenv('MYSQL_PASSWORD'),
                database=os.getenv('MYSQL_DATABASE'),
                connect_timeout=10
            )
            return conn
        except mysql.connector.Error as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            if attempt < max_attempts - 1:
                time.sleep(5)
            else:
                raise
```

### Handling Null Values

```python
def safe_transform(value, default=''):
    """Safely handle None values"""
    return value if value is not None else default

def transform_artifacts_safe(raw_data):
    """Transform with null handling"""
    transformed = []
    for artifact in raw_data:
        record = {
            'object_id': artifact.get('id', 0),
            'title': safe_transform(artifact.get('title'), 'Untitled'),
            'culture': safe_transform(artifact.get('culture'), 'Unknown'),
            # ... more fields
        }
        transformed.append(record)
    return pd.DataFrame(transformed)
```

## Advanced Usage

### Custom Analytics Query

```python
def run_custom_query(sql_query: str, connection):
    """Execute custom SQL query"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(sql_query)
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)

# Example: Complex join query
complex_query = """
    SELECT 
        a.culture,
        a.century,
        COUNT(DISTINCT a.object_id) as artifacts,
        COUNT(m.media_id) as total_images,
        AVG(m.width * m.height) as avg_image_size
    FROM artifactmetadata a
    LEFT JOIN artifactmedia m ON a.object_id = m.object_id
    WHERE a.culture IS NOT NULL
    GROUP BY a.culture, a.century
    ORDER BY artifacts DESC
    LIMIT 20
"""

results = run_custom_query(complex_query, conn)
```

This skill enables AI agents to help developers build complete data engineering pipelines using museum API data, from extraction through visualization.
