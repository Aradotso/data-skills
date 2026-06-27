---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with MySQL and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and museum data
  - connect to Harvard Art Museums API and store in MySQL
  - visualize artifact collections with Plotly
  - implement pagination for Harvard API requests
  - query museum artifact metadata with SQL
  - transform nested JSON museum data into relational tables
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline development, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application demonstrates a complete data pipeline:
- **Extract**: Fetch artifact data from Harvard Art Museums API with pagination
- **Transform**: Convert nested JSON into relational database schema
- **Load**: Batch insert into MySQL/TiDB Cloud with proper foreign keys
- **Analyze**: Execute SQL queries for insights
- **Visualize**: Create interactive dashboards with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

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

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to `.env` file

## Database Schema

The application uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    people TEXT,
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_url VARCHAR(500),
    format VARCHAR(50),
    image_id VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Code Patterns

### 1. API Data Extraction with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(num_records=100):
    """Fetch artifacts with pagination handling"""
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': API_KEY,
            'page': page,
            'size': min(size, num_records - len(artifacts))
        }
        
        response = requests.get(BASE_URL, params=params)
        response.raise_for_status()
        
        data = response.json()
        artifacts.extend(data.get('records', []))
        
        if not data.get('records') or data['info']['next'] is None:
            break
            
        page += 1
    
    return artifacts[:num_records]
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON to relational format"""
    metadata = []
    media = []
    colors = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'people': ', '.join([p.get('name', '') for p in artifact.get('people', [])]),
            'url': artifact.get('url')
        })
        
        # Extract media/images
        for img in artifact.get('images', []):
            media.append({
                'artifact_id': artifact.get('id'),
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'image_id': img.get('imageid')
            })
        
        # Extract colors
        for color_obj in artifact.get('colors', []):
            colors.append({
                'artifact_id': artifact.get('id'),
                'color': color_obj.get('color'),
                'percentage': color_obj.get('percent')
            })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media),
        pd.DataFrame(colors)
    )
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(df_metadata, df_media, df_colors):
    """Batch insert data into MySQL"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Load metadata
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, people, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Load media
        media_query = """
        INSERT INTO artifactmedia (artifact_id, base_url, format, image_id)
        VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Load colors
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, percentage)
        VALUES (%s, %s, %s)
        """
        cursor.executemany(colors_query, df_colors.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytical SQL Queries

```python
# Example queries for analytics

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
    
    "images_per_artifact": """
        SELECT 
            CASE 
                WHEN image_count = 0 THEN 'No Images'
                WHEN image_count = 1 THEN '1 Image'
                ELSE '2+ Images'
            END as image_category,
            COUNT(*) as artifact_count
        FROM (
            SELECT a.id, COUNT(m.media_id) as image_count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY a.id
        ) as counts
        GROUP BY image_category
    """,
    
    "top_colors": """
        SELECT color, 
               COUNT(*) as frequency,
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "department_distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """
}

def run_query(query_name):
    """Execute analytical query and return DataFrame"""
    conn = get_db_connection()
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_option = st.sidebar.selectbox(
        "Choose Query",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df = run_query(query_option)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) >= 2:
                fig = px.bar(
                    df,
                    x=df.columns[0],
                    y=df.columns[1],
                    title=query_option.replace('_', ' ').title()
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Section
    st.sidebar.header("ETL Operations")
    num_records = st.sidebar.number_input("Records to fetch", 10, 1000, 100)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            # Extract
            raw_data = fetch_artifacts(num_records)
            st.success(f"Extracted {len(raw_data)} records")
            
            # Transform
            df_meta, df_media, df_colors = transform_artifacts(raw_data)
            st.success("Transformation complete")
            
            # Load
            load_to_database(df_meta, df_media, df_colors)
            st.success("Data loaded to database")

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Error Handling for API Requests

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
            time.sleep(wait_time)
```

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts():
    """Fetch only new artifacts since last load"""
    last_id = get_last_artifact_id()
    # Implement API filtering by ID if supported
    # Otherwise filter after fetching
```

## Troubleshooting

**API Rate Limiting**: Harvard API has rate limits. Add delays between requests:
```python
import time
time.sleep(0.5)  # 500ms between requests
```

**Database Connection Issues**: Verify environment variables and network access:
```python
try:
    conn = get_db_connection()
    print("Connection successful")
except Error as e:
    print(f"Connection failed: {e}")
```

**Memory Issues with Large Datasets**: Use batch processing:
```python
BATCH_SIZE = 1000
for i in range(0, len(df), BATCH_SIZE):
    batch = df.iloc[i:i+BATCH_SIZE]
    load_batch(batch)
```

**Null Value Handling**:
```python
df = df.fillna({
    'culture': 'Unknown',
    'century': 'Unknown',
    'classification': 'Unclassified'
})
```
