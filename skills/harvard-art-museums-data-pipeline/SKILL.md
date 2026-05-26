---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, MySQL, and Streamlit
triggers:
  - how do I use the Harvard Art Museums API for data engineering
  - build an ETL pipeline with Harvard artifacts data
  - create analytics dashboard for museum collection data
  - connect to Harvard Art Museums API and store in SQL
  - extract and transform Harvard museum artifacts data
  - build Streamlit app for art collection analytics
  - query and visualize Harvard art museums data
  - set up data pipeline for museum collection
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL operations, SQL database design, analytics queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: 20+ predefined analytical queries for insights on artifacts, cultures, media, and colors
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time data exploration

**Architecture**: `API → ETL → SQL → Analytics → Visualization`

## Installation

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

## Configuration

### 1. Harvard Art Museums API Key

Get your API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api).

### 2. Database Setup

The project supports MySQL or TiDB Cloud. Configure connection parameters:

```python
# Database configuration (use environment variables)
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}
```

### 3. Environment Variables

Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

## Database Schema

The ETL pipeline creates three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    division VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    period VARCHAR(255),
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    imageurl VARCHAR(1000),
    height INT,
    width INT,
    format VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Usage Patterns

### 1. Fetching Data from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'fields': 'id,title,culture,century,classification,division,department,dated,medium,technique,period,totalpageviews,totaluniquepageviews,images,colors'
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data['records']
total_records = data['info']['totalrecords']
```

### 2. ETL Pipeline - Transform and Load

```python
import pandas as pd
import mysql.connector

def transform_artifacts(raw_data):
    """
    Transform nested JSON to flat structure for SQL insertion
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'division': artifact.get('division', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'period': artifact.get('period', '')[:255],
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'artifact_id': artifact['id'],
                    'imageurl': img.get('baseimageurl', '')[:1000],
                    'height': img.get('height'),
                    'width': img.get('width'),
                    'format': img.get('format', '')[:50]
                }
                media_list.append(media)
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = {
                    'artifact_id': artifact['id'],
                    'color': color.get('color', '')[:50],
                    'spectrum': color.get('spectrum', '')[:50],
                    'hue': color.get('hue', '')[:50],
                    'percent': color.get('percent', 0.0)
                }
                colors_list.append(color_data)
    
    return metadata_list, media_list, colors_list

def load_to_database(metadata, media, colors, db_config):
    """
    Batch insert data into MySQL database
    """
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor()
    
    # Insert metadata
    metadata_query = """
    INSERT IGNORE INTO artifactmetadata 
    (id, title, culture, century, classification, division, department, 
     dated, medium, technique, period, totalpageviews, totaluniquepageviews)
    VALUES (%(id)s, %(title)s, %(culture)s, %(century)s, %(classification)s,
            %(division)s, %(department)s, %(dated)s, %(medium)s, %(technique)s,
            %(period)s, %(totalpageviews)s, %(totaluniquepageviews)s)
    """
    cursor.executemany(metadata_query, metadata)
    
    # Insert media
    if media:
        media_query = """
        INSERT INTO artifactmedia (artifact_id, imageurl, height, width, format)
        VALUES (%(artifact_id)s, %(imageurl)s, %(height)s, %(width)s, %(format)s)
        """
        cursor.executemany(media_query, media)
    
    # Insert colors
    if colors:
        colors_query = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%(artifact_id)s, %(color)s, %(spectrum)s, %(hue)s, %(percent)s)
        """
        cursor.executemany(colors_query, colors)
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 3. Analytical SQL Queries

```python
def run_analytics_query(query_name, db_config):
    """
    Execute predefined analytical queries
    """
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'top_viewed_artifacts': """
            SELECT title, culture, century, totalpageviews
            FROM artifactmetadata
            ORDER BY totalpageviews DESC
            LIMIT 20
        """,
        
        'media_availability': """
            SELECT 
                CASE WHEN EXISTS (
                    SELECT 1 FROM artifactmedia m WHERE m.artifact_id = a.id
                ) THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            GROUP BY media_status
        """,
        
        'color_distribution': """
            SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY spectrum
            ORDER BY count DESC
        """
    }
    
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df
```

### 4. Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for API configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # ETL Section
    st.header("📥 Data Collection")
    num_records = st.number_input("Number of records to fetch", 
                                   min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching artifacts..."):
            data = fetch_artifacts(api_key, page=1, size=num_records)
            metadata, media, colors = transform_artifacts(data['records'])
            load_to_database(metadata, media, colors, DB_CONFIG)
            st.success(f"Loaded {len(metadata)} artifacts successfully!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_options = {
        "Artifacts by Culture": "artifacts_by_culture",
        "Artifacts by Century": "artifacts_by_century",
        "Top Viewed Artifacts": "top_viewed_artifacts",
        "Media Availability": "media_availability",
        "Color Distribution": "color_distribution"
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Analysis"):
        df = run_analytics_query(query_options[selected_query], DB_CONFIG)
        
        # Display table
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df) > 0 and len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access the dashboard at http://localhost:8501
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(api_key, max_records=1000):
    """
    Fetch multiple pages of artifacts
    """
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        all_artifacts.extend(data['records'])
        
        if len(data['records']) < size:
            break  # No more pages
        
        page += 1
        time.sleep(0.5)  # Rate limiting
    
    return all_artifacts[:max_records]
```

### Error Handling in ETL

```python
def safe_etl_pipeline(api_key, db_config, num_records=100):
    """
    ETL pipeline with error handling
    """
    try:
        raw_data = fetch_artifacts(api_key, size=num_records)
        metadata, media, colors = transform_artifacts(raw_data['records'])
        load_to_database(metadata, media, colors, db_config)
        return True, f"Successfully loaded {len(metadata)} artifacts"
    except requests.RequestException as e:
        return False, f"API Error: {str(e)}"
    except mysql.connector.Error as e:
        return False, f"Database Error: {str(e)}"
    except Exception as e:
        return False, f"Unexpected Error: {str(e)}"
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """
    Fetch with exponential backoff on rate limit
    """
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except Exception as e:
            if "429" in str(e):  # Too Many Requests
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_database_connection(db_config):
    """
    Test database connectivity before ETL
    """
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True, "Connection successful"
    except mysql.connector.Error as e:
        return False, f"Connection failed: {str(e)}"
```

### Memory Management for Large Datasets

```python
def batch_load_artifacts(api_key, db_config, total_records=10000, batch_size=100):
    """
    Load large datasets in batches to manage memory
    """
    num_batches = (total_records + batch_size - 1) // batch_size
    
    for batch in range(num_batches):
        page = batch + 1
        data = fetch_artifacts(api_key, page=page, size=batch_size)
        metadata, media, colors = transform_artifacts(data['records'])
        load_to_database(metadata, media, colors, db_config)
        time.sleep(1)  # Rate limiting
```

This skill enables AI agents to guide developers through building complete data engineering pipelines with the Harvard Art Museums API, from initial data collection through visualization.
