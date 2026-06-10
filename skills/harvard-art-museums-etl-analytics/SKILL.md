---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API, ETL workflows, SQL analytics, and Streamlit visualizations
triggers:
  - build an ETL pipeline for museum artifact data
  - create analytics dashboard with Harvard Art Museums API
  - set up SQL database for art collections data
  - extract and transform Harvard museum artifacts
  - visualize museum collection data with Streamlit
  - analyze artifact metadata and color patterns
  - create data engineering pipeline for art collections
  - query and visualize museum API data
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytics queries, and interactive Streamlit visualizations for museum artifact collections.

## What This Project Does

The Harvard Artifacts Collection application provides:
- **API Integration**: Secure data collection from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract artifact metadata, transform nested JSON to relational format, load into SQL databases
- **SQL Analytics**: 20+ predefined analytical queries for insights on artifacts, cultures, centuries, colors, and media
- **Interactive Visualization**: Streamlit dashboard with Plotly charts for real-time data exploration

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites

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

### Environment Configuration

Create a `.env` file with your credentials:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

Get your Harvard API key from: https://harvardartmuseums.org/collections/api

## Database Setup

### SQL Schema Design

The application uses three main tables with foreign key relationships:

```sql
-- Artifact Metadata Table
CREATE TABLE IF NOT EXISTS artifactmetadata (
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
    objectnumber VARCHAR(100),
    url VARCHAR(500),
    verificationlevel INT,
    primaryimageurl TEXT
);

-- Artifact Media Table
CREATE TABLE IF NOT EXISTS artifactmedia (
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE IF NOT EXISTS artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

### Start Streamlit Dashboard

```bash
streamlit run app.py
```

The dashboard opens at `http://localhost:8501`

### Application Workflow

1. **Configure API Key**: Enter your Harvard API key in the sidebar
2. **Collect Data**: Specify number of artifacts to fetch (default: 100)
3. **ETL Process**: Extract, transform, and load data into SQL database
4. **Run Analytics**: Select from 20+ predefined SQL queries
5. **Visualize Results**: View interactive charts and data tables

## ETL Pipeline Implementation

### Extract: API Data Collection

```python
import requests
import pandas as pd

def fetch_artifacts(api_key, num_artifacts=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # API limit per page
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': min(size, num_artifacts - len(artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) < size:
                break  # No more data
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]
```

### Transform: Data Cleaning and Structuring

```python
def transform_artifacts(raw_data):
    """
    Transform nested JSON into flat relational format
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'division': artifact.get('division', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'period': artifact.get('period', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'objectnumber': artifact.get('objectnumber', '')[:100],
            'url': artifact.get('url', '')[:500],
            'verificationlevel': artifact.get('verificationlevel', 0),
            'primaryimageurl': artifact.get('primaryimageurl', '')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'baseimageurl': image.get('baseimageurl', ''),
                'format': image.get('format', ''),
                'height': image.get('height', 0),
                'width': image.get('width', 0)
            }
            media_list.append(media)
        
        # Extract color information
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Batch Insert into SQL

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load transformed data into MySQL database with batch inserts
    """
    try:
        connection = mysql.connector.connect(**db_config)
        cursor = connection.cursor()
        
        # Insert metadata (batch)
        metadata_sql = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             division, dated, period, technique, objectnumber, url, 
             verificationlevel, primaryimageurl)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        cursor.executemany(metadata_sql, metadata_df.values.tolist())
        
        # Insert media (batch)
        if not media_df.empty:
            media_sql = """
                INSERT INTO artifactmedia 
                (artifact_id, media_type, baseimageurl, format, height, width)
                VALUES (%s, %s, %s, %s, %s, %s)
            """
            cursor.executemany(media_sql, media_df.values.tolist())
        
        # Insert colors (batch)
        if not colors_df.empty:
            colors_sql = """
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """
            cursor.executemany(colors_sql, colors_df.values.tolist())
        
        connection.commit()
        print(f"✅ Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"❌ Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
# Query 1: Artifacts by Culture
query_by_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 15;
"""

# Query 2: Artifacts by Century
query_by_century = """
    SELECT century, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY artifact_count DESC;
"""

# Query 3: Color Distribution
query_color_distribution = """
    SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percentage
    FROM artifactcolors
    WHERE color IS NOT NULL
    GROUP BY color
    ORDER BY usage_count DESC
    LIMIT 10;
"""

# Query 4: Media Availability
query_media_stats = """
    SELECT 
        COUNT(DISTINCT am.id) as total_artifacts,
        COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
        ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as media_percentage
    FROM artifactmetadata am
    LEFT JOIN artifactmedia media ON am.id = media.artifact_id;
"""

# Query 5: Department-wise Classification
query_dept_classification = """
    SELECT department, classification, COUNT(*) as count
    FROM artifactmetadata
    WHERE department IS NOT NULL AND classification IS NOT NULL
    GROUP BY department, classification
    ORDER BY count DESC
    LIMIT 20;
"""
```

## Streamlit Dashboard Implementation

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
from dotenv import load_dotenv
import os

# Load environment variables
load_dotenv()

# Page configuration
st.set_page_config(
    page_title="Harvard Artifacts Analytics",
    page_icon="🏛️",
    layout="wide"
)

