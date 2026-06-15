---
name: harvard-artifacts-etl-streamlit-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - build a data pipeline for Harvard Art Museums API
  - create ETL pipeline with Streamlit dashboard
  - analyze Harvard artifacts data with SQL
  - setup museum data engineering project
  - visualize art collection data with Plotly
  - build analytics app from API to database
  - create museum artifact data warehouse
  - implement end-to-end data pipeline with visualization
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection application provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Structured storage with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

Architecture flow: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites
- Python 3.8+
- MySQL or TiDB Cloud database
- Harvard Art Museums API key (get from https://harvardartmuseums.org/collections/api)

### Setup Steps

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Configure environment variables
# Create .env file with:
# HARVARD_API_KEY=your_api_key_here
# DB_HOST=your_database_host
# DB_USER=your_database_user
# DB_PASSWORD=your_database_password
# DB_NAME=harvard_artifacts

# Run the Streamlit application
streamlit run app.py
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

### Environment Variables

```python
import os
from dotenv import load_dotenv

load_dotenv()

# API Configuration
HARVARD_API_KEY = os.getenv('HARVARD_API_KEY')
API_BASE_URL = 'https://api.harvardartmuseums.org/object'

# Database Configuration
DB_CONFIG = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}
```

### Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    period VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    provenance TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    image_height INT,
    image_width INT,
    format VARCHAR(50),
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

## Key Components & Usage

### 1. API Data Extraction

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    url = f'https://api.harvardartmuseums.org/object'
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        data = response.json()
        
        # Rate limiting
        time.sleep(0.5)
        
        return data.get('records', []), data.get('info', {})
    except requests.exceptions.RequestException as e:
        print(f"API Error: {e}")
        return [], {}

# Usage
artifacts, info = fetch_artifacts(HARVARD_API_KEY, page=1, size=50)
print(f"Fetched {len(artifacts)} artifacts")
print(f"Total available: {info.get('totalrecords', 0)}")
```

### 2. ETL Transform Function

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform nested JSON into normalized dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'technique': artifact.get('technique', 'Unknown')[:300],
            'period': artifact.get('period', 'Unknown')[:200],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'description': artifact.get('description', '')[:5000],
            'provenance': artifact.get('provenance', '')[:5000]
        }
        metadata_list.append(metadata)
        
        # Extract media
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': img.get('baseimageurl', '')[:500],
                'image_height': img.get('height', 0),
                'image_width': img.get('width', 0),
                'format': img.get('format', '')[:50]
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', '')[:10],
                'color_name': color.get('color', '')[:100],
                'percentage': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(config):
    """Create MySQL database connection"""
    try:
        connection = mysql.connector.connect(**config)
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Batch insert dataframes into SQL tables"""
    connection = create_database_connection(db_config)
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
        INSERT IGNORE INTO artifactmetadata 
        (id, title, culture, century, classification, department, technique, period, dated, description, provenance)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        if not media_df.empty:
            media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, image_url, image_height, image_width, format)
            VALUES (%s, %s, %s, %s, %s)
            """
            media_values = media_df.values.tolist()
            cursor.executemany(media_query, media_values)
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_name, percentage)
            VALUES (%s, %s, %s, %s)
            """
            colors_values = colors_df.values.tolist()
            cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        return True
        
    except Error as e:
        print(f"Database insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

### 4. Analytical SQL Queries

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Top Departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT color_name, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 20
    """,
    
    "Artifacts with Multiple Images": """
        SELECT m.title, m.culture, COUNT(a.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        GROUP BY m.id, m.title, m.culture
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Media Format Distribution": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        WHERE format != ''
        GROUP BY format
        ORDER BY count DESC
    """
}

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    connection = create_database_connection(db_config)
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("📥 ETL Pipeline - Collect Data")
        
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
        
        if st.button("Start ETL Process"):
            with st.spinner("Fetching data from API..."):
                all_artifacts = []
                for page_num in range(1, num_pages + 1):
                    artifacts, _ = fetch_artifacts(HARVARD_API_KEY, page=page_num)
                    all_artifacts.extend(artifacts)
                    st.write(f"Fetched page {page_num}")
                
                st.success(f"Collected {len(all_artifacts)} artifacts")
                
                # Transform
                metadata_df, media_df, colors_df = transform_artifact_data(all_artifacts)
                st.write(f"Transformed into {len(metadata_df)} metadata records")
                
                # Load
                success = load_to_database(metadata_df, media_df, colors_df, DB_CONFIG)
                if success:
                    st.success("✅ Data loaded successfully!")
    
    elif page == "SQL Analytics":
        st.header("📊 SQL Query Analytics")
        
        query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
        
        if st.button("Run Query"):
            query = ANALYTICAL_QUERIES[query_name]
            
            with st.spinner("Executing query..."):
                results_df = execute_query(query, DB_CONFIG)
                
                if results_df is not None and not results_df.empty:
                    st.subheader("Query Results")
                    st.dataframe(results_df)
                    
                    # Auto-generate visualization
                    if len(results_df.columns) >= 2:
                        fig = px.bar(
                            results_df,
                            x=results_df.columns[0],
                            y=results_df.columns[1],
                            title=query_name
                        )
                        st.plotly_chart(fig, use_container_width=True)
                else:
                    st.warning("No results found")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(api_key, db_config, num_pages=10):
    """Complete ETL pipeline execution"""
    print("Starting ETL Pipeline...")
    
    # Extract
    all_artifacts = []
    for page in range(1, num_pages + 1):
        artifacts, info = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(artifacts)
        print(f"Extracted page {page}/{num_pages}")
    
    # Transform
    metadata_df, media_df, colors_df = transform_artifact_data(all_artifacts)
    print(f"Transformed {len(metadata_df)} artifacts")
    
    # Load
    success = load_to_database(metadata_df, media_df, colors_df, db_config)
    
    if success:
        print("ETL Pipeline completed successfully!")
        return True
    else:
        print("ETL Pipeline failed")
        return False
```

### Incremental Data Loading

```python
def get_max_artifact_id(db_config):
    """Get the highest artifact ID already in database"""
    connection = create_database_connection(db_config)
    if not connection:
        return 0
    
    cursor = connection.cursor()
    cursor.execute("SELECT COALESCE(MAX(id), 0) FROM artifactmetadata")
    max_id = cursor.fetchone()[0]
    cursor.close()
    connection.close()
    
    return max_id

def incremental_etl(api_key, db_config):
    """Load only new artifacts not already in database"""
    max_id = get_max_artifact_id(db_config)
    print(f"Latest artifact ID in DB: {max_id}")
    
    # Fetch and filter new artifacts
    artifacts, _ = fetch_artifacts(api_key, page=1, size=100)
    new_artifacts = [a for a in artifacts if a.get('id', 0) > max_id]
    
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifact_data(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df, db_config)
        print(f"Loaded {len(new_artifacts)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Rate Limiting
```python
# Add exponential backoff for rate limits
import time
from requests.exceptions import HTTPError

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            artifacts, info = fetch_artifacts(api_key, page)
            return artifacts, info
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return [], {}
```

### Database Connection Issues
```python
# Test database connection
def test_db_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✅ Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Missing Data Handling
```python
# Safe data extraction with defaults
def safe_get(data, key, default='Unknown', max_length=None):
    value = data.get(key, default)
    if value is None:
        value = default
    if max_length and isinstance(value, str):
        value = value[:max_length]
    return value
```

### Memory Management for Large Datasets
```python
# Process in batches
def batch_etl(api_key, db_config, total_pages=100, batch_size=10):
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        print(f"Processing batch: pages {batch_start}-{batch_end-1}")
        
        run_etl_pipeline(api_key, db_config, num_pages=batch_size)
        
        # Clear memory
        import gc
        gc.collect()
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit and SQL databases.
