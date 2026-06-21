---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with the Harvard Art Museums API
  - show me how to create an ETL workflow for museum artifacts
  - help me set up a Streamlit analytics dashboard for art data
  - how to extract and transform Harvard museum API data
  - build an end-to-end data engineering project with museum artifacts
  - create SQL analytics from Harvard Art Museums collection
  - visualize museum artifact data with Plotly and Streamlit
  - implement batch data loading for art museum collections
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations using Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Run predefined analytical queries on structured museum data
- **Interactive Dashboards**: Visualize insights using Streamlit and Plotly
- **Database Design**: Proper relational schema with foreign keys for artifacts, media, and colors

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

1. Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Create database connection
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)
```

### Database Schema

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    primaryimageurl TEXT,
    url TEXT
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    iiifbaseuri TEXT,
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

## Key API Patterns

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only get artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages
def fetch_all_artifacts(max_pages=10):
    """Fetch artifacts across multiple pages"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(page=page, size=100)
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            
            # Check if there are more pages
            if data.get('info', {}).get('next') is None:
                break
                
        except Exception as e:
            print(f"Error fetching page {page}: {e}")
            break
    
    return all_artifacts
```

## ETL Pipeline Implementation

### Extract

```python
import pandas as pd

def extract_artifact_data(artifacts):
    """Extract artifact metadata from API response"""
    metadata_records = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'url': artifact.get('url')
        }
        metadata_records.append(record)
    
    return pd.DataFrame(metadata_records)
```

### Transform

```python
def transform_media_data(artifacts):
    """Extract and transform media data from nested structure"""
    media_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        images = artifact.get('images', [])
        
        for image in images:
            media_record = {
                'artifact_id': artifact_id,
                'baseimageurl': image.get('baseimageurl'),
                'format': image.get('format'),
                'iiifbaseuri': image.get('iiifbaseuri')
            }
            media_records.append(media_record)
    
    return pd.DataFrame(media_records)

def transform_color_data(artifacts):
    """Extract and transform color data"""
    color_records = []
    
    for artifact in artifacts:
        artifact_id = artifact.get('id')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_record = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_records.append(color_record)
    
    return pd.DataFrame(color_records)
```

### Load

```python
def load_to_database(df, table_name, conn):
    """Load DataFrame to SQL database using batch insert"""
    cursor = conn.cursor()
    
    # Prepare column names and placeholders
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    # Create insert query
    query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Convert DataFrame to list of tuples
    records = [tuple(x) for x in df.to_numpy()]
    
    # Batch insert
    try:
        cursor.executemany(query, records)
        conn.commit()
        print(f"Successfully loaded {len(records)} records into {table_name}")
    except Exception as e:
        conn.rollback()
        print(f"Error loading data: {e}")
    finally:
        cursor.close()

# Usage
artifacts = fetch_all_artifacts(max_pages=5)
metadata_df = extract_artifact_data(artifacts)
media_df = transform_media_data(artifacts)
colors_df = transform_color_data(artifacts)

load_to_database(metadata_df, 'artifactmetadata', conn)
load_to_database(media_df, 'artifactmedia', conn)
load_to_database(colors_df, 'artifactcolors', conn)
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Count artifacts by culture
query_by_culture = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Query 2: Artifacts by century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Query 3: Department distribution
query_by_department = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Query 4: Color analysis
query_color_analysis = """
SELECT c.color, AVG(c.percent) as avg_percent, COUNT(*) as frequency
FROM artifactcolors c
GROUP BY c.color
ORDER BY frequency DESC
LIMIT 15
"""

# Query 5: Artifacts with media
query_media_availability = """
SELECT 
    COUNT(DISTINCT am.artifact_id) as artifacts_with_media,
    COUNT(am.id) as total_media_items
FROM artifactmedia am
"""

# Query 6: Join artifacts with color data
query_artifacts_with_colors = """
SELECT 
    a.title,
    a.culture,
    c.color,
    c.percent
FROM artifactmetadata a
JOIN artifactcolors c ON a.id = c.artifact_id
WHERE c.percent > 50
ORDER BY c.percent DESC
LIMIT 20
"""

def execute_query(query, conn):
    """Execute SQL query and return DataFrame"""
    return pd.read_sql(query, conn)
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox(
    "Select Analysis",
    ["Overview", "Culture Analysis", "Color Analysis", "Department Stats"]
)

if page == "Culture Analysis":
    st.header("Artifacts by Culture")
    
    df = execute_query(query_by_culture, conn)
    
    # Display table
    st.dataframe(df)
    
    # Visualization
    fig = px.bar(
        df,
        x='culture',
        y='artifact_count',
        title='Top Cultures by Artifact Count',
        labels={'culture': 'Culture', 'artifact_count': 'Number of Artifacts'}
    )
    st.plotly_chart(fig, use_container_width=True)

elif page == "Color Analysis":
    st.header("Color Distribution Analysis")
    
    df = execute_query(query_color_analysis, conn)
    
    fig = px.bar(
        df,
        x='color',
        y='frequency',
        title='Most Common Colors in Artifacts',
        color='avg_percent',
        labels={'color': 'Color', 'frequency': 'Frequency'}
    )
    st.plotly_chart(fig, use_container_width=True)
```

### Interactive Query Builder

```python
st.subheader("Custom Query Builder")

# Dropdown for predefined queries
queries = {
    "Artifacts by Culture": query_by_culture,
    "Artifacts by Century": query_by_century,
    "Color Analysis": query_color_analysis,
    "Department Distribution": query_by_department
}

selected_query = st.selectbox("Select a query", list(queries.keys()))

if st.button("Run Query"):
    with st.spinner("Executing query..."):
        result_df = execute_query(queries[selected_query], conn)
        
        st.success(f"Retrieved {len(result_df)} rows")
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) >= 2:
            fig = px.bar(
                result_df,
                x=result_df.columns[0],
                y=result_df.columns[1],
                title=f"{selected_query} Results"
            )
            st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Fetch data with rate limiting"""
    data = fetch_artifacts(page=page)
    time.sleep(delay)  # Delay between requests
    return data
```

### Error Handling

```python
def safe_fetch_artifacts(page):
    """Fetch artifacts with error handling"""
    try:
        return fetch_artifacts(page=page)
    except requests.exceptions.RequestException as e:
        st.error(f"API Error: {e}")
        return None
    except Exception as e:
        st.error(f"Unexpected error: {e}")
        return None
```

### Caching for Performance

```python
@st.cache_data(ttl=3600)
def cached_query(query_text):
    """Cache query results for 1 hour"""
    return execute_query(query_text, conn)
```

## Troubleshooting

**API Key Issues**: Ensure `HARVARD_API_KEY` is set in `.env` and the key is valid.

**Database Connection Errors**: Verify database credentials and ensure the database server is accessible.

**Empty Results**: Check if the API response contains data; use `hasimage=1` parameter to filter artifacts with images.

**Memory Issues**: Process data in smaller batches instead of loading all artifacts at once.

**Slow Queries**: Add indexes on frequently queried columns like `culture`, `century`, and `department`.

```sql
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_century ON artifactmetadata(century);
CREATE INDEX idx_department ON artifactmetadata(department);
```
