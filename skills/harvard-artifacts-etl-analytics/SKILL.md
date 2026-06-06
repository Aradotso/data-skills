---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I extract data from the Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create a Streamlit dashboard for art collection analytics
  - query Harvard artifacts database with SQL
  - visualize museum collection data with Plotly
  - set up TiDB Cloud for artifact metadata storage
  - implement pagination for Harvard API requests
  - transform nested JSON art data into relational tables
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL analytics, and interactive data visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data
- **SQL Storage**: Structured relational database with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **Analytics**: 20+ predefined SQL queries for insights on culture, century, media, and color patterns
- **Visualization**: Interactive Plotly dashboards integrated with Streamlit

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

### 1. Harvard API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```env
HARVARD_API_KEY=your_api_key_here
```

### 2. Database Configuration

Set up MySQL or TiDB Cloud credentials in `.env`:
```env
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 3. Database Schema

Create the following tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    division VARCHAR(255),
    department VARCHAR(255),
    classification VARCHAR(255),
    dated VARCHAR(255),
    url VARCHAR(500),
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key API Usage Patterns

### Fetching Data from Harvard API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
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
        return data['records'], data['info']['totalrecords']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch first page
artifacts, total = fetch_artifacts(page=1, size=100)
print(f"Total artifacts: {total}")
print(f"Fetched: {len(artifacts)} artifacts")
```

### ETL Pipeline Implementation

```python
import pandas as pd
import mysql.connector
from typing import List, Dict

def extract_artifacts(num_pages=5):
    """Extract artifact data from API"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        artifacts, total = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(artifacts)
        print(f"Extracted page {page}/{num_pages}")
    
    return all_artifacts

def transform_artifacts(artifacts: List[Dict]):
    """Transform nested JSON into relational format"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'division': artifact.get('division'),
            'department': artifact.get('department'),
            'classification': artifact.get('classification'),
            'dated': artifact.get('dated'),
            'url': artifact.get('url'),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        }
        metadata_list.append(metadata)
        
        # Extract media
        if artifact.get('images'):
            for img in artifact['images']:
                media_list.append({
                    'artifact_id': artifact['id'],
                    'media_type': 'image',
                    'media_url': img.get('baseimageurl')
                })
        
        # Extract colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors_list.append({
                    'artifact_id': artifact['id'],
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL/TiDB"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, division, department, 
             classification, dated, url, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, media_type, media_url)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    print("Data loaded successfully!")

# Run ETL pipeline
artifacts = extract_artifacts(num_pages=5)
metadata_df, media_df, colors_df = transform_artifacts(artifacts)
load_to_database(metadata_df, media_df, colors_df)
```

## SQL Analytics Queries

### Common Analytics Patterns

```python
def execute_query(query: str):
    """Execute SQL query and return DataFrame"""
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

# Example queries
queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Most Popular Artifacts": """
        SELECT title, culture, totalpageviews
        FROM artifactmetadata
        ORDER BY totalpageviews DESC
        LIMIT 20
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT am.id) as artifacts_with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts
        FROM artifactmetadata am
        INNER JOIN artifactmedia m ON am.id = m.artifact_id
    """
}

# Execute and display
for name, query in queries.items():
    print(f"\n{name}:")
    result = execute_query(query)
    print(result)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar configuration
st.sidebar.header("Configuration")
api_key_input = st.sidebar.text_input("Harvard API Key", type="password")

# Query selection
query_options = list(queries.keys())
selected_query = st.sidebar.selectbox("Select Analytics Query", query_options)

# Execute query button
if st.sidebar.button("Run Query"):
    with st.spinner("Executing query..."):
        query = queries[selected_query]
        result_df = execute_query(query)
        
        # Display results
        st.subheader(f"Results: {selected_query}")
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(
                result_df,
                x=result_df.columns[0],
                y=result_df.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig, use_container_width=True)

# ETL Pipeline section
st.sidebar.markdown("---")
st.sidebar.header("ETL Pipeline")
num_pages = st.sidebar.slider("Pages to Extract", 1, 10, 5)

if st.sidebar.button("Run ETL Pipeline"):
    with st.spinner("Running ETL..."):
        artifacts = extract_artifacts(num_pages=num_pages)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        st.success(f"Loaded {len(metadata_df)} artifacts!")
```

## Common Patterns

### Rate Limiting and Error Handling

```python
import time
from requests.exceptions import RequestException

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except RequestException as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt
            print(f"Retry {attempt + 1}/{max_retries} after {wait_time}s")
            time.sleep(wait_time)
```

### Batch Processing

```python
def batch_insert(cursor, table, data, batch_size=1000):
    """Insert data in batches for better performance"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        # Insert batch
        cursor.executemany(insert_query, batch)
        print(f"Inserted batch {i//batch_size + 1}")
```

## Troubleshooting

**API Rate Limiting**: Harvard API has rate limits. Add delays between requests:
```python
time.sleep(0.5)  # 500ms between requests
```

**Database Connection Errors**: Verify credentials and network access to TiDB Cloud/MySQL.

**Memory Issues with Large Datasets**: Process in smaller batches:
```python
for page in range(1, 101):  # Process 100 pages
    artifacts, _ = fetch_artifacts(page=page, size=100)
    # Process and insert immediately
    metadata_df, media_df, colors_df = transform_artifacts(artifacts)
    load_to_database(metadata_df, media_df, colors_df)
```

**Missing Data Fields**: Always use `.get()` with defaults:
```python
culture = artifact.get('culture', 'Unknown')
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

This skill provides the foundation for building production-ready data engineering pipelines for museum collections and cultural heritage data analytics.
