---
name: harvard-artifacts-etl-streamlit
description: Build ETL pipelines and analytics dashboards for the Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - build a data pipeline for Harvard Art Museums API
  - create an ETL process for museum artifacts data
  - set up analytics dashboard with Streamlit and SQL
  - extract and transform Harvard museum collection data
  - build artifact data engineering pipeline
  - create museum data visualization with Plotly
  - implement batch data loading for art collections
  - query and analyze Harvard artifacts database
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering pipelines using the Harvard Art Museums API. It covers ETL processes, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection app demonstrates a complete data engineering workflow:

1. **Extract**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
2. **Transform**: Convert nested JSON into relational tables (metadata, media, colors)
3. **Load**: Batch insert data into MySQL/TiDB Cloud databases
4. **Analyze**: Execute 20+ predefined SQL analytical queries
5. **Visualize**: Display interactive dashboards using Streamlit and Plotly

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
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
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Connect to database
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

# Create tables
cursor = conn.cursor()

# Artifact metadata table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accession_year INT,
    verification_level INT,
    PRIMARY KEY (id)
)
""")

# Artifact media table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

# Artifact colors table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
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
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Example: Fetch first 100 artifacts
data = fetch_artifacts(page=1, size=100)
total_records = data['info']['totalrecords']
artifacts = data['records']

print(f"Total artifacts available: {total_records}")
print(f"Fetched: {len(artifacts)} artifacts")
```

### Handling Pagination and Rate Limits

```python
import time

def fetch_all_artifacts(max_artifacts=1000, batch_size=100):
    """Fetch multiple pages with rate limiting"""
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < max_artifacts:
        try:
            data = fetch_artifacts(page=page, size=batch_size)
            artifacts = data['records']
            
            if not artifacts:
                break
                
            all_artifacts.extend(artifacts)
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
            page += 1
            time.sleep(0.5)  # Rate limiting - be respectful
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return all_artifacts[:max_artifacts]
```

## ETL Pipeline Implementation

### Extract and Transform

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform raw API data into structured DataFrames"""
    
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'accession_year': artifact.get('accessionyear'),
            'verification_level': artifact.get('verificationlevel', 0)
        }
        metadata_records.append(metadata)
        
        # Extract media
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media = {
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl'),
                    'media_type': 'image'
                }
                media_records.append(media)
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_record = {
                    'artifact_id': artifact.get('id'),
                    'color_hex': color.get('hex'),
                    'color_percent': color.get('percent')
                }
                color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

### Load to Database

```python
def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert DataFrames into SQL database"""
    
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
            (id, title, culture, century, classification, department, dated, accession_year, verification_level)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts to database")
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")

# Sidebar for configuration
st.sidebar.header("Configuration")

# Data Collection Section
st.header("📥 Data Collection")

num_artifacts = st.number_input("Number of artifacts to fetch", min_value=10, max_value=5000, value=100)

if st.button("Fetch & Load Data"):
    with st.spinner("Fetching artifacts from API..."):
        artifacts = fetch_all_artifacts(max_artifacts=num_artifacts)
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        load_to_database(metadata_df, media_df, colors_df)
        st.success(f"✅ Successfully loaded {len(artifacts)} artifacts!")

# Analytics Section
st.header("📊 SQL Analytics")

# Sample analytical queries
queries = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    "Top Colors Used": """
        SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    "Artifacts with Media": """
        SELECT 
            CASE WHEN media_count > 0 THEN 'With Media' ELSE 'Without Media' END as media_status,
            COUNT(*) as count
        FROM (
            SELECT m.id, COUNT(am.image_url) as media_count
            FROM artifactmetadata m
            LEFT JOIN artifactmedia am ON m.id = am.artifact_id
            GROUP BY m.id
        ) as media_summary
        GROUP BY media_status
    """
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df_result = pd.read_sql(queries[selected_query], conn)
    conn.close()
    
    st.subheader("Query Results")
    st.dataframe(df_result)
    
    # Auto-generate visualization
    if len(df_result.columns) == 2:
        fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                     title=selected_query)
        st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Error Handling in ETL

```python
def safe_etl_pipeline(num_artifacts=100):
    """ETL with error handling and logging"""
    try:
        # Extract
        artifacts = fetch_all_artifacts(max_artifacts=num_artifacts)
        
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate
        assert not metadata_df.empty, "Metadata is empty"
        
        # Load
        load_to_database(metadata_df, media_df, colors_df)
        
        return {
            'status': 'success',
            'artifacts_processed': len(artifacts),
            'metadata_records': len(metadata_df),
            'media_records': len(media_df),
            'color_records': len(colors_df)
        }
        
    except Exception as e:
        return {
            'status': 'error',
            'error_message': str(e)
        }
```

## Troubleshooting

**API Rate Limit Errors:**
- Add `time.sleep()` between requests
- Reduce batch size
- Use the API's built-in pagination properly

**Database Connection Issues:**
- Verify `.env` file exists and has correct credentials
- Check if database server is accessible
- Ensure database and tables are created

**Missing Data in Results:**
- Some artifacts may not have all fields (images, colors)
- Use `LEFT JOIN` in SQL queries
- Handle `None` values during transformation

**Streamlit Performance:**
- Cache database connections with `@st.cache_resource`
- Limit query result sizes
- Use pagination for large datasets

```python
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```
