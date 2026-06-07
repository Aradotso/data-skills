---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering app with museum artifacts
  - set up analytics dashboard for Harvard art collection
  - extract and transform Harvard museum data to SQL
  - build Streamlit app for artifact analytics
  - implement batch processing for museum API data
  - query and visualize art museum metadata
  - design relational database for artifact collections
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection application:
- Extracts artifact data from the Harvard Art Museums API with pagination support
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases with batch inserts
- Executes analytical SQL queries on artifact metadata, media, and color data
- Visualizes results through interactive Plotly charts in a Streamlit interface

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements** (typical dependencies):
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
MYSQL_HOST=your_mysql_host
MYSQL_PORT=3306
MYSQL_USER=your_username
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=harvard_artifacts
```

### Harvard API Key

Get your free API key from: https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z1MWSWpl4/viewform

### Database Setup

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accession_number VARCHAR(100)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_url TEXT,
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

Access at `http://localhost:8501`

## Core ETL Pipeline Pattern

### Extract: Fetching API Data with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Extract artifacts from Harvard Art Museums API with pagination."""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max records per request
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            print(f"Error: {response.status_code}")
            break
        
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
        
        artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return artifacts[:num_records]
```

### Transform: Normalizing Nested JSON

```python
import pandas as pd

def transform_artifacts(raw_artifacts):
    """Transform raw API data into normalized dataframes."""
    
    # Metadata extraction
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        artifact_id = artifact.get('id')
        
        # Extract metadata
        metadata = {
            'id': artifact_id,
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accession_number': artifact.get('accessionNumber', 'Unknown')
        }
        metadata_list.append(metadata)
        
        # Extract media (images)
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'media_url': img.get('baseimageurl', ''),
                'media_type': 'image'
            }
            media_list.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0)
            }
            colors_list.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_list)
    df_media = pd.DataFrame(media_list)
    df_colors = pd.DataFrame(colors_list)
    
    return df_metadata, df_media, df_colors
```

### Load: Batch Insert into SQL

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection."""
    return mysql.connector.connect(
        host=os.getenv('MYSQL_HOST'),
        port=int(os.getenv('MYSQL_PORT', 3306)),
        user=os.getenv('MYSQL_USER'),
        password=os.getenv('MYSQL_PASSWORD'),
        database=os.getenv('MYSQL_DATABASE')
    )

def load_to_sql(df_metadata, df_media, df_colors):
    """Load dataframes into SQL database with batch inserts."""
    
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, dated, accession_number)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = df_metadata.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia (artifact_id, media_url, media_type)
            VALUES (%s, %s, %s)
        """
        media_values = df_media.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
        """
        colors_values = df_colors.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        conn.commit()
        print(f"Successfully loaded {len(df_metadata)} artifacts")
        
    except Error as e:
        print(f"Error loading data: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytical SQL Queries

### Example Queries for Insights

```python
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count 
        FROM artifactmetadata 
        GROUP BY culture 
        ORDER BY count DESC 
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count 
        FROM artifactmetadata 
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century 
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY artifact_count DESC
    """,
    
    "Most Common Classifications": """
        SELECT classification, COUNT(*) as count 
        FROM artifactmetadata 
        GROUP BY classification 
        ORDER BY count DESC 
        LIMIT 15
    """,
    
    "Color Usage Analysis": """
        SELECT color_hex, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percent
        FROM artifactcolors 
        GROUP BY color_hex 
        ORDER BY usage_count DESC 
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(a.id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.id = a.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """
}

def execute_query(query_name):
    """Execute analytical query and return results."""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    query = ANALYTICAL_QUERIES.get(query_name)
    if not query:
        return None
    
    try:
        cursor.execute(query)
        columns = [desc[0] for desc in cursor.description]
        results = cursor.fetchall()
        df = pd.DataFrame(results, columns=columns)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        cursor.close()
        conn.close()
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Select Module",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "ETL Pipeline":
        st.header("Data Collection & ETL")
        
        num_records = st.number_input(
            "Number of artifacts to collect",
            min_value=10,
            max_value=1000,
            value=100
        )
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_artifacts(num_records)
            
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                df_meta, df_media, df_colors = transform_artifacts(artifacts)
            
            st.info(f"Transformed into {len(df_meta)} metadata, "
                   f"{len(df_media)} media, {len(df_colors)} color records")
            
            with st.spinner("Loading to database..."):
                load_to_sql(df_meta, df_media, df_colors)
            
            st.success("ETL Pipeline Complete!")
    
    elif menu == "SQL Analytics":
        st.header("SQL Query Analytics")
        
        query_name = st.selectbox(
            "Select Analysis",
            list(ANALYTICAL_QUERIES.keys())
        )
        
        if st.button("Execute Query"):
            df_result = execute_query(query_name)
            
            if df_result is not None and not df_result.empty:
                st.dataframe(df_result)
                
                # Auto-generate visualization
                if len(df_result.columns) == 2:
                    fig = px.bar(
                        df_result,
                        x=df_result.columns[0],
                        y=df_result.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig)
            else:
                st.warning("No results found")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental Data Loading

```python
def get_last_artifact_id():
    """Get the highest artifact ID in database."""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    conn.close()
    return result or 0

def incremental_etl():
    """Only fetch artifacts newer than last loaded."""
    last_id = get_last_artifact_id()
    # Implement API call with id filter if supported
    # Otherwise, filter after extraction
    pass
```

### Error Handling for API Rate Limits

```python
from time import sleep

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff on rate limit."""
    for attempt in range(max_retries):
        response = requests.get(url, params=params)
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 429:  # Rate limit
            wait_time = 2 ** attempt
            print(f"Rate limited. Waiting {wait_time}s...")
            sleep(wait_time)
        else:
            raise Exception(f"API Error: {response.status_code}")
    
    raise Exception("Max retries exceeded")
```

## Troubleshooting

**API Key Issues**: Ensure your `.env` file is in the project root and properly loaded with `python-dotenv`.

**Database Connection Errors**: Verify MySQL credentials and that the database exists. For TiDB Cloud, use the provided SSL connection string.

**Missing Data in Queries**: Check that ETL pipeline has completed successfully. Run `SELECT COUNT(*) FROM artifactmetadata` to verify data exists.

**Streamlit Not Loading**: Ensure all dependencies are installed and you're running from the correct directory with `streamlit run app.py`.

**Memory Issues with Large Datasets**: Use pagination and batch processing. Process data in chunks of 100-500 records at a time.
