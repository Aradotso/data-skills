---
name: harvard-artifacts-data-engineering-pipeline
description: Build end-to-end data engineering pipelines with Harvard Art Museums API, ETL processes, SQL analytics, and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - create ETL pipeline with Harvard Art Museums API
  - set up artifact collection analytics dashboard
  - extract and analyze museum data from Harvard API
  - build Streamlit app with museum artifacts
  - implement data engineering workflow for art collections
  - create SQL analytics for Harvard Art Museums data
  - visualize museum artifact data with interactive dashboards
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for collecting, transforming, storing, and visualizing artifact data from the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive dashboards using Streamlit and Plotly.

**Architecture Flow:** API → ETL → SQL Database → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required dependencies:**
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

1. Get your Harvard Art Museums API key from: https://harvardartmuseums.org/collections/api
2. Create a `.env` file or configure via Streamlit sidebar:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=your_database_name
```

### Database Setup

The application uses MySQL/TiDB Cloud with three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    division VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    classification VARCHAR(255),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    image_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    color_id INT PRIMARY KEY AUTO_INCREMENT,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The Streamlit app will open in your browser with options to:
- Configure API and database credentials
- Collect artifacts from Harvard API
- Run ETL pipeline
- Execute analytical SQL queries
- Visualize results

## Key Components

### 1. API Data Collection

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_artifacts=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # API page size
    
    while len(artifacts) < num_artifacts:
        params = {
            "apikey": api_key,
            "page": page,
            "size": min(size, num_artifacts - len(artifacts))
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get("records", []))
            
            if len(data.get("records", [])) == 0:
                break
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]
```

### 2. ETL Pipeline

#### Extract & Transform

```python
def transform_artifacts(raw_artifacts):
    """Transform nested JSON into relational data structures"""
    metadata = []
    media_data = []
    color_data = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'division': artifact.get('division', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'medium': artifact.get('medium', 'Unknown'),
            'dimensions': artifact.get('dimensions', 'Unknown'),
            'url': artifact.get('url', '')
        })
        
        # Extract media information
        images = artifact.get('images', [])
        for image in images:
            media_data.append({
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl', ''),
                'media_type': 'image'
            })
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_data.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', 'Unknown'),
                'percentage': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media_data),
        pd.DataFrame(color_data)
    )
```

#### Load to Database

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """Load transformed data into MySQL database"""
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            sql = """INSERT INTO artifactmetadata 
                     (id, title, culture, century, division, department, 
                      dated, classification, medium, dimensions, url)
                     VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                     ON DUPLICATE KEY UPDATE title=VALUES(title)"""
            cursor.execute(sql, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            sql = """INSERT INTO artifactmedia 
                     (artifact_id, image_url, media_type)
                     VALUES (%s, %s, %s)"""
            cursor.execute(sql, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            sql = """INSERT INTO artifactcolors 
                     (artifact_id, color, percentage)
                     VALUES (%s, %s, %s)"""
            cursor.execute(sql, tuple(row))
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. Analytical SQL Queries

```python
def get_analytics_queries():
    """Predefined analytical queries for museum data"""
    return {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century != 'Unknown'
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "Department Distribution": """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        
        "Artifacts with Images": """
            SELECT 
                CASE WHEN media_count > 0 THEN 'With Images' 
                     ELSE 'Without Images' END as category,
                COUNT(*) as count
            FROM (
                SELECT a.id, COUNT(m.media_id) as media_count
                FROM artifactmetadata a
                LEFT JOIN artifactmedia m ON a.id = m.artifact_id
                GROUP BY a.id
            ) as subquery
            GROUP BY category
        """,
        
        "Top Colors in Collection": """
            SELECT color, 
                   COUNT(*) as artifact_count,
                   AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY artifact_count DESC
            LIMIT 15
        """,
        
        "Classification Analysis": """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification != 'Unknown'
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 10
        """
    }

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    try:
        connection = mysql.connector.connect(**db_config)
        df = pd.read_sql(query, connection)
        connection.close()
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    db_config = {
        'host': st.sidebar.text_input("Database Host"),
        'user': st.sidebar.text_input("Database User"),
        'password': st.sidebar.text_input("Database Password", type="password"),
        'database': st.sidebar.text_input("Database Name")
    }
    
    # Data collection section
    st.header("📥 Data Collection")
    num_artifacts = st.number_input("Number of artifacts to collect", 
                                     min_value=10, max_value=1000, value=100)
    
    if st.button("Collect & Load Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_artifacts)
            st.success(f"Fetched {len(artifacts)} artifacts")
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed")
            
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("Data loaded successfully!")
    
    # Analytics section
    st.header("📊 Analytics Dashboard")
    
    queries = get_analytics_queries()
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            results = execute_query(queries[selected_query], db_config)
            
            if not results.empty:
                st.dataframe(results)
                
                # Auto-generate visualization
                if len(results.columns) == 2:
                    fig = px.bar(results, 
                                x=results.columns[0], 
                                y=results.columns[1],
                                title=selected_query)
                    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_collect_artifacts(api_key, total_artifacts, batch_size=100, delay=1):
    """Collect artifacts in batches with rate limiting"""
    all_artifacts = []
    
    for start in range(0, total_artifacts, batch_size):
        batch = fetch_artifacts(api_key, batch_size)
        all_artifacts.extend(batch)
        print(f"Collected {len(all_artifacts)} / {total_artifacts}")
        time.sleep(delay)  # Respect API rate limits
    
    return all_artifacts
```

### Data Quality Validation

```python
def validate_artifact_data(df):
    """Validate artifact data before loading"""
    # Check for required fields
    required_cols = ['id', 'title']
    missing = [col for col in required_cols if col not in df.columns]
    if missing:
        raise ValueError(f"Missing required columns: {missing}")
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['id'])
    
    # Handle null values
    df = df.fillna('Unknown')
    
    return df
```

## Troubleshooting

**API Rate Limit Errors:**
- Add delays between requests using `time.sleep()`
- Reduce batch sizes
- Check API documentation for rate limits

**Database Connection Issues:**
```python
# Test connection
def test_db_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("Database connection successful")
            connection.close()
            return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

**Empty API Responses:**
- Verify API key validity
- Check API endpoint availability
- Ensure proper pagination parameters

**Memory Issues with Large Datasets:**
```python
# Stream data in chunks
def stream_load_artifacts(artifacts, db_config, chunk_size=100):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i+chunk_size]
        metadata, media, colors = transform_artifacts(chunk)
        load_to_database(metadata, media, colors, db_config)
```
