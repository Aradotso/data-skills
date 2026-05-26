---
name: harvard-artifacts-etl-analytics
description: End-to-end data engineering and analytics application for Harvard Art Museums API with ETL pipelines, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with museum artifacts
  - set up Harvard API data collection and analytics
  - implement SQL analytics dashboard for art collections
  - extract and transform Harvard museum artifact data
  - build a Streamlit app for museum data visualization
  - create an end-to-end data pipeline with Harvard Art API
  - analyze Harvard Art Museums collection with SQL
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering and analytics application that extracts artifact data from the Harvard Art Museums API, performs ETL transformations, stores data in SQL databases, and provides interactive analytics dashboards using Streamlit.

## What It Does

- **API Integration**: Collects artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts metadata, media, and color data; transforms nested JSON into relational tables
- **SQL Storage**: Stores structured data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for artifact insights
- **Visualization**: Interactive Plotly charts and Streamlit dashboards

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages**:
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

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Database Setup

The application expects three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    dated VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    primaryimageurl TEXT,
    verification VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    renditionnumber VARCHAR(50),
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

## Core Components

### API Data Collection

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
        data = response.json()
        return data.get('records', []), data.get('info', {})
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Paginated collection
def collect_all_artifacts(max_pages=10):
    """Collect artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(records)
        
        total_pages = info.get('pages', 1)
        if page >= total_pages:
            break
            
        # Rate limiting
        time.sleep(0.5)
    
    return all_artifacts
```

### ETL Transform

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifacts into metadata DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Untitled')[:500],
            'culture': artifact.get('culture', 'Unknown')[:255],
            'century': artifact.get('century', 'Unknown')[:100],
            'dated': artifact.get('dated', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'primaryimageurl': artifact.get('primaryimageurl', ''),
            'verification': artifact.get('verificationlevel', '')[:50]
        })
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """Extract media data from artifacts"""
    media = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media.append({
                'artifact_id': artifact_id,
                'baseimageurl': img.get('baseimageurl', ''),
                'format': img.get('format', ''),
                'height': img.get('height', 0),
                'width': img.get('width', 0),
                'renditionnumber': img.get('renditionnumber', '')
            })
    
    return pd.DataFrame(media)

def transform_artifact_colors(artifacts):
    """Extract color data from artifacts"""
    colors = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        color_list = artifact.get('colors', [])
        
        for color_obj in color_list:
            colors.append({
                'artifact_id': artifact_id,
                'color': color_obj.get('color', ''),
                'spectrum': color_obj.get('spectrum', ''),
                'hue': color_obj.get('hue', ''),
                'percent': color_obj.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### SQL Loading

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME'),
        port=int(os.getenv('DB_PORT', 3306))
    )

def load_metadata_to_sql(df):
    """Batch insert metadata into SQL"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, dated, classification, 
         department, division, primaryimageurl, verification)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
    
    return len(data)

def load_media_to_sql(df):
    """Batch insert media data"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia 
        (artifact_id, baseimageurl, format, height, width, renditionnumber)
        VALUES (%s, %s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
```

### Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
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
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Media Format Analysis": """
        SELECT format, COUNT(*) as count
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "High Resolution Images": """
        SELECT a.title, m.width, m.height, 
               (m.width * m.height) as total_pixels
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        WHERE m.width > 2000 AND m.height > 2000
        ORDER BY total_pixels DESC
        LIMIT 20
    """
}

def execute_analytics_query(query_name):
    """Execute predefined analytics query"""
    conn = get_db_connection()
    query = ANALYTICS_QUERIES.get(query_name)
    
    if not query:
        return None
    
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("Collect Artifact Data")
        
        num_pages = st.number_input("Number of pages to collect", 
                                     min_value=1, max_value=100, value=5)
        
        if st.button("Start ETL Pipeline"):
            with st.spinner("Collecting data..."):
                artifacts = collect_all_artifacts(max_pages=num_pages)
                st.success(f"Collected {len(artifacts)} artifacts")
                
                # Transform
                metadata_df = transform_artifact_metadata(artifacts)
                media_df = transform_artifact_media(artifacts)
                colors_df = transform_artifact_colors(artifacts)
                
                # Load
                load_metadata_to_sql(metadata_df)
                load_media_to_sql(media_df)
                
                st.success("ETL Pipeline Complete!")
    
    elif page == "Analytics":
        st.header("SQL Analytics")
        
        query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Run Query"):
            df = execute_analytics_query(query_name)
            
            if df is not None and not df.empty:
                st.dataframe(df)
                
                # Auto-visualization
                if len(df.columns) == 2:
                    fig = px.bar(df, x=df.columns[0], y=df.columns[1],
                                title=query_name)
                    st.plotly_chart(fig)
    
    elif page == "Visualizations":
        st.header("Interactive Visualizations")
        
        # Custom visualizations
        viz_type = st.selectbox("Visualization Type", 
                               ["Culture Distribution", "Color Analysis", 
                                "Department Breakdown"])
        
        if viz_type == "Culture Distribution":
            df = execute_analytics_query("Artifacts by Culture")
            fig = px.pie(df, values='count', names='culture',
                        title='Artifact Distribution by Culture')
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_full_etl_pipeline(max_pages=10):
    """Complete ETL pipeline execution"""
    # Extract
    print("Extracting data from API...")
    artifacts = collect_all_artifacts(max_pages=max_pages)
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading to database...")
    metadata_count = load_metadata_to_sql(metadata_df)
    media_count = load_media_to_sql(media_df)
    
    print(f"Pipeline complete: {metadata_count} metadata, {media_count} media records")
    
    return {
        'metadata': metadata_count,
        'media': media_count,
        'artifacts': artifacts
    }
```

### Error Handling

```python
def safe_etl_execution():
    """ETL with error handling and retry logic"""
    max_retries = 3
    
    for attempt in range(max_retries):
        try:
            artifacts = collect_all_artifacts(max_pages=5)
            metadata_df = transform_artifact_metadata(artifacts)
            load_metadata_to_sql(metadata_df)
            return True
        except requests.RequestException as e:
            print(f"API error (attempt {attempt + 1}): {e}")
            time.sleep(2 ** attempt)  # Exponential backoff
        except Error as e:
            print(f"Database error: {e}")
            return False
    
    return False
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

**API Rate Limiting**:
```python
# Add delays between requests
import time
time.sleep(0.5)  # 500ms delay
```

**Database Connection Issues**:
```python
# Test connection
try:
    conn = get_db_connection()
    print("Connection successful")
    conn.close()
except Error as e:
    print(f"Connection failed: {e}")
```

**Missing API Key**:
- Ensure `.env` file exists with `HARVARD_API_KEY`
- Get API key from: https://www.harvardartmuseums.org/collections/api

**Large Dataset Memory Issues**:
```python
# Process in smaller batches
def batch_process(max_pages, batch_size=2):
    for start_page in range(1, max_pages, batch_size):
        end_page = min(start_page + batch_size, max_pages)
        artifacts = collect_all_artifacts_range(start_page, end_page)
        # Process batch
```
