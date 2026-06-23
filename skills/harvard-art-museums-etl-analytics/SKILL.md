---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit visualization dashboards
triggers:
  - create an ETL pipeline for museum artifact data
  - build a data engineering app with Harvard Art Museums API
  - set up artifact collection analytics dashboard
  - implement SQL analytics for museum data
  - create a Streamlit app for art museum data visualization
  - build an end-to-end data pipeline with museum artifacts
  - analyze Harvard art collection data with Python
  - design a museum artifact database schema
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON data into relational SQL tables
- **Database Design**: Structures data across `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: Executes 20+ predefined analytical queries for insights
- **Visualization**: Creates interactive Plotly charts through Streamlit dashboards

Architecture: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

```bash
# Required Python packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file in your project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_db_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=your_database_name
```

Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api

### Database Setup

```sql
-- Create database
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    classification VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    accession_year INT,
    people_count INT,
    total_page_views INT,
    total_unique_page_views INT
);

-- Artifact media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_id INT,
    base_image_url VARCHAR(500),
    primary_image_url VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Example usage
artifacts, page_info = fetch_artifacts(page=1, size=50)
print(f"Total records: {page_info['totalrecords']}")
print(f"Total pages: {page_info['pages']}")
```

### 2. ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def transform_artifacts(raw_data: List[Dict]) -> tuple:
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'century': artifact.get('century'),
            'dated': artifact.get('dated'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'accession_year': artifact.get('accessionyear'),
            'people_count': artifact.get('peoplecount', 0),
            'total_page_views': artifact.get('totalpageviews', 0),
            'total_unique_page_views': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if artifact.get('primaryimageurl'):
            media = {
                'artifact_id': artifact.get('id'),
                'media_id': artifact.get('imageid'),
                'base_image_url': artifact.get('baseimageurl'),
                'primary_image_url': artifact.get('primaryimageurl')
            }
            media_list.append(media)
        
        # Extract color data
        if artifact.get('colors'):
            for color_info in artifact['colors']:
                color = {
                    'artifact_id': artifact.get('id'),
                    'color': color_info.get('color'),
                    'color_percent': color_info.get('percent')
                }
                colors_list.append(color)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### 3. Database Loading

```python
def load_to_database(df_metadata, df_media, df_colors):
    """
    Load transformed data into MySQL/TiDB database
    """
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Insert metadata (batch insert)
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, classification, century, dated, division, 
         department, accession_year, people_count, total_page_views, 
         total_unique_page_views)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    metadata_values = [tuple(row) for row in df_metadata.values]
    cursor.executemany(metadata_query, metadata_values)
    
    # Insert media data
    if not df_media.empty:
        media_query = """
            INSERT INTO artifactmedia 
            (artifact_id, media_id, base_image_url, primary_image_url)
            VALUES (%s, %s, %s, %s)
        """
        media_values = [tuple(row) for row in df_media.values]
        cursor.executemany(media_query, media_values)
    
    # Insert color data
    if not df_colors.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, color_percent)
            VALUES (%s, %s, %s)
        """
        colors_values = [tuple(row) for row in df_colors.values]
        cursor.executemany(colors_query, colors_values)
    
    conn.commit()
    cursor.close()
    conn.close()
    
    return len(df_metadata)
```

### 4. SQL Analytics Queries

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
    
    "Top Departments": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
        LIMIT 10
    """,
    
    "Most Popular Artifacts": """
        SELECT title, culture, total_page_views
        FROM artifactmetadata
        WHERE total_page_views > 0
        ORDER BY total_page_views DESC
        LIMIT 20
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as occurrences, 
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as percent_with_media
        FROM artifactmedia m
    """
}

def execute_query(query: str) -> pd.DataFrame:
    """
    Execute SQL query and return results as DataFrame
    """
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for data collection
    st.sidebar.header("Data Collection")
    
    if st.sidebar.button("Fetch New Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts, info = fetch_artifacts(page=1, size=100)
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            count = load_to_database(df_meta, df_media, df_colors)
            st.sidebar.success(f"Loaded {count} artifacts!")
    
    # Analytics section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Query"):
        query = ANALYTICS_QUERIES[query_name]
        
        # Display SQL query
        st.subheader("SQL Query")
        st.code(query, language='sql')
        
        # Execute and display results
        with st.spinner("Executing query..."):
            df_results = execute_query(query)
            
            st.subheader("Results")
            st.dataframe(df_results)
            
            # Auto-generate visualization
            if len(df_results.columns) == 2:
                fig = px.bar(
                    df_results,
                    x=df_results.columns[0],
                    y=df_results.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Full ETL Pipeline Execution

```python
def run_etl_pipeline(num_pages=5, page_size=100):
    """
    Execute complete ETL pipeline for multiple pages
    """
    total_loaded = 0
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}/{num_pages}...")
        
        # Extract
        artifacts, page_info = fetch_artifacts(page=page, size=page_size)
        
        # Transform
        df_meta, df_media, df_colors = transform_artifacts(artifacts)
        
        # Load
        count = load_to_database(df_meta, df_media, df_colors)
        total_loaded += count
        
        print(f"Loaded {count} artifacts from page {page}")
    
    print(f"\n✓ ETL Complete: {total_loaded} total artifacts loaded")
    return total_loaded

# Execute
run_etl_pipeline(num_pages=10, page_size=100)
```

### Error Handling and Retry Logic

```python
import time

def fetch_with_retry(page, size=100, max_retries=3):
    """
    Fetch artifacts with exponential backoff retry
    """
    for attempt in range(max_retries):
        try:
            artifacts, info = fetch_artifacts(page, size)
            return artifacts, info
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
                time.sleep(wait_time)
            else:
                raise e
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_rate_limit(pages, delay=1):
    """
    Fetch multiple pages with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, pages + 1):
        artifacts, info = fetch_artifacts(page=page)
        all_artifacts.extend(artifacts)
        
        if page < pages:
            time.sleep(delay)  # Wait between requests
    
    return all_artifacts
```

### Database Connection Issues

```python
def test_db_connection():
    """
    Test database connectivity
    """
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        print("✓ Database connection successful")
        conn.close()
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_transform_artifacts(raw_data):
    """
    Transform artifacts with null-safe operations
    """
    metadata_list = []
    
    for artifact in raw_data:
        metadata = {
            'id': artifact.get('id', 0),
            'title': artifact.get('title', 'Unknown')[:500],  # Truncate
            'culture': artifact.get('culture', 'Unknown')[:200],
            'classification': artifact.get('classification', 'Unclassified')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'division': artifact.get('division', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'accession_year': artifact.get('accessionyear') or None,
            'people_count': artifact.get('peoplecount', 0) or 0,
            'total_page_views': artifact.get('totalpageviews', 0) or 0,
            'total_unique_page_views': artifact.get('totaluniquepageviews', 0) or 0
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)
```

## Advanced Analytics Examples

### Complex Join Query

```python
advanced_query = """
    SELECT 
        m.culture,
        m.century,
        COUNT(DISTINCT m.id) as artifact_count,
        COUNT(DISTINCT med.media_id) as media_count,
        AVG(m.total_page_views) as avg_views
    FROM artifactmetadata m
    LEFT JOIN artifactmedia med ON m.id = med.artifact_id
    WHERE m.culture IS NOT NULL
    GROUP BY m.culture, m.century
    HAVING artifact_count > 5
    ORDER BY avg_views DESC
    LIMIT 20
"""

df_advanced = execute_query(advanced_query)
```

This skill provides comprehensive guidance for building production-ready data engineering pipelines with museum artifact data, from API integration through to interactive analytics dashboards.
