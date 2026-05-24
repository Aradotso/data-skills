---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I extract data from Harvard Art Museums API
  - build an ETL pipeline for art museum collections
  - create analytics dashboard with Streamlit and Harvard API
  - query Harvard artifacts database with SQL
  - visualize art museum data with Plotly
  - implement data engineering pipeline for museum artifacts
  - process Harvard Art Museums API responses
  - design relational schema for artifact metadata
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates ETL pipeline construction, relational database design, SQL analytics, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

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

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Configuration

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
```

## Database Schema

### Create Tables

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    base_url VARCHAR(500),
    alt_text TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_name VARCHAR(100),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from Harvard API

```python
import requests
import time

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        
        # Rate limiting
        time.sleep(0.5)
        
        return data
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

def fetch_all_artifacts(api_key, max_pages=10):
    """
    Fetch multiple pages of artifacts
    """
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
        else:
            break
    
    return all_artifacts
```

### Transform: Process and Structure Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API response to structured metadata
    """
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'description': artifact.get('description', ''),
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'accession_number': artifact.get('accessionyear', ''),
            'url': artifact.get('url', '')[:500]
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """
    Extract media/image data from artifacts
    """
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for img in images:
            media = {
                'artifact_id': artifact_id,
                'image_url': img.get('iiifbaseuri', '')[:500],
                'base_url': img.get('baseimageurl', '')[:500],
                'alt_text': img.get('alttext', '')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """
    Extract color information from artifacts
    """
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_data = {
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_name': color.get('color', '')[:100],
                'percentage': color.get('percent', 0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Insert Data into Database

```python
def load_metadata_to_db(df, conn):
    """
    Batch insert artifact metadata into database
    """
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, dated, 
     description, technique, medium, dimensions, creditline, accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    data_tuples = list(df.itertuples(index=False, name=None))
    cursor.executemany(insert_query, data_tuples)
    conn.commit()
    
    print(f"Inserted {cursor.rowcount} metadata records")
    cursor.close()

def load_media_to_db(df, conn):
    """
    Insert artifact media data
    """
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, base_url, alt_text)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = list(df.itertuples(index=False, name=None))
    cursor.executemany(insert_query, data_tuples)
    conn.commit()
    
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()

def load_colors_to_db(df, conn):
    """
    Insert artifact color data
    """
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color_hex, color_name, percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    data_tuples = list(df.itertuples(index=False, name=None))
    cursor.executemany(insert_query, data_tuples)
    conn.commit()
    
    print(f"Inserted {cursor.rowcount} color records")
    cursor.close()
```

## Analytics SQL Queries

### Common Analysis Patterns

```python
# Query 1: Artifacts by Culture
query_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by Century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != ''
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Department Distribution
query_department = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Query 4: Most Common Colors
query_colors = """
SELECT color_name, COUNT(*) as frequency, AVG(percentage) as avg_percentage
FROM artifactcolors
WHERE color_name IS NOT NULL
GROUP BY color_name
ORDER BY frequency DESC
LIMIT 15
"""

# Query 5: Artifacts with Media
query_media_availability = """
SELECT 
    CASE WHEN media_count > 0 THEN 'With Images' ELSE 'No Images' END as media_status,
    COUNT(*) as artifact_count
FROM (
    SELECT m.id, COUNT(am.media_id) as media_count
    FROM artifactmetadata m
    LEFT JOIN artifactmedia am ON m.id = am.artifact_id
    GROUP BY m.id
) as media_summary
GROUP BY media_status
"""

# Query 6: Classification Distribution
query_classification = """
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL AND classification != ''
GROUP BY classification
ORDER BY count DESC
LIMIT 10
"""
```

### Execute Queries and Return Results

```python
def execute_query(query, conn):
    """
    Execute SQL query and return results as DataFrame
    """
    cursor = conn.cursor()
    cursor.execute(query)
    
    # Fetch column names
    columns = [desc[0] for desc in cursor.description]
    
    # Fetch all rows
    rows = cursor.fetchall()
    cursor.close()
    
    # Convert to DataFrame
    df = pd.DataFrame(rows, columns=columns)
    return df
```

## Streamlit Dashboard Implementation

### Complete ETL and Visualization App

```python
import streamlit as st
import pandas as pd
import plotly.express as px
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Main app
st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")

queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as total
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total DESC
    """,
    "Color Analysis": """
        SELECT color_name, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color_name IS NOT NULL
        GROUP BY color_name
        ORDER BY frequency DESC
        LIMIT 15
    """
}

selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute query
if st.sidebar.button("Run Query"):
    conn = get_db_connection()
    df = execute_query(queries[selected_query], conn)
    
    st.subheader(f"Results: {selected_query}")
    st.dataframe(df, use_container_width=True)
    
    # Visualization
    if len(df) > 0 and len(df.columns) >= 2:
        fig = px.bar(
            df,
            x=df.columns[0],
            y=df.columns[1],
            title=selected_query,
            labels={df.columns[0]: df.columns[0].title(), df.columns[1]: 'Count'}
        )
        st.plotly_chart(fig, use_container_width=True)
```

## Complete ETL Workflow

```python
def run_complete_etl_pipeline(api_key, db_config, max_pages=5):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Starting extraction...")
    artifacts = fetch_all_artifacts(api_key, max_pages)
    print(f"Extracted {len(artifacts)} artifacts")
    
    # Transform
    print("Transforming data...")
    metadata_df = transform_artifact_metadata(artifacts)
    media_df = transform_artifact_media(artifacts)
    colors_df = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading to database...")
    conn = mysql.connector.connect(**db_config)
    
    load_metadata_to_db(metadata_df, conn)
    load_media_to_db(media_df, conn)
    load_colors_to_db(colors_df, conn)
    
    conn.close()
    print("ETL pipeline completed successfully!")

# Usage
if __name__ == "__main__":
    api_key = os.getenv('HARVARD_API_KEY')
    db_config = {
        'host': os.getenv('DB_HOST'),
        'user': os.getenv('DB_USER'),
        'password': os.getenv('DB_PASSWORD'),
        'database': os.getenv('DB_NAME')
    }
    
    run_complete_etl_pipeline(api_key, db_config, max_pages=10)
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting
- Add `time.sleep(0.5)` between requests
- Reduce batch size if getting 429 errors
- Check API key validity

### Database Connection Issues
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Connection successful!")
except mysql.connector.Error as e:
    print(f"Connection failed: {e}")
```

### Missing Data Fields
```python
# Safe field extraction
def safe_get(dictionary, key, default='', max_length=None):
    value = dictionary.get(key, default)
    if max_length and len(str(value)) > max_length:
        return str(value)[:max_length]
    return value
```

### Duplicate Records
```sql
-- Use ON DUPLICATE KEY UPDATE in insert statements
INSERT INTO artifactmetadata (...) VALUES (...)
ON DUPLICATE KEY UPDATE title=VALUES(title)
```

## Best Practices

1. **Use environment variables** for all credentials
2. **Implement pagination** for large datasets
3. **Add rate limiting** to respect API quotas
4. **Use batch inserts** for performance
5. **Handle NULL values** gracefully in transformations
6. **Cache database connections** in Streamlit with `@st.cache_resource`
7. **Index foreign keys** for query performance
