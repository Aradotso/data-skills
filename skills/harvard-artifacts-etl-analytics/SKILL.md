---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data analytics app with Harvard Art Museums API
  - set up streamlit dashboard for museum artifacts
  - extract and analyze Harvard museum collection data
  - build SQL analytics for art museum API data
  - create visualization pipeline for Harvard artifacts
  - develop end-to-end data engineering app with museum API
  - transform Harvard Art Museums JSON to SQL tables
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It covers ETL pipeline development, SQL database design, analytical queries, and interactive Streamlit dashboards with Plotly visualizations.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based UI with Plotly visualizations
- **Database Design**: Normalized schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# Create database connection
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Create tables
def create_tables():
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            department VARCHAR(255),
            dated VARCHAR(255),
            period VARCHAR(255),
            technique VARCHAR(500),
            medium VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            division VARCHAR(255),
            primaryimageurl TEXT,
            verificationlevel INT,
            totalpageviews INT,
            totaluniquepageviews INT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            mediaid INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            baseimageurl TEXT,
            alttext TEXT,
            height INT,
            width INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            colorid INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    conn.commit()
    cursor.close()
    conn.close()

create_tables()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_artifacts=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_artifacts:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_artifacts]

# Usage
artifacts_data = fetch_artifacts(500)
print(f"Fetched {len(artifacts_data)} artifacts")
```

### Transform: Normalize JSON to Relational Tables

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into normalized dataframes"""
    
    # Metadata DataFrame
    metadata_records = []
    for artifact in artifacts:
        metadata_records.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'period': artifact.get('period'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions'),
            'creditline': artifact.get('creditline'),
            'division': artifact.get('division'),
            'primaryimageurl': artifact.get('primaryimageurl'),
            'verificationlevel': artifact.get('verificationlevel', 0),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
    
    # Media DataFrame
    media_records = []
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        images = artifact.get('images', [])
        for img in images:
            media_records.append({
                'objectid': objectid,
                'baseimageurl': img.get('baseimageurl'),
                'alttext': img.get('alttext'),
                'height': img.get('height'),
                'width': img.get('width')
            })
    
    # Colors DataFrame
    color_records = []
    for artifact in artifacts:
        objectid = artifact.get('objectid')
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'objectid': objectid,
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

# Usage
df_metadata, df_media, df_colors = transform_artifacts(artifacts_data)
```

### Load: Insert into SQL Database

```python
def load_to_database(df_metadata, df_media, df_colors):
    """Batch insert dataframes into SQL database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (objectid, title, culture, century, classification, department, 
             dated, period, technique, medium, dimensions, creditline, 
             division, primaryimageurl, verificationlevel, totalpageviews, 
             totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (objectid, baseimageurl, alttext, height, width)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    print("Data loaded successfully!")

# Execute ETL
load_to_database(df_metadata, df_media, df_colors)
```

## SQL Analytics Queries

### Sample Analytical Queries

```python
def get_analytics_queries():
    """20 predefined analytical queries"""
    return {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 15
        """,
        
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
            LIMIT 10
        """,
        
        "Department Distribution": """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        
        "Most Popular Artifacts (Page Views)": """
            SELECT title, totalpageviews, culture, century
            FROM artifactmetadata
            WHERE totalpageviews > 0
            ORDER BY totalpageviews DESC
            LIMIT 20
        """,
        
        "Color Analysis": """
            SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """,
        
        "Media Availability": """
            SELECT 
                COUNT(DISTINCT m.objectid) as artifacts_with_media,
                COUNT(*) as total_media_items,
                AVG(m.height) as avg_height,
                AVG(m.width) as avg_width
            FROM artifactmedia m
        """,
        
        "Classification by Period": """
            SELECT classification, period, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL AND period IS NOT NULL
            GROUP BY classification, period
            ORDER BY count DESC
            LIMIT 20
        """,
        
        "Top Techniques Used": """
            SELECT technique, COUNT(*) as count
            FROM artifactmetadata
            WHERE technique IS NOT NULL
            GROUP BY technique
            ORDER BY count DESC
            LIMIT 15
        """
    }

def execute_query(query_name, queries):
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    query = queries[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    return df

# Usage
queries = get_analytics_queries()
results = execute_query("Artifacts by Culture", queries)
print(results)
```

## Streamlit Dashboard

### Main Application

