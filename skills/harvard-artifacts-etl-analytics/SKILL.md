---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with MySQL and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - show me how to use the Harvard artifacts analytics app
  - help me set up a Streamlit dashboard for museum data
  - create a data engineering pipeline for art museum collections
  - analyze Harvard Art Museums data with SQL
  - build an analytics app with the Harvard API
  - set up ETL for museum artifact data
  - visualize Harvard museum collections with Plotly
---

# Harvard Artifacts ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project demonstrates a production-grade ETL pipeline that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into MySQL/TiDB, and visualizes analytics through a Streamlit dashboard.

## What This Project Does

The Harvard Artifacts Collection app provides:
- **API Integration**: Paginated data collection from Harvard Art Museums API with rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized SQL tables (metadata, media, colors)
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboard**: Streamlit UI with Plotly visualizations

**Architecture**: API → ETL (Python/Pandas) → SQL (MySQL/TiDB) → Analytics → Visualization (Streamlit/Plotly)

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

### 1. Harvard API Key

Get your API key from: https://www.harvardartmuseums.org/collections/api

Store it in environment variables:
```bash
export HARVARD_API_KEY="your_api_key_here"
```

Or create a `.env` file:
```
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Setup

Configure MySQL or TiDB Cloud connection:
```python
import os

db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER', 'root'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts')
}
```

Environment variables:
```bash
export DB_HOST="your_db_host"
export DB_PORT="3306"
export DB_USER="your_username"
export DB_PASSWORD="your_password"
export DB_NAME="harvard_artifacts"
```

## Database Schema

The ETL pipeline creates three relational tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    technique VARCHAR(500)
);

CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

## Running the Application

```bash
streamlit run app.py
```

The Streamlit dashboard will open at `http://localhost:8501`

## Key Code Patterns

### 1. Fetching Data from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts, info = fetch_artifacts(api_key, page=1, size=100)

print(f"Total artifacts: {info['totalrecords']}")
print(f"Total pages: {info['pages']}")
```

### 2. ETL Pipeline - Extract and Transform

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract artifact metadata into DataFrame"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata_list.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'technique': artifact.get('technique')
        })
    
    return pd.DataFrame(metadata_list)

def extract_media(artifacts):
    """Extract media information"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        media_list.append({
            'artifact_id': artifact_id,
            'baseimageurl': artifact.get('baseimageurl'),
            'iiifbaseuri': artifact.get('iiifbaseuri'),
            'primaryimageurl': artifact.get('primaryimageurl')
        })
    
    return pd.DataFrame(media_list)

def extract_colors(artifacts):
    """Extract color information"""
    colors_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            colors_list.append({
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    return pd.DataFrame(colors_list)

# Usage
metadata_df = extract_metadata(artifacts)
media_df = extract_media(artifacts)
colors_df = extract_colors(artifacts)
```

### 3. Loading Data to MySQL

```python
import mysql.connector
from mysql.connector import Error

def create_connection(db_config):
    """Create database connection"""
    try:
        connection = mysql.connector.connect(**db_config)
        return connection
    except Error as e:
        print(f"Error: {e}")
        return None

def load_metadata(connection, df):
    """Batch insert metadata"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, period, century, classification, division, department, dated, technique)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    # Convert DataFrame to list of tuples
    records = df.to_records(index=False).tolist()
    
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {cursor.rowcount} metadata records")

def load_media(connection, df):
    """Batch insert media data"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
    VALUES (%s, %s, %s, %s)
    """
    
    records = df.to_records(index=False).tolist()
    cursor.executemany(insert_query, records)
    connection.commit()
    print(f"Inserted {cursor.rowcount} media records")

# Usage
connection = create_connection(db_config)
if connection:
    load_metadata(connection, metadata_df)
    load_media(connection, media_df)
    connection.close()
```

### 4. Running Analytics Queries

```python
def run_query(connection, query):
    """Execute SQL query and return DataFrame"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    return pd.DataFrame(results)

# Example analytics queries
queries = {
    "Artifacts by Culture": """
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
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

# Execute query
connection = create_connection(db_config)
df_result = run_query(connection, queries["Artifacts by Culture"])
print(df_result)
```

### 5. Streamlit Dashboard with Visualization

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Queries")
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(queries.keys())
    )
    
    # Database connection
    connection = create_connection(db_config)
    
    if connection:
        # Run selected query
        with st.spinner("Running query..."):
            df = run_query(connection, queries[query_name])
        
        # Display results
        st.subheader(query_name)
        st.dataframe(df)
        
        # Visualization
        if len(df) > 0 and len(df.columns) >= 2:
            fig = px.bar(
                df.head(15),
                x=df.columns[0],
                y=df.columns[1],
                title=query_name,
                labels={df.columns[0]: df.columns[0].title(), 
                       df.columns[1]: df.columns[1].title()}
            )
            st.plotly_chart(fig, use_container_width=True)
        
        connection.close()

if __name__ == "__main__":
    main()
```

### 6. Complete ETL Pipeline Function

```python
def run_etl_pipeline(api_key, db_config, num_pages=5):
    """Complete ETL pipeline"""
    connection = create_connection(db_config)
    
    if not connection:
        print("Failed to connect to database")
        return
    
    total_artifacts = 0
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}/{num_pages}")
        
        # Extract
        artifacts, info = fetch_artifacts(api_key, page=page, size=100)
        
        # Transform
        metadata_df = extract_metadata(artifacts)
        media_df = extract_media(artifacts)
        colors_df = extract_colors(artifacts)
        
        # Load
        load_metadata(connection, metadata_df)
        load_media(connection, media_df)
        
        if not colors_df.empty:
            load_colors(connection, colors_df)
        
        total_artifacts += len(artifacts)
        print(f"Processed {len(artifacts)} artifacts")
    
    connection.close()
    print(f"ETL Complete: {total_artifacts} total artifacts processed")

# Run pipeline
api_key = os.getenv('HARVARD_API_KEY')
run_etl_pipeline(api_key, db_config, num_pages=10)
```

## Common Patterns

### Pagination Handling

```python
def fetch_all_artifacts(api_key, max_pages=None):
    """Fetch all artifacts with pagination"""
    all_artifacts = []
    page = 1
    
    while True:
        artifacts, info = fetch_artifacts(api_key, page=page)
        all_artifacts.extend(artifacts)
        
        print(f"Fetched page {page}/{info['pages']}")
        
        if page >= info['pages'] or (max_pages and page >= max_pages):
            break
        
        page += 1
    
    return all_artifacts
```

### Error Handling

```python
def safe_api_call(api_key, page, retries=3):
    """API call with retry logic"""
    import time
    
    for attempt in range(retries):
        try:
            artifacts, info = fetch_artifacts(api_key, page=page)
            return artifacts, info
        except requests.exceptions.RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt < retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

## Troubleshooting

**API Rate Limiting**: Harvard API has rate limits. Add delays between requests:
```python
import time
time.sleep(0.5)  # 500ms delay between requests
```

**Database Connection Issues**: Verify credentials and network access:
```python
try:
    connection = mysql.connector.connect(**db_config)
    print("Connected successfully")
except Error as e:
    print(f"Connection failed: {e}")
```

**Missing Environment Variables**: Check all required vars are set:
```python
required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_PASSWORD']
missing = [var for var in required_vars if not os.getenv(var)]
if missing:
    raise ValueError(f"Missing environment variables: {missing}")
```

**Empty DataFrames**: Handle cases where API returns no data:
```python
if df.empty:
    print("No data returned from API")
else:
    load_metadata(connection, df)
```
