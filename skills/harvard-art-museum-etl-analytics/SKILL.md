---
name: harvard-art-museum-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - help me create a data engineering project with museum artifacts
  - show me how to extract and analyze Harvard art collection data
  - I want to build a Streamlit dashboard for art museum data
  - how to design SQL schemas for art artifacts
  - help me visualize museum collection analytics
  - create an end-to-end data pipeline with Harvard API
  - build analytics queries for art museum collections
---

# Harvard Art Museum ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project is an end-to-end data engineering and analytics application that demonstrates how to:
- Extract artifact data from the Harvard Art Museums API
- Transform nested JSON into relational database structures
- Load data into MySQL/TiDB Cloud databases
- Run analytical SQL queries on art collection data
- Visualize insights using Streamlit and Plotly

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

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

1. Get a free API key from [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Store it securely using environment variables:

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.getenv('HARVARD_API_KEY')
```

### Database Configuration

```python
import mysql.connector

db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME'),
    'port': int(os.getenv('DB_PORT', 3306))
}

connection = mysql.connector.connect(**db_config)
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
    division VARCHAR(200),
    dated VARCHAR(200),
    period VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline

### Extract: Fetch Data from API

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
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
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage with pagination
all_artifacts = []
page = 1
max_pages = 5

while page <= max_pages:
    records, info = fetch_artifacts(os.getenv('HARVARD_API_KEY'), page=page)
    all_artifacts.extend(records)
    page += 1
    print(f"Fetched page {page-1}, total artifacts: {len(all_artifacts)}")
```

### Transform: Parse Nested JSON

```python
def transform_artifacts(records):
    """Transform nested JSON into structured dataframes"""
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        artifact_id = record.get('id')
        
        # Metadata
        metadata_list.append({
            'id': artifact_id,
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'dated': record.get('dated'),
            'period': record.get('period'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'url': record.get('url')
        })
        
        # Media
        media_list.append({
            'artifact_id': artifact_id,
            'iiifbaseuri': record.get('images', [{}])[0].get('iiifbaseuri') if record.get('images') else None,
            'baseimageurl': record.get('images', [{}])[0].get('baseimageurl') if record.get('images') else None,
            'primaryimageurl': record.get('primaryimageurl')
        })
        
        # Colors
        if record.get('colors'):
            for color in record['colors']:
                colors_list.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into Database

```python
def load_to_database(connection, metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, division, 
         dated, period, technique, medium, dimensions, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE title=VALUES(title)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, iiifbaseuri, baseimageurl, primaryimageurl)
        VALUES (%s, %s, %s, %s)
    """
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
    print(f"Loaded {len(metadata_df)} artifacts successfully")
```

## Analytical SQL Queries

### Sample Analytics

```python
# Top 10 cultures by artifact count
query_cultures = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Artifacts by century
query_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL
    GROUP BY century
    ORDER BY count DESC
"""

# Color distribution
query_colors = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 15
"""

# Artifacts with images vs without
query_images = """
    SELECT 
        CASE 
            WHEN primaryimageurl IS NOT NULL THEN 'Has Image'
            ELSE 'No Image'
        END as image_status,
        COUNT(*) as count
    FROM artifactmedia
    GROUP BY image_status
"""

# Department distribution
query_departments = """
    SELECT department, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY count DESC
"""
```

### Execute Queries

```python
def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox("Select Page", [
    "ETL Pipeline",
    "SQL Analytics",
    "Visualizations"
])

if page == "ETL Pipeline":
    st.header("ETL Pipeline Control")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", 1, 20, 5)
    
    with col2:
        page_size = st.number_input("Records per page", 10, 100, 50)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            # ETL logic here
            st.success(f"Successfully processed {num_pages * page_size} artifacts")

elif page == "SQL Analytics":
    st.header("SQL Query Analytics")
    
    queries = {
        "Top Cultures": query_cultures,
        "Century Distribution": query_century,
        "Color Analysis": query_colors,
        "Image Availability": query_images,
        "Department Breakdown": query_departments
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        df_result = execute_query(connection, queries[selected_query])
        st.dataframe(df_result)
        
        # Auto-visualization
        if len(df_result.columns) == 2:
            fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1])
            st.plotly_chart(fig, use_container_width=True)
```

### Visualization Examples

```python
# Bar chart with Plotly
def create_bar_chart(df, x_col, y_col, title):
    fig = px.bar(
        df,
        x=x_col,
        y=y_col,
        title=title,
        color=y_col,
        color_continuous_scale='Viridis'
    )
    fig.update_layout(
        xaxis_title=x_col.replace('_', ' ').title(),
        yaxis_title=y_col.replace('_', ' ').title()
    )
    return fig

# Pie chart for distributions
def create_pie_chart(df, names_col, values_col, title):
    fig = px.pie(
        df,
        names=names_col,
        values=values_col,
        title=title,
        hole=0.3
    )
    return fig

# Usage in Streamlit
st.plotly_chart(create_bar_chart(df, 'culture', 'artifact_count', 'Top Cultures'))
```

## Common Patterns

### Rate Limiting and Retries

```python
import time
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry

def create_session_with_retries():
    session = requests.Session()
    retries = Retry(total=3, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
    session.mount('https://', HTTPAdapter(max_retries=retries))
    return session

def fetch_with_rate_limit(api_key, page, delay=0.5):
    time.sleep(delay)  # Respect API rate limits
    session = create_session_with_retries()
    response = session.get(url, params=params)
    return response
```

### Batch Processing

```python
def batch_insert(cursor, query, data, batch_size=1000):
    """Insert data in batches for better performance"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(query, batch)
        connection.commit()
        print(f"Inserted batch {i//batch_size + 1}")
```

## Troubleshooting

### API Connection Issues

```python
# Check API key validity
response = requests.get(
    f"https://api.harvardartmuseums.org/object?apikey={API_KEY}&size=1"
)
if response.status_code == 401:
    print("Invalid API key")
elif response.status_code == 200:
    print("API connection successful")
```

### Database Connection Issues

```python
try:
    connection = mysql.connector.connect(**db_config)
    print("Database connected successfully")
except mysql.connector.Error as err:
    if err.errno == 2003:
        print("Cannot connect to database server")
    elif err.errno == 1045:
        print("Invalid credentials")
    else:
        print(f"Error: {err}")
```

### Empty Results

```python
# Verify data exists
cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
count = cursor.fetchone()[0]
if count == 0:
    print("No data in database. Run ETL pipeline first.")
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY=your_api_key_here
export DB_HOST=your_host
export DB_USER=your_user
export DB_PASSWORD=your_password
export DB_NAME=harvard_art

# Run Streamlit app
streamlit run app.py
```

The application will be available at `http://localhost:8501`
