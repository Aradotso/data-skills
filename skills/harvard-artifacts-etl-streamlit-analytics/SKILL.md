---
name: harvard-artifacts-etl-streamlit-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a museum artifacts analytics dashboard
  - extract and visualize Harvard museum collection data
  - set up a data engineering pipeline with Streamlit
  - query and analyze art museum artifact data
  - build a museum data warehouse with SQL
  - create an interactive art collection analytics app
  - process Harvard API data into relational database
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering application that demonstrates ETL pipelines, SQL analytics, and interactive visualization using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into MySQL/TiDB, and provides 20+ analytical queries through a Streamlit dashboard with Plotly visualizations.

**Architecture**: API → ETL → SQL → Analytics → Visualization

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

# MySQL/TiDB Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema():
    """Initialize the database schema for Harvard artifacts"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            url TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            media_type VARCHAR(100),
            base_url TEXT,
            thumbnail_url TEXT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            percentage DECIMAL(5,2),
            hue VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()
    cursor.close()
    connection.close()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Core ETL Pipeline

### Extract: Fetching Data from Harvard API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """Extract artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        
        # Respect rate limits
        time.sleep(0.5)
        
        return data.get('records', []), data.get('info', {})
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
        return [], {}

def extract_all_artifacts(api_key, max_pages=10):
    """Extract multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        records, info = fetch_artifacts(api_key, page=page)
        
        if not records:
            break
            
        all_artifacts.extend(records)
        print(f"Fetched page {page}: {len(records)} artifacts")
        
        # Check if we've reached the last page
        if info.get('next') is None:
            break
    
    return all_artifacts
```

### Transform: Processing Nested JSON

```python
import pandas as pd

def transform_metadata(artifacts):
    """Transform artifact data into metadata DataFrame"""
    metadata_records = []
    
    for artifact in artifacts:
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url')
        })
    
    return pd.DataFrame(metadata_records)

def transform_media(artifacts):
    """Extract media information from artifacts"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'artifact_id': artifact_id,
                'media_type': 'image',
                'base_url': img.get('baseimageurl'),
                'thumbnail_url': img.get('thumbnailurl')
            })
    
    return pd.DataFrame(media_records)

def transform_colors(artifacts):
    """Extract color data from artifacts"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'percentage': color.get('percent'),
                'hue': color.get('hue')
            })
    
    return pd.DataFrame(color_records)
```

### Load: Batch Insert into SQL

```python
def load_to_database(df, table_name, connection):
    """Load DataFrame to SQL table using batch insert"""
    if df.empty:
        print(f"No data to load for {table_name}")
        return
    
    cursor = connection.cursor()
    
    # Prepare column names and placeholders
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    # Build INSERT query with ON DUPLICATE KEY UPDATE for idempotency
    if table_name == 'artifactmetadata':
        query = f"""
            INSERT INTO {table_name} ({columns})
            VALUES ({placeholders})
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
    else:
        query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data_tuples = [tuple(row) for row in df.values]
    cursor.executemany(query, data_tuples)
    connection.commit()
    
    print(f"Loaded {len(df)} records into {table_name}")
    cursor.close()

def run_etl_pipeline(api_key, db_config, max_pages=5):
    """Complete ETL pipeline execution"""
    # Extract
    print("Starting extraction...")
    artifacts = extract_all_artifacts(api_key, max_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_metadata(artifacts)
    media_df = transform_media(artifacts)
    colors_df = transform_colors(artifacts)
    
    # Load
    print("Loading to database...")
    connection = mysql.connector.connect(**db_config)
    
    load_to_database(metadata_df, 'artifactmetadata', connection)
    load_to_database(media_df, 'artifactmedia', connection)
    load_to_database(colors_df, 'artifactcolors', connection)
    
    connection.close()
    print("ETL pipeline completed successfully!")
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Artifact count by culture
culture_query = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 15
"""

# Query 2: Century distribution
century_query = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Query 3: Artifacts with media availability
media_query = """
    SELECT 
        m.classification,
        COUNT(DISTINCT m.id) as total_artifacts,
        COUNT(DISTINCT a.artifact_id) as artifacts_with_media,
        ROUND(COUNT(DISTINCT a.artifact_id) * 100.0 / COUNT(DISTINCT m.id), 2) as media_percentage
    FROM artifactmetadata m
    LEFT JOIN artifactmedia a ON m.id = a.artifact_id
    GROUP BY m.classification
    HAVING total_artifacts > 10
    ORDER BY media_percentage DESC
"""

# Query 4: Most common colors across artifacts
color_query = """
    SELECT 
        color,
        COUNT(*) as occurrence,
        AVG(percentage) as avg_percentage
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY occurrence DESC
    LIMIT 10
"""

# Query 5: Department-wise artifact distribution
department_query = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""
```

## Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px

def run_query_and_visualize(query, query_name, connection):
    """Execute SQL query and create visualization"""
    st.subheader(query_name)
    
    # Execute query
    df = pd.read_sql(query, connection)
    
    # Display results table
    st.dataframe(df)
    
    # Auto-generate visualization
    if len(df.columns) == 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=query_name
        )
        st.plotly_chart(fig, use_container_width=True)
    
    return df

def main():
    st.title("🎨 Harvard Artifacts Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # ETL Controls
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            api_key = os.getenv('HARVARD_API_KEY')
            db_config = {
                'host': os.getenv('DB_HOST'),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            run_etl_pipeline(api_key, db_config)
            st.success("ETL completed!")
    
    # Analytics section
    st.header("📊 Analytics Queries")
    
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # Query selector
    queries = {
        "Artifacts by Culture": culture_query,
        "Century Distribution": century_query,
        "Media Availability": media_query,
        "Color Analysis": color_query,
        "Department Distribution": department_query
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Analysis"):
        run_query_and_visualize(
            queries[selected_query], 
            selected_query, 
            connection
        )
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Common Patterns

### Error Handling for API Calls

```python
def safe_api_call(api_key, page, retries=3):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            records, info = fetch_artifacts(api_key, page)
            return records, info
        except Exception as e:
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

### Data Quality Checks

```python
def validate_data(df, required_columns):
    """Validate DataFrame before loading"""
    missing_cols = set(required_columns) - set(df.columns)
    if missing_cols:
        raise ValueError(f"Missing columns: {missing_cols}")
    
    # Check for duplicates
    if df.duplicated(subset=['id']).any():
        df = df.drop_duplicates(subset=['id'])
    
    return df
```

## Troubleshooting

**API Rate Limiting**: Add `time.sleep()` between requests or implement exponential backoff.

**Database Connection Errors**: Verify credentials in `.env` and ensure database server is accessible.

**Large Dataset Performance**: Use batch inserts (shown in `load_to_database`) and add indexes:
```sql
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_century ON artifactmetadata(century);
CREATE INDEX idx_artifact_id ON artifactmedia(artifact_id);
```

**Missing Data**: Handle NULL values in transform functions:
```python
df['culture'].fillna('Unknown', inplace=True)
```
