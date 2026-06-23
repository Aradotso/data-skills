---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data pipelines with the Harvard Art Museums API using Python, SQL, and Streamlit for ETL and analytics
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - set up a data engineering project with museum artifacts
  - create analytics dashboards for art museum data
  - extract and load Harvard museum data into SQL
  - build a Streamlit app for museum data analytics
  - implement ETL for Harvard Art Museums collection
  - analyze art museum artifacts with SQL queries
  - visualize museum collection data with Python
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum data into relational SQL databases
- **SQL Analytics**: Run predefined analytical queries on artifacts, media, and color data
- **Visualization**: Interactive Streamlit dashboards with Plotly charts
- **Database Design**: Properly normalized tables with foreign key relationships

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

Obtain a free API key from Harvard Art Museums: https://www.harvardartmuseums.org/collections/api

Store credentials in environment variables or `.env` file:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

The project uses MySQL or TiDB Cloud. Create the database:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    accessionyear INT,
    dated VARCHAR(255)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Core Components

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
    response.raise_for_status()
    return response.json()

def fetch_multiple_pages(num_pages=5):
    """Fetch multiple pages of artifact data"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        print(f"Fetched page {page}, total artifacts: {len(all_artifacts)}")
    
    return all_artifacts
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform raw API data into structured dataframes"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'period': artifact.get('period', '')[:255],
            'century': artifact.get('century', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'technique': artifact.get('technique', '')[:255],
            'accessionyear': artifact.get('accessionyear'),
            'dated': artifact.get('dated', '')[:255]
        })
        
        # Extract media
        if artifact.get('images'):
            for img in artifact['images']:
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'media_url': img.get('baseimageurl', '')[:1000],
                    'media_type': img.get('format', '')[:100]
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors_list.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', '')[:50],
                    'percentage': color.get('percent')
                })
    
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

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load dataframes into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Load metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, technique, accessionyear, dated)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, media_url, media_type)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, percentage)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### 4. Analytics Queries

```python
def run_analytics_query(query_name):
    """Execute predefined analytics queries"""
    queries = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
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
        
        "top_colors": """
            SELECT color, COUNT(*) as frequency, AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """,
        
        "artifacts_with_images": """
            SELECT 
                COUNT(DISTINCT am.id) as total_artifacts,
                COUNT(DISTINCT med.artifact_id) as with_images,
                ROUND(COUNT(DISTINCT med.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as percentage
            FROM artifactmetadata am
            LEFT JOIN artifactmedia med ON am.id = med.artifact_id
        """,
        
        "department_distribution": """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    cursor.execute(queries[query_name])
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("📥 Collect Artifact Data")
        
        num_pages = st.number_input("Number of pages to fetch", 1, 20, 5)
        
        if st.button("Fetch Data"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_multiple_pages(num_pages)
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                
                st.success(f"Fetched {len(metadata_df)} artifacts")
                st.dataframe(metadata_df.head())
                
                if st.button("Load to Database"):
                    load_to_database(metadata_df, media_df, colors_df)
                    st.success("Data loaded successfully!")
    
    elif page == "Analytics":
        st.header("📊 SQL Analytics")
        
        query_choice = st.selectbox(
            "Select Analysis",
            ["artifacts_by_culture", "artifacts_by_century", 
             "top_colors", "artifacts_with_images", "department_distribution"]
        )
        
        if st.button("Run Query"):
            results_df = run_analytics_query(query_choice)
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(
                    results_df, 
                    x=results_df.columns[0], 
                    y=results_df.columns[1],
                    title=f"{query_choice.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Pipeline

```python
def run_full_etl_pipeline(num_pages=5):
    """Complete ETL workflow"""
    print("Step 1: Extracting data from API...")
    raw_data = fetch_multiple_pages(num_pages)
    
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    print("Step 3: Loading data to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("ETL pipeline completed successfully!")
    return metadata_df, media_df, colors_df
```

### Incremental Data Loading

```python
def incremental_load(last_processed_page=0, batch_size=5):
    """Load new data incrementally"""
    start_page = last_processed_page + 1
    end_page = start_page + batch_size
    
    for page in range(start_page, end_page):
        artifacts = fetch_artifacts(page=page)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts['records'])
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Processed page {page}")
```

## Troubleshooting

**API Rate Limiting:**
```python
import time

def fetch_with_retry(page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
```

**Database Connection Issues:**
```python
def safe_db_operation(operation_func):
    """Wrapper for safe database operations"""
    try:
        conn = get_db_connection()
        result = operation_func(conn)
        return result
    except Error as e:
        print(f"Database error: {e}")
        return None
    finally:
        if conn.is_connected():
            conn.close()
```

**Memory Management for Large Datasets:**
```python
def batch_load(artifacts, batch_size=100):
    """Load data in batches to manage memory"""
    for i in range(0, len(artifacts), batch_size):
        batch = artifacts[i:i+batch_size]
        metadata_df, media_df, colors_df = transform_artifacts(batch)
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Loaded batch {i//batch_size + 1}")
```