```python
import streamlit as st
import plotly.express as px
import pandas as pd

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for navigation
page = st.sidebar.selectbox("Navigate", ["ETL Pipeline", "SQL Analytics", "Visualizations"])

if page == "ETL Pipeline":
    st.header("Data Collection & ETL")
    
    num_artifacts = st.number_input("Number of artifacts to collect", 
                                     min_value=10, max_value=1000, value=100)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_artifacts)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            load_to_database(df_metadata, df_media, df_colors)
            st.success("Data loaded to database!")
        
        # Display sample data
        st.subheader("Sample Metadata")
        st.dataframe(df_metadata.head(10))

elif page == "SQL Analytics":
    st.header("SQL Query Analytics")
    
    queries = get_analytics_queries()
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        results = execute_query(selected_query, queries)
        st.subheader("Query Results")
        st.dataframe(results)
        
        # Auto-generate visualization
        if len(results.columns) >= 2:
            st.subheader("Visualization")
            fig = px.bar(results, x=results.columns[0], y=results.columns[1],
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

elif page == "Visualizations":
    st.header("Interactive Data Visualizations")
    
    # Culture distribution
    query = "SELECT culture, COUNT(*) as count FROM artifactmetadata WHERE culture IS NOT NULL GROUP BY culture ORDER BY count DESC LIMIT 10"
    df = pd.read_sql(query, get_db_connection())
    
    fig = px.bar(df, x='culture', y='count', 
                 title="Top 10 Cultures in Collection",
                 color='count', color_continuous_scale='viridis')
    st.plotly_chart(fig, use_container_width=True)
    
    # Century timeline
    query = "SELECT century, COUNT(*) as count FROM artifactmetadata WHERE century IS NOT NULL GROUP BY century"
    df = pd.read_sql(query, get_db_connection())
    
    fig = px.line(df, x='century', y='count',
                  title="Artifacts Distribution by Century")
    st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(num_artifacts, delay=0.5):
    """Fetch artifacts with rate limiting"""
    artifacts = []
    batch_size = 100
    
    for i in range(0, num_artifacts, batch_size):
        batch = fetch_artifacts_batch(i, batch_size)
        artifacts.extend(batch)
        time.sleep(delay)  # Respect API rate limits
    
    return artifacts
```

### Error Handling in ETL

```python
def safe_etl_pipeline(num_artifacts):
    """ETL pipeline with error handling"""
    try:
        # Extract
        artifacts = fetch_artifacts(num_artifacts)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        
        # Validate
        if df_metadata.empty:
            raise ValueError("No metadata to load")
        
        # Load
        load_to_database(df_metadata, df_media, df_colors)
        
        return True, "ETL completed successfully"
    
    except Exception as e:
        return False, f"ETL failed: {str(e)}"

success, message = safe_etl_pipeline(100)
print(message)
```

### Incremental Data Loading

```python
def get_existing_object_ids():
    """Get already loaded artifact IDs"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT objectid FROM artifactmetadata")
    existing_ids = set(row[0] for row in cursor.fetchall())
    cursor.close()
    conn.close()
    return existing_ids

def incremental_load(new_artifacts):
    """Load only new artifacts"""
    existing_ids = get_existing_object_ids()
    new_records = [a for a in new_artifacts if a.get('objectid') not in existing_ids]
    
    if new_records:
        df_metadata, df_media, df_colors = transform_artifacts(new_records)
        load_to_database(df_metadata, df_media, df_colors)
        print(f"Loaded {len(new_records)} new artifacts")
    else:
        print("No new artifacts to load")
```

## Troubleshooting

### API Key Issues

```python
# Verify API key is loaded
import os
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')

if not api_key:
    raise ValueError("HARVARD_API_KEY not found in environment variables")

# Test API connection
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': api_key, 'size': 1}
)
print(f"API Status: {response.status_code}")
```

### Database Connection Issues

```python
# Test database connection
try:
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT 1")
    result = cursor.fetchone()
    print("Database connection successful")
    cursor.close()
    conn.close()
except Exception as e:
    print(f"Database connection failed: {e}")
```

### Memory Management for Large Datasets

```python
def batch_etl(total_artifacts, batch_size=100):
    """Process large datasets in batches"""
    for offset in range(0, total_artifacts, batch_size):
        print(f"Processing batch {offset}-{offset+batch_size}")
        
        artifacts = fetch_artifacts_batch(offset, batch_size)
        df_metadata, df_media, df_colors = transform_artifacts(artifacts)
        load_to_database(df_metadata, df_media, df_colors)
        
        # Clear memory
        del artifacts, df_metadata, df_media, df_colors

batch_etl(1000, batch_size=100)
```

### Query Performance Optimization

```python
# Add indexes for frequently queried columns
def create_indexes():
    conn = get_db_connection()
    cursor = conn.cursor()
    
    cursor.execute("CREATE INDEX idx_culture ON artifactmetadata(culture)")
    cursor.execute("CREATE INDEX idx_century ON artifactmetadata(century)")
    cursor.execute("CREATE INDEX idx_department ON artifactmetadata(department)")
    cursor.execute("CREATE INDEX idx_objectid_media ON artifactmedia(objectid)")
    cursor.execute("CREATE INDEX idx_objectid_colors ON artifactcolors(objectid)")
    
    conn.commit()
    cursor.close()
    conn.close()
    print("Indexes created successfully")
```

This skill provides comprehensive guidance for building production-ready ETL pipelines and analytics dashboards using the Harvard Art Museums API.
