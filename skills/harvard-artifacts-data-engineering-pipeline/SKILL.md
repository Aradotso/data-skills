---
name: harvard-artifacts-data-engineering-pipeline
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - build a data pipeline with Harvard Art Museums API
  - create an ETL pipeline for museum artifacts
  - analyze Harvard museum collection data
  - build a Streamlit analytics dashboard for artifacts
  - extract and transform Harvard API data to SQL
  - visualize museum artifact data with Plotly
  - create a data engineering project with museum API
  - design SQL schema for artifact collections
---

# Harvard Artifacts Data Engineering Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project demonstrates an end-to-end data engineering pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into structured formats, loads it into a SQL database, and provides interactive analytics through a Streamlit dashboard.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design and foreign keys
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly charts and Streamlit dashboard

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

### Environment Variables

Create a `.env` file or use Streamlit secrets:

```python
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

## Database Schema

The pipeline creates three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(255),
    medium VARCHAR(500),
    dated VARCHAR(100),
    accessionyear INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    imagecount INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Functions

### Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_pages=5):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,  # Max per page
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### Transform: Process JSON to DataFrames

```python
def transform_artifacts(artifacts):
    """Transform raw artifact data into normalized dataframes"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_records.append(metadata)
        
        # Extract media information
        primary_image = artifact.get('primaryimageurl')
        image_count = artifact.get('totalpageviews', 0)
        media_records.append({
            'artifact_id': artifact.get('id'),
            'baseimageurl': primary_image,
            'imagecount': image_count
        })
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent')
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def create_connection(host, user, password, database):
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def load_to_sql(df_metadata, df_media, df_colors, connection):
    """Batch insert dataframes into SQL tables"""
    cursor = connection.cursor()
    
    # Insert metadata
    insert_metadata = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, technique, medium, dated, accessionyear)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    
    metadata_values = df_metadata.values.tolist()
    cursor.executemany(insert_metadata, metadata_values)
    
    # Insert media
    insert_media = """
    INSERT INTO artifactmedia (artifact_id, baseimageurl, imagecount)
    VALUES (%s, %s, %s)
    """
    media_values = df_media.values.tolist()
    cursor.executemany(insert_media, media_values)
    
    # Insert colors
    insert_colors = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, percentage)
    VALUES (%s, %s, %s, %s)
    """
    color_values = df_colors.values.tolist()
    cursor.executemany(insert_colors, color_values)
    
    connection.commit()
    cursor.close()
    print("Data loaded successfully!")
```

## Streamlit Dashboard Integration

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password", 
                                     value=os.getenv('HARVARD_API_KEY', ''))
    
    # Database connection
    conn = create_connection(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    # Navigation
    page = st.sidebar.selectbox("Choose Page", 
                                 ["ETL Pipeline", "SQL Analytics", "Visualizations"])
    
    if page == "ETL Pipeline":
        show_etl_page(api_key, conn)
    elif page == "SQL Analytics":
        show_analytics_page(conn)
    elif page == "Visualizations":
        show_viz_page(conn)

def show_etl_page(api_key, conn):
    st.header("ETL Pipeline")
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=10, value=2)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(api_key, num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_sql(df_metadata, df_media, df_colors, conn)
            st.success("Data loaded to SQL database")
        
        st.dataframe(df_metadata.head())

def show_analytics_page(conn):
    st.header("SQL Analytics")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as count 
            FROM artifactmetadata 
            WHERE culture IS NOT NULL 
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
            SELECT department, COUNT(*) as count 
            FROM artifactmetadata 
            GROUP BY department 
            ORDER BY count DESC
        """,
        "Most Common Colors": """
            SELECT color, COUNT(*) as frequency 
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY frequency DESC 
            LIMIT 10
        """,
        "Artifacts with Images": """
            SELECT 
                COUNT(CASE WHEN baseimageurl IS NOT NULL THEN 1 END) as with_images,
                COUNT(CASE WHEN baseimageurl IS NULL THEN 1 END) as without_images
            FROM artifactmedia
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df = pd.read_sql(queries[selected_query], conn)
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2 and df.shape[0] > 1:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Analytics Queries

### Artifact Distribution Analysis

```python
# Artifacts by classification and century
query = """
SELECT classification, century, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND century IS NOT NULL
GROUP BY classification, century
ORDER BY count DESC
LIMIT 20
"""

# Color spectrum analysis
query = """
SELECT spectrum, AVG(percentage) as avg_percentage, COUNT(*) as frequency
FROM artifactcolors
GROUP BY spectrum
ORDER BY frequency DESC
"""

# Accession year trends
query = """
SELECT accessionyear, COUNT(*) as count
FROM artifactmetadata
WHERE accessionyear IS NOT NULL
GROUP BY accessionyear
ORDER BY accessionyear
"""
```

## Running the Application

```bash
# Run the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_artifacts_with_delay(api_key, num_pages=5, delay=1):
    """Add delay between requests to avoid rate limiting"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        response = requests.get(base_url, params=params)
        all_artifacts.extend(response.json().get('records', []))
        time.sleep(delay)  # Rate limiting
    
    return all_artifacts
```

### Database Connection Issues

```python
def create_connection_with_retry(host, user, password, database, max_retries=3):
    """Retry database connection"""
    for attempt in range(max_retries):
        try:
            connection = mysql.connector.connect(
                host=host, user=user, password=password, database=database
            )
            return connection
        except Error as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
            else:
                raise e
```

### Handling Missing Data

```python
def transform_artifacts_safe(artifacts):
    """Handle missing fields gracefully"""
    metadata_records = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            # Use .get() with defaults for all fields
        }
        metadata_records.append(metadata)
    
    return pd.DataFrame(metadata_records)
```

## Performance Optimization

```python
# Batch processing for large datasets
def batch_insert(cursor, query, data, batch_size=1000):
    """Insert data in batches for better performance"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(query, batch)
        cursor.connection.commit()
```
