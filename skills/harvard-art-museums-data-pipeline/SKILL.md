---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build a data pipeline for Harvard Art Museums
  - create ETL workflow with museum artifacts
  - analyze Harvard museum collection data
  - set up Streamlit analytics dashboard for art data
  - query Harvard Art Museums API
  - design SQL schema for artifact data
  - visualize museum collection with Plotly
  - process museum API data with Python
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides an end-to-end data engineering solution for the Harvard Art Museums API. It demonstrates ETL pipeline construction, SQL database design, analytical querying, and interactive visualization using Streamlit. The architecture follows: **API → ETL → SQL → Analytics → Visualization**.

Key capabilities:
- Extract artifact data from Harvard Art Museums API with pagination
- Transform nested JSON into normalized relational tables
- Load data into MySQL/TiDB Cloud databases
- Execute analytical SQL queries
- Visualize results with interactive Plotly dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements** (typical setup):
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

1. Obtain a free API key from [Harvard Art Museums](https://www.harvardartmuseums.org/collections/api)
2. Store credentials securely:

```python
# .env file
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Configuration

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

## Database Schema

Create the following tables:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    credit_line TEXT,
    accession_number VARCHAR(100),
    PRIMARY KEY (id)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url VARCHAR(1000),
    primary_image_url VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
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

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'fields': 'id,title,culture,century,classification,department,division,dated,period,technique,medium,dimensions,creditline,accessionnumber,primaryimageurl,baseimageurl,colors'
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
def collect_artifacts(num_pages=5):
    all_artifacts = []
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(data.get('records', []))
        print(f"Fetched page {page}, total artifacts: {len(all_artifacts)}")
    return all_artifacts
```

### Transform: Normalize JSON Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform nested JSON into normalized dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'credit_line': artifact.get('creditline'),
            'accession_number': artifact.get('accessionnumber')
        })
        
        # Media
        media_records.append({
            'artifact_id': artifact.get('id'),
            'base_image_url': artifact.get('baseimageurl'),
            'primary_image_url': artifact.get('primaryimageurl')
        })
        
        # Colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load: Insert into Database

```python
def load_to_database(metadata_df, media_df, colors_df, cursor, conn):
    """
    Batch insert data into MySQL tables
    """
    # Insert metadata
    metadata_sql = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, 
         dated, period, technique, medium, dimensions, credit_line, accession_number)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_sql, metadata_df.values.tolist())
    
    # Insert media
    media_sql = """
        INSERT INTO artifactmedia (artifact_id, base_image_url, primary_image_url)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(media_sql, media_df.values.tolist())
    
    # Insert colors
    colors_sql = """
        INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
        VALUES (%s, %s, %s, %s, %s)
    """
    cursor.executemany(colors_sql, colors_df.values.tolist())
    
    conn.commit()
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Artifacts by Culture
artifacts_by_culture = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 20
"""

# Query 2: Media Availability Analysis
media_availability = """
    SELECT 
        COUNT(*) as total_artifacts,
        SUM(CASE WHEN primary_image_url IS NOT NULL THEN 1 ELSE 0 END) as with_images,
        ROUND(100.0 * SUM(CASE WHEN primary_image_url IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*), 2) as image_percentage
    FROM artifactmedia
"""

# Query 3: Top Colors Across Collection
top_colors = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 4: Artifacts by Department and Century
dept_century_analysis = """
    SELECT department, century, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND century IS NOT NULL
    GROUP BY department, century
    ORDER BY count DESC
    LIMIT 30
"""

# Execute query
def execute_query(cursor, query):
    cursor.execute(query)
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import mysql.connector
import pandas as pd
import os
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Database connection
@st.cache_resource
def get_database_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

conn = get_database_connection()
cursor = conn.cursor()

# Sidebar navigation
analysis_type = st.sidebar.selectbox(
    "Select Analysis",
    ["Artifacts by Culture", "Media Availability", "Color Analysis", "Department Distribution"]
)

if analysis_type == "Artifacts by Culture":
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """
    df = execute_query(cursor, query)
    
    st.subheader("Top 20 Cultures in Collection")
    st.dataframe(df)
    
    fig = px.bar(df, x='culture', y='count', 
                 title='Artifact Distribution by Culture',
                 labels={'count': 'Number of Artifacts', 'culture': 'Culture'})
    st.plotly_chart(fig, use_container_width=True)

elif analysis_type == "Color Analysis":
    query = """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_coverage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """
    df = execute_query(cursor, query)
    
    st.subheader("Most Common Colors")
    
    fig = px.bar(df, x='color', y='frequency',
                 color='avg_coverage',
                 title='Color Frequency and Average Coverage',
                 labels={'frequency': 'Usage Count', 'avg_coverage': 'Avg Coverage %'})
    st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Full ETL Workflow

```python
from dotenv import load_dotenv
import mysql.connector
import os

load_dotenv()

# 1. Extract
artifacts = collect_artifacts(num_pages=10)

# 2. Transform
metadata_df, media_df, colors_df = transform_artifacts(artifacts)

# 3. Load
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)
cursor = conn.cursor()

load_to_database(metadata_df, media_df, colors_df, cursor, conn)

cursor.close()
conn.close()
```

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, size=100, delay=0.5):
    """Fetch with rate limiting to avoid API throttling"""
    data = fetch_artifacts(page=page, size=size)
    time.sleep(delay)  # Wait between requests
    return data
```

## Troubleshooting

**API Connection Issues:**
- Verify API key is valid and active
- Check internet connectivity
- Ensure API rate limits aren't exceeded (add delays between requests)

**Database Connection Errors:**
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Database connected successfully")
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

**Missing Data in Results:**
- Use `IS NOT NULL` filters in SQL queries
- Handle null values during transformation:
```python
metadata_records.append({
    'culture': artifact.get('culture', 'Unknown'),
    'century': artifact.get('century', 'Unknown')
})
```

**Streamlit Caching Issues:**
```python
# Clear cache
st.cache_data.clear()
st.cache_resource.clear()
```

**Large Dataset Performance:**
- Use batch inserts (executemany)
- Add database indexes:
```sql
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_department ON artifactmetadata(department);
```
