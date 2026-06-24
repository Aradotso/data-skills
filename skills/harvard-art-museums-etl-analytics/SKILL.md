---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums data
  - set up Harvard artifacts data engineering app
  - create analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums API data
  - build Streamlit app for art collection analytics
  - query Harvard museum artifacts with SQL
  - visualize art museum data with Plotly
  - implement data pipeline for museum collections
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application for the Harvard Art Museums API. It demonstrates ETL pipelines, SQL analytics, and interactive visualizations using Streamlit, Pandas, and Plotly.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into relational database schemas
- **Loads** structured data into MySQL/TiDB Cloud databases
- **Analyzes** data using predefined SQL queries
- **Visualizes** insights through interactive Plotly dashboards

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

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

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://docs.api.harvardartmuseums.org/
2. Register for a free API key
3. Add to `.env` file

## Database Schema

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    dated VARCHAR(200),
    url TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    image_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    hex_code VARCHAR(10),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Usage Patterns

### 1. ETL Pipeline Implementation

```python
import requests
import pandas as pd
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# API Configuration
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'

# Database Connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Extract: Fetch data from API
def extract_artifacts(page=1, size=100):
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size
    }
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Transform: Process JSON to DataFrames
def transform_artifacts(data):
    records = data.get('records', [])
    
    metadata_list = []
    media_list = []
    colors_list = []
    
    for record in records:
        # Metadata
        metadata_list.append({
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'period': record.get('period'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'dated': record.get('dated'),
            'url': record.get('url')
        })
        
        # Media
        for image in record.get('images', []):
            media_list.append({
                'artifact_id': record.get('id'),
                'media_type': 'image',
                'image_url': image.get('baseimageurl')
            })
        
        # Colors
        for color in record.get('colors', []):
            colors_list.append({
                'artifact_id': record.get('id'),
                'color': color.get('color'),
                'hex_code': color.get('hex'),
                'percentage': color.get('percent')
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

# Load: Insert into database
def load_to_database(metadata_df, media_df, colors_df):
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, department, division, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, media_type, image_url)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, hex_code, percentage)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()

# Complete ETL Pipeline
def run_etl(num_pages=5):
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        data = extract_artifacts(page=page)
        metadata_df, media_df, colors_df = transform_artifacts(data)
        load_to_database(metadata_df, media_df, colors_df)
        print(f"Page {page} completed.")
```

### 2. Streamlit Application Structure

```python
import streamlit as st
import pandas as pd
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

# Sidebar Navigation
page = st.sidebar.selectbox(
    "Navigation",
    ["Data Collection", "SQL Analytics", "Visualizations"]
)

if page == "Data Collection":
    st.title("🎨 Harvard Art Museums Data Collection")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    if st.button("Start ETL Pipeline"):
        with st.spinner("Running ETL..."):
            run_etl(num_pages=num_pages)
            st.success(f"Successfully processed {num_pages} pages!")

elif page == "SQL Analytics":
    st.title("📊 SQL Analytics Dashboard")
    
    # Predefined Queries
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
        "Media Availability": """
            SELECT 
                CASE WHEN COUNT(m.id) > 0 THEN 'Has Media' ELSE 'No Media' END as media_status,
                COUNT(*) as count
            FROM artifactmetadata a
            LEFT JOIN artifactmedia m ON a.id = m.artifact_id
            GROUP BY media_status
        """,
        "Top Colors": """
            SELECT color, COUNT(*) as count, AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = pd.read_sql(queries[selected_query], conn)
        conn.close()
        
        st.dataframe(df)
        
        # Auto-visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)
```

### 3. Advanced Analytics Queries

```python
# Artifacts without images
no_images_query = """
SELECT a.id, a.title, a.culture, a.century
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id
WHERE m.id IS NULL
LIMIT 100
"""

# Color distribution analysis
color_distribution_query = """
SELECT 
    c.color,
    COUNT(DISTINCT c.artifact_id) as artifact_count,
    AVG(c.percentage) as avg_percentage,
    MAX(c.percentage) as max_percentage
FROM artifactcolors c
GROUP BY c.color
HAVING artifact_count > 5
ORDER BY artifact_count DESC
"""

# Department-wise classification
department_classification_query = """
SELECT 
    department,
    classification,
    COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL AND classification IS NOT NULL
GROUP BY department, classification
ORDER BY department, count DESC
"""
```

### 4. Batch Processing with Pagination

```python
import time

def extract_all_artifacts(max_pages=100, delay=1):
    """Extract artifacts with rate limiting"""
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, max_pages + 1):
        try:
            data = extract_artifacts(page=page)
            
            if not data.get('records'):
                print(f"No more records at page {page}")
                break
            
            metadata_df, media_df, colors_df = transform_artifacts(data)
            all_metadata.append(metadata_df)
            all_media.append(media_df)
            all_colors.append(colors_df)
            
            print(f"Page {page}: {len(data['records'])} records")
            time.sleep(delay)  # Rate limiting
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return (
        pd.concat(all_metadata, ignore_index=True),
        pd.concat(all_media, ignore_index=True),
        pd.concat(all_colors, ignore_index=True)
    )
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting
```python
# Add retry logic with exponential backoff
from time import sleep

def extract_with_retry(page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return extract_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues
```python
# Test connection
def test_db_connection():
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        print("✓ Database connection successful")
        return True
    except Exception as e:
        print(f"✗ Database connection failed: {e}")
        return False
```

### Missing Data Handling
```python
# Safe data extraction
def safe_get(record, key, default=None):
    return record.get(key) if record.get(key) else default

# In transform function
metadata_list.append({
    'id': record.get('id'),
    'title': safe_get(record, 'title', 'Untitled'),
    'culture': safe_get(record, 'culture', 'Unknown'),
    # ... other fields
})
```

## Common Patterns

### Export Query Results to CSV
```python
def export_query_results(query, filename):
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    df.to_csv(filename, index=False)
    return df
```

### Create Interactive Dashboard
```python
# Multi-metric dashboard
col1, col2, col3 = st.columns(3)

with col1:
    total_artifacts = pd.read_sql("SELECT COUNT(*) as count FROM artifactmetadata", conn)
    st.metric("Total Artifacts", total_artifacts['count'][0])

with col2:
    artifacts_with_media = pd.read_sql(
        "SELECT COUNT(DISTINCT artifact_id) as count FROM artifactmedia", conn)
    st.metric("With Media", artifacts_with_media['count'][0])

with col3:
    unique_cultures = pd.read_sql(
        "SELECT COUNT(DISTINCT culture) as count FROM artifactmetadata", conn)
    st.metric("Unique Cultures", unique_cultures['count'][0])
```
