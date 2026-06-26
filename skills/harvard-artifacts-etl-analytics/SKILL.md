---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API data with MySQL and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create analytics dashboard for museum artifact data
  - set up Harvard artifacts data engineering project
  - query and visualize Harvard museum collection data
  - build Streamlit app for art museum analytics
  - extract transform load Harvard art data
  - create SQL database for museum artifacts
  - analyze Harvard museum collection with Python
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

The Harvard Artifacts Collection project provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color information
- **SQL Database**: Structured relational schema with foreign key relationships
- **Analytics Queries**: 20+ predefined SQL analytics queries for insights
- **Visualization**: Interactive Streamlit dashboards with Plotly charts

## Installation

### Prerequisites

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
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

### Environment Setup

Create a `.env` file in the project root:

```bash
# Harvard API Configuration
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema

### Table Structure

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(200),
    dimensions VARCHAR(500),
    url TEXT,
    credit_line TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        api_key: Harvard API key
        page: Page number (default 1)
        size: Records per page (max 100)
    
    Returns:
        JSON response with artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)

print(f"Total records: {data['info']['totalrecords']}")
print(f"Pages available: {data['info']['pages']}")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def extract_metadata(artifacts):
    """Extract artifact metadata from API response"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'technique': artifact.get('technique', 'Unknown')[:500],
            'medium': artifact.get('medium', 'Unknown')[:500],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'dimensions': artifact.get('dimensions', 'Unknown')[:500],
            'url': artifact.get('url'),
            'credit_line': artifact.get('creditline')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)

def extract_media(artifacts):
    """Extract media/image data from artifacts"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'media_type': 'image'
            })
    
    return pd.DataFrame(media_records)

def extract_colors(artifacts):
    """Extract color data from artifacts"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    return pd.DataFrame(color_records)

# Complete ETL workflow
def run_etl_pipeline(api_key, num_pages=5):
    """Run complete ETL pipeline"""
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        response = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(response['records'])
    
    # Transform
    metadata_df = extract_metadata(all_artifacts)
    media_df = extract_media(all_artifacts)
    colors_df = extract_colors(all_artifacts)
    
    return metadata_df, media_df, colors_df
```

### 3. Database Loading

```python
def get_db_connection():
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    conn = get_db_connection()
    if not conn:
        return False
    
    cursor = conn.cursor()
    
    try:
        # Load metadata (batch insert)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (artifact_id, title, culture, century, classification, department, 
         technique, medium, dated, dimensions, url, credit_line)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        return True
        
    except Error as e:
        print(f"Database load error: {e}")
        conn.rollback()
        return False
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY culture
        ORDER BY artifact_count DESC
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
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors Used": """
        SELECT color_hex, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "Artifacts with Most Images": """
        SELECT am.artifact_id, am.title, COUNT(media.media_id) as image_count
        FROM artifactmetadata am
        JOIN artifactmedia media ON am.artifact_id = media.artifact_id
        GROUP BY am.artifact_id, am.title
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_analytics_query(query_name):
    """Execute analytics query and return results"""
    conn = get_db_connection()
    if not conn:
        return None
    
    query = ANALYTICS_QUERIES.get(query_name)
    if not query:
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

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # ETL Section
    st.header("📥 Data Collection")
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching and processing data..."):
            api_key = os.getenv('HARVARD_API_KEY')
            metadata_df, media_df, colors_df = run_etl_pipeline(api_key, num_pages)
            
            success = load_to_database(metadata_df, media_df, colors_df)
            
            if success:
                st.success(f"✅ Loaded {len(metadata_df)} artifacts successfully!")
            else:
                st.error("❌ ETL pipeline failed")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_options = list(ANALYTICS_QUERIES.keys())
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            result_df = execute_analytics_query(selected_query)
            
            if result_df is not None and not result_df.empty:
                st.dataframe(result_df)
                
                # Auto-generate visualization
                if len(result_df.columns) >= 2:
                    fig = px.bar(
                        result_df,
                        x=result_df.columns[0],
                        y=result_df.columns[1],
                        title=selected_query
                    )
                    st.plotly_chart(fig, use_container_width=True)
            else:
                st.warning("No data returned from query")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact_id already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    conn.close()
    return result[0] if result[0] else 0

def incremental_etl(api_key):
    """Only fetch new artifacts since last run"""
    last_id = get_last_artifact_id()
    # Implement logic to fetch only newer artifacts
    pass
```

### Error Handling and Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def safe_api_call(api_key, page, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.exceptions.RequestException as e:
            logger.warning(f"Attempt {attempt + 1} failed: {e}")
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(api_key, num_pages, delay=1):
    """Fetch data with rate limiting"""
    all_data = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page)
        all_data.extend(data['records'])
        time.sleep(delay)  # Prevent rate limiting
    return all_data
```

### Database Connection Issues

```python
# Verify connection
def test_db_connection():
    conn = get_db_connection()
    if conn and conn.is_connected():
        print("✅ Database connected successfully")
        conn.close()
        return True
    else:
        print("❌ Database connection failed")
        return False
```

### Missing API Key

```python
def validate_env_vars():
    """Validate required environment variables"""
    required = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD']
    missing = [var for var in required if not os.getenv(var)]
    
    if missing:
        raise EnvironmentError(f"Missing environment variables: {', '.join(missing)}")
```

## Best Practices

1. **Batch Processing**: Use `executemany()` for bulk inserts (faster than row-by-row)
2. **Data Validation**: Clean and validate data before loading to database
3. **Connection Pooling**: Reuse database connections for better performance
4. **Error Logging**: Log all ETL errors for debugging and monitoring
5. **Incremental Loads**: Implement upsert logic to avoid duplicate data
6. **API Pagination**: Always handle pagination for large datasets
