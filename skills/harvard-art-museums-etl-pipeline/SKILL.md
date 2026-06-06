---
name: harvard-art-museums-etl-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering workflow for museum artifacts
  - set up Harvard Art Museums data collection and analytics
  - build a Streamlit dashboard for art museum data
  - implement SQL analytics on Harvard artifacts collection
  - extract and transform Harvard Art Museums API data
  - create visualizations from museum artifact metadata
  - design relational database schema for art collection data
---

# Harvard Art Museums ETL Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill teaches AI agents how to build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytical queries, and interactive Streamlit dashboards for artifact metadata.

## What This Project Does

The Harvard Artifacts Collection application provides a complete data engineering workflow:

- **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transform**: Convert nested JSON into normalized relational tables
- **Load**: Batch insert structured data into MySQL/TiDB Cloud
- **Analyze**: Execute predefined SQL queries for insights
- **Visualize**: Display results through interactive Plotly charts in Streamlit

Architecture: `API → ETL → SQL → Analytics → Visualization`

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

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

Get your API key from: https://www.harvardartmuseums.org/collections/api

### Database Setup

Create the required MySQL/TiDB database and tables:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    period VARCHAR(200),
    classification VARCHAR(200),
    medium VARCHAR(500),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(300),
    description TEXT,
    provenance TEXT,
    copyright VARCHAR(500),
    accession_number VARCHAR(100),
    url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    thumbnail_url VARCHAR(1000),
    height INT,
    width INT,
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

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will open at `http://localhost:8501`

## Key Code Patterns

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
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
def collect_artifacts(num_pages=10):
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        records, info = fetch_artifacts(page=page)
        all_artifacts.extend(records)
        print(f"Fetched page {page}/{info['pages']}")
    
    return all_artifacts
```

### 2. ETL Transformation

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact JSON into structured dataframe"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', ''),
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'dated': artifact.get('dated', ''),
            'period': artifact.get('period', ''),
            'classification': artifact.get('classification', ''),
            'medium': artifact.get('medium', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'technique': artifact.get('technique', ''),
            'description': artifact.get('description', ''),
            'provenance': artifact.get('provenance', ''),
            'copyright': artifact.get('copyright', ''),
            'accession_number': artifact.get('accessionyear', ''),
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image information"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl', ''),
                'thumbnail_url': img.get('thumbnailurl', ''),
                'height': img.get('height'),
                'width': img.get('width'),
                'format': img.get('format', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color palette information"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('color', ''),
                'percentage': color.get('percent', 0.0)
            }
            color_list.append(color_data)
    
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
        port=os.getenv('DB_PORT'),
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
    (id, title, culture, century, dated, period, classification, 
     medium, department, division, technique, description, 
     provenance, copyright, accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data = df_metadata.values.tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return len(data)

def load_media_to_db(df_media):
    """Load media information"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, image_url, thumbnail_url, height, width, format)
    VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data = df_media.values.tolist()
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return len(data)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics")
    st.markdown("End-to-end ETL Pipeline and Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Navigation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection()
    elif page == "SQL Analytics":
        show_sql_analytics()
    else:
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", 1, 100, 5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = collect_artifacts(num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            count_meta = load_metadata_to_db(df_metadata)
            count_media = load_media_to_db(df_media)
            count_colors = load_colors_to_db(df_colors)
            st.success(f"Loaded {count_meta} metadata, {count_media} media, {count_colors} colors")

def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Top 10 Cultures": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        "Department Distribution": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            GROUP BY department
            ORDER BY artifact_count DESC
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        conn = get_db_connection()
        df_result = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df_result)
        
        # Auto-generate visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

### 5. Advanced Analytics Query

```python
def execute_custom_query(query):
    """Execute custom SQL query and return results"""
    try:
        conn = get_db_connection()
        df = pd.read_sql(query, conn)
        conn.close()
        return df, None
    except Error as e:
        return None, str(e)

# Example: Color analysis
color_query = """
SELECT 
    c.color_name,
    COUNT(DISTINCT c.artifact_id) as artifact_count,
    AVG(c.percentage) as avg_percentage
FROM artifactcolors c
JOIN artifactmetadata m ON c.artifact_id = m.id
WHERE c.color_name != ''
GROUP BY c.color_name
ORDER BY artifact_count DESC
LIMIT 15
"""

df_colors, error = execute_custom_query(color_query)

if df_colors is not None:
    fig = px.bar(
        df_colors,
        x='color_name',
        y='artifact_count',
        title='Most Common Colors in Art Collection',
        color='avg_percentage',
        color_continuous_scale='viridis'
    )
    st.plotly_chart(fig)
```

## Common Analytical Queries

### Artifacts with Images
```sql
SELECT 
    m.title,
    m.culture,
    m.century,
    COUNT(med.media_id) as image_count
FROM artifactmetadata m
LEFT JOIN artifactmedia med ON m.id = med.artifact_id
GROUP BY m.id
HAVING image_count > 0
ORDER BY image_count DESC;
```

### Classification Analysis
```sql
SELECT 
    classification,
    COUNT(*) as count,
    COUNT(DISTINCT culture) as unique_cultures
FROM artifactmetadata
WHERE classification != ''
GROUP BY classification
ORDER BY count DESC;
```

### Period and Technique Correlation
```sql
SELECT 
    period,
    technique,
    COUNT(*) as artifact_count
FROM artifactmetadata
WHERE period != '' AND technique != ''
GROUP BY period, technique
ORDER BY artifact_count DESC
LIMIT 20;
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_with_retry(page, max_retries=3):
    for attempt in range(max_retries):
        try:
            records, info = fetch_artifacts(page=page)
            return records, info
        except Exception as e:
            if "429" in str(e):  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Pooling
```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_pooled_connection():
    return db_pool.get_connection()
```

### Handling Missing Data
```python
def safe_get(dictionary, key, default=''):
    """Safely extract value from nested dictionary"""
    try:
        value = dictionary.get(key, default)
        return value if value is not None else default
    except:
        return default

# Use in transformation
metadata = {
    'title': safe_get(artifact, 'title'),
    'culture': safe_get(artifact, 'culture'),
    # ...
}
```

## Best Practices

1. **Incremental Loading**: Only fetch new artifacts by tracking last processed ID
2. **Batch Processing**: Insert data in batches of 1000 records for performance
3. **Error Logging**: Log all ETL errors to a separate table for debugging
4. **Data Validation**: Validate data types and constraints before loading
5. **Caching**: Cache API responses to avoid redundant calls during development

This skill enables AI agents to help developers build production-ready data engineering pipelines for museum collections and similar cultural heritage APIs.