st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")

# Sidebar configuration
with st.sidebar:
    st.header("⚙️ Configuration")
    api_key = st.text_input("Harvard API Key", type="password", 
                            value=os.getenv("HARVARD_API_KEY", ""))
    num_artifacts = st.number_input("Number of Artifacts", 
                                    min_value=10, max_value=1000, 
                                    value=100, step=10)
    
    if st.button("🔄 Collect & Load Data"):
        with st.spinner("Fetching artifacts..."):
            # Run ETL pipeline
            raw_data = fetch_artifacts(api_key, num_artifacts)
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            
            db_config = {
                'host': os.getenv('DB_HOST'),
                'port': int(os.getenv('DB_PORT', 3306)),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            
            load_to_database(metadata_df, media_df, colors_df, db_config)
            st.success("✅ Data loaded successfully!")

# Analytics section
st.header("📊 SQL Analytics")

queries = {
    "Artifacts by Culture": query_by_culture,
    "Artifacts by Century": query_by_century,
    "Top Colors": query_color_distribution,
    "Media Statistics": query_media_stats,
    "Department Classification": query_dept_classification
}

selected_query = st.selectbox("Select Analysis", list(queries.keys()))

if st.button("Run Query"):
    connection = mysql.connector.connect(**db_config)
    df_result = pd.read_sql(queries[selected_query], connection)
    connection.close()
    
    # Display results
    st.dataframe(df_result, use_container_width=True)
    
    # Auto-generate visualization
    if len(df_result.columns) >= 2:
        fig = px.bar(df_result, 
                     x=df_result.columns[0], 
                     y=df_result.columns[1],
                     title=selected_query)
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Pattern 1: Rate-Limited API Calls

```python
import time

def fetch_with_rate_limit(api_key, num_artifacts, delay=0.5):
    """Fetch artifacts with rate limiting"""
    artifacts = []
    page = 1
    
    while len(artifacts) < num_artifacts:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'page': page, 'size': 100}
        )
        
        if response.status_code == 200:
            artifacts.extend(response.json().get('records', []))
            page += 1
            time.sleep(delay)  # Rate limiting
        else:
            break
    
    return artifacts[:num_artifacts]
```

### Pattern 2: Incremental Data Updates

```python
def get_latest_artifact_id(connection):
    """Get the latest artifact ID in database"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def fetch_new_artifacts(api_key, last_id):
    """Fetch only artifacts newer than last_id"""
    # Implementation depends on API filtering capabilities
    pass
```

### Pattern 3: Data Quality Checks

```python
def validate_artifact_data(df):
    """Validate data quality before loading"""
    issues = []
    
    # Check for required fields
    if df['id'].isnull().any():
        issues.append("Missing artifact IDs")
    
    # Check for duplicates
    if df['id'].duplicated().any():
        issues.append(f"{df['id'].duplicated().sum()} duplicate IDs found")
    
    # Check string lengths
    if (df['title'].str.len() > 500).any():
        issues.append("Title exceeds max length")
    
    return issues
```

## Troubleshooting

### API Key Issues

```python
# Test API connection
def test_api_connection(api_key):
    response = requests.get(
        "https://api.harvardartmuseums.org/object",
        params={'apikey': api_key, 'size': 1}
    )
    if response.status_code == 401:
        return "❌ Invalid API key"
    elif response.status_code == 200:
        return "✅ API connection successful"
    else:
        return f"⚠️ Unexpected status: {response.status_code}"
```

### Database Connection Errors

```python
# Debug database connection
def test_db_connection(db_config):
    try:
        connection = mysql.connector.connect(**db_config)
        if connection.is_connected():
            print("✅ Database connected")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE();")
            print(f"Current database: {cursor.fetchone()[0]}")
            connection.close()
    except Error as e:
        print(f"❌ Connection failed: {e}")
        # Common fixes:
        # - Check DB_HOST and DB_PORT
        # - Verify DB_USER has proper permissions
        # - Ensure database exists
```

### Empty Results

```python
# Check data availability
def diagnose_empty_results(connection):
    cursor = connection.cursor()
    
    cursor.execute("SELECT COUNT(*) FROM artifactmetadata")
    metadata_count = cursor.fetchone()[0]
    print(f"Metadata records: {metadata_count}")
    
    cursor.execute("SELECT COUNT(*) FROM artifactmedia")
    media_count = cursor.fetchone()[0]
    print(f"Media records: {media_count}")
    
    cursor.execute("SELECT COUNT(*) FROM artifactcolors")
    colors_count = cursor.fetchone()[0]
    print(f"Color records: {colors_count}")
    
    if metadata_count == 0:
        print("⚠️ Run ETL pipeline to populate data")
```

### Performance Optimization

```python
# Use connection pooling for better performance
from mysql.connector import pooling

connection_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    **db_config
)

def execute_query_pooled(query):
    connection = connection_pool.get_connection()
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Key Takeaways

- Always use environment variables for sensitive credentials
- Implement batch inserts for large datasets (significant performance gain)
- Handle API pagination and rate limiting properly
- Validate data before loading to database
- Use connection pooling for multiple queries
- Cache Streamlit results with `@st.cache_data` for better UX
