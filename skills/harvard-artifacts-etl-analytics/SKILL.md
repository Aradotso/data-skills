---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - set up analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums API data
  - build streamlit app for Harvard artifacts visualization
  - query and analyze Harvard museum collection data
  - implement SQL analytics for art museum metadata
  - create data pipeline from Harvard API to database
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipeline construction, SQL database design, analytical querying, and interactive visualization using Streamlit.

## What This Project Does

- **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Parse nested JSON into normalized relational structures
- **Load**: Batch insert data into MySQL/TiDB databases with proper schema design
- **Analyze**: Execute analytical SQL queries on artifact metadata, media, and color data
- **Visualize**: Display results through interactive Plotly charts in Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Install core packages if requirements.txt is unavailable
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
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to `.env` file

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    accessionyear INT,
    PRIMARY KEY (id)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Usage Patterns

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
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
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch first 100 artifacts
data = fetch_artifacts(page=1, size=100)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform API response to metadata DataFrame"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'accessionyear': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Transform media/image data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_list.append({
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl', '')[:500],
                'iiifbaseuri': image.get('iiifbaseuri', '')[:500]
            })
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Transform color data"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(color_list)
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata_to_db(df_metadata):
    """Batch insert metadata into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, dated, accessionyear)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = [tuple(x) for x in df_metadata.to_numpy()]
    
    try:
        cursor.executemany(insert_query, data_tuples)
        conn.commit()
        print(f"Inserted {cursor.rowcount} rows")
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()

def load_media_to_db(df_media):
    """Load media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(x) for x in df_media.to_numpy()]
    cursor.executemany(insert_query, data_tuples)
    conn.commit()
    cursor.close()
    conn.close()
```

### 4. Complete ETL Pipeline

```python
def run_etl_pipeline(num_pages=5, page_size=100):
    """Execute complete ETL pipeline"""
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        
        # Extract
        data = fetch_artifacts(page=page, size=page_size)
        artifacts = data['records']
        
        # Transform
        df_metadata = transform_artifact_metadata(artifacts)
        df_media = transform_artifact_media(artifacts)
        df_colors = transform_artifact_colors(artifacts)
        
        all_metadata.append(df_metadata)
        all_media.append(df_media)
        all_colors.append(df_colors)
    
    # Combine all pages
    final_metadata = pd.concat(all_metadata, ignore_index=True)
    final_media = pd.concat(all_media, ignore_index=True)
    final_colors = pd.concat(all_colors, ignore_index=True)
    
    # Load
    load_metadata_to_db(final_metadata)
    load_media_to_db(final_media)
    load_colors_to_db(final_colors)
    
    print("ETL pipeline completed!")
    return final_metadata, final_media, final_colors
```

### 5. SQL Analytics Queries

```python
def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 20
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL THEN 'Has Media' ELSE 'No Media' END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """,
    
    "Top Colors": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

# Execute a query
results = execute_query(ANALYTICS_QUERIES["Artifacts by Culture"])
print(results)
```

### 6. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_name = st.sidebar.selectbox(
        "Choose Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute selected query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = execute_query(ANALYTICS_QUERIES[query_name])
            
            # Display results
            st.subheader(f"Results: {query_name}")
            st.dataframe(df)
            
            # Visualize if appropriate
            if len(df.columns) >= 2:
                fig = px.bar(
                    df.head(15),
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Controls
    st.sidebar.header("ETL Pipeline")
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            run_etl_pipeline(num_pages=3, page_size=50)
            st.success("ETL completed successfully!")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(max_records=500):
    """Fetch artifacts with pagination"""
    page_size = 100
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(page=page, size=page_size)
        artifacts = data['records']
        
        if not artifacts:
            break
            
        all_artifacts.extend(artifacts)
        page += 1
    
    return all_artifacts[:max_records]
```

### Error Handling in ETL

```python
def safe_etl_pipeline(num_pages=5):
    """ETL pipeline with error handling"""
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(page=page)
            artifacts = data['records']
            
            df_metadata = transform_artifact_metadata(artifacts)
            load_metadata_to_db(df_metadata)
            
        except requests.exceptions.RequestException as e:
            print(f"API error on page {page}: {e}")
            continue
        except Error as e:
            print(f"Database error on page {page}: {e}")
            continue
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(page, size=100, delay=1):
    """Fetch with rate limiting"""
    time.sleep(delay)  # Wait between requests
    return fetch_artifacts(page, size)
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Error as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Missing API Key

```python
def validate_config():
    """Validate required configuration"""
    api_key = os.getenv('HARVARD_API_KEY')
    if not api_key:
        raise ValueError("HARVARD_API_KEY not found in environment variables")
    
    required_vars = ['DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
    missing = [var for var in required_vars if not os.getenv(var)]
    
    if missing:
        raise ValueError(f"Missing required environment variables: {', '.join(missing)}")
```

## Advanced Features

### Incremental Loading

```python
def get_latest_artifact_id():
    """Get the most recent artifact ID in database"""
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    df = execute_query(query)
    return df['max_id'].iloc[0] or 0

def incremental_etl():
    """Load only new artifacts"""
    latest_id = get_latest_artifact_id()
    
    # Fetch artifacts with ID > latest_id
    # Implement filtering logic based on API capabilities
    pass
```

This skill enables AI coding agents to help developers build complete data engineering pipelines using the Harvard Art Museums API, from extraction through visualization.
