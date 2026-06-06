---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with MySQL/TiDB and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - show me how to use the Harvard Art Museums API with SQL analytics
  - create a data engineering pipeline with Streamlit visualization
  - how to fetch and analyze artifact data from Harvard museums
  - build an end-to-end data pipeline with API and database
  - set up Harvard Art Museums data collection and analytics
  - create interactive dashboards for museum artifact data
  - implement batch ETL processing for art museum collections
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational tables, loading into SQL databases (MySQL/TiDB), and visualizing analytics through interactive Streamlit dashboards.

## What It Does

- **API Data Collection**: Fetches artifact metadata, media, and color information from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (artifacts, media, colors)
- **SQL Storage**: Batch inserts data into MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for insights (distribution by culture, century, department, color analysis)
- **Visualization**: Renders interactive Plotly charts in Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
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

# MySQL/TiDB Connection
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Setup

```python
import mysql.connector
from mysql.connector import Error

def create_database_schema():
    """Initialize database tables with proper schema"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=os.getenv('DB_PORT'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Create artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            medium VARCHAR(500),
            technique VARCHAR(500),
            dimensions VARCHAR(500),
            creditline TEXT,
            accessionyear INT,
            url VARCHAR(500)
        )
    """)
    
    # Create media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            mediaid INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Create colors table
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
    
    connection.commit()
    cursor.close()
    connection.close()
```

## API Integration

### Fetching Artifacts with Pagination

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        page: Page number (starts at 1)
        size: Number of records per page (max 100)
    
    Returns:
        dict: API response with records and pagination info
    """
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
        raise Exception(f"API Error {response.status_code}: {response.text}")

# Batch collection with rate limiting
import time

def collect_all_artifacts(total_records=1000, batch_size=100):
    """Collect artifacts across multiple pages"""
    all_records = []
    pages = (total_records // batch_size) + 1
    
    for page in range(1, pages + 1):
        print(f"Fetching page {page}/{pages}...")
        data = fetch_artifacts(page=page, size=batch_size)
        all_records.extend(data.get('records', []))
        
        # Rate limiting: 1 request per second
        time.sleep(1)
        
        if len(all_records) >= total_records:
            break
    
    return all_records[:total_records]
```

## ETL Pipeline

### Extract & Transform

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into relational tables
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'dimensions': artifact.get('dimensions', '')[:500],
            'creditline': artifact.get('creditline', ''),
            'accessionyear': artifact.get('accessionyear'),
            'url': artifact.get('url', '')[:500]
        }
        metadata_records.append(metadata)
        
        # Extract media (images)
        if artifact.get('primaryimageurl'):
            media = {
                'objectid': artifact.get('objectid'),
                'baseimageurl': artifact.get('primaryimageurl', '')[:500],
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri', '')[:500] if artifact.get('images') else ''
            }
            media_records.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_record = {
                'objectid': artifact.get('objectid'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'hue': color.get('hue', '')[:50],
                'percent': color.get('percent', 0.0)
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
def batch_insert_artifacts(metadata_df, media_df, colors_df):
    """Batch insert DataFrames into MySQL/TiDB"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT')),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = connection.cursor()
    
    # Insert metadata
    metadata_query = """
        INSERT IGNORE INTO artifactmetadata 
        (objectid, title, culture, century, classification, department, 
         dated, medium, technique, dimensions, creditline, accessionyear, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    if not media_df.empty:
        media_query = """
            INSERT INTO artifactmedia (objectid, baseimageurl, iiifbaseuri)
            VALUES (%s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    if not colors_df.empty:
        colors_query = """
            INSERT INTO artifactcolors (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(colors_query, colors_df.values.tolist())
    
    connection.commit()
    cursor.close()
    connection.close()
    
    print(f"Inserted {len(metadata_df)} artifacts, {len(media_df)} media, {len(colors_df)} colors")
```

## Analytics Queries

### Example SQL Queries

```python
# Query: Artifacts by Culture
query_by_culture = """
    SELECT culture, COUNT(*) as count
    FROM artifactmetadata
    WHERE culture IS NOT NULL AND culture != ''
    GROUP BY culture
    ORDER BY count DESC
    LIMIT 20
"""

# Query: Artifacts by Century
query_by_century = """
    SELECT century, COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != ''
    GROUP BY century
    ORDER BY century
"""

# Query: Color Distribution
query_color_distribution = """
    SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
    FROM artifactcolors
    GROUP BY color
    ORDER BY frequency DESC
    LIMIT 15
"""

# Query: Department Distribution
query_departments = """
    SELECT department, COUNT(*) as artifact_count
    FROM artifactmetadata
    WHERE department IS NOT NULL
    GROUP BY department
    ORDER BY artifact_count DESC
"""

# Query: Media Availability
query_media_stats = """
    SELECT 
        (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
        COUNT(DISTINCT m.objectid) as artifacts_with_images,
        ROUND(COUNT(DISTINCT m.objectid) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
    FROM artifactmedia m
"""

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT')),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

## Streamlit Dashboard

### Basic Dashboard Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Analysis",
        ["Overview", "Culture Analysis", "Color Insights", "Department Stats", "Custom Query"]
    )
    
    if page == "Culture Analysis":
        st.header("Artifacts by Culture")
        
        df = execute_query(query_by_culture)
        
        # Display table
        st.dataframe(df, use_container_width=True)
        
        # Visualization
        fig = px.bar(
            df, 
            x='culture', 
            y='count',
            title="Top 20 Cultures in Collection",
            labels={'culture': 'Culture', 'count': 'Number of Artifacts'}
        )
        st.plotly_chart(fig, use_container_width=True)
    
    elif page == "Color Insights":
        st.header("Color Distribution Analysis")
        
        df = execute_query(query_color_distribution)
        
        col1, col2 = st.columns(2)
        
        with col1:
            fig = px.pie(
                df, 
                values='frequency', 
                names='color',
                title="Color Frequency Distribution"
            )
            st.plotly_chart(fig, use_container_width=True)
        
        with col2:
            fig = px.bar(
                df, 
                x='color', 
                y='avg_percent',
                title="Average Color Percentage",
                color='avg_percent'
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

### ETL Page in Streamlit

```python
def etl_page():
    st.header("🔄 ETL Pipeline Control")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_records = st.number_input("Number of records to fetch", min_value=10, max_value=10000, value=100)
    
    with col2:
        batch_size = st.number_input("Batch size", min_value=10, max_value=100, value=100)
    
    if st.button("▶️ Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            raw_data = collect_all_artifacts(total_records=num_records, batch_size=batch_size)
            st.success(f"✅ Fetched {len(raw_data)} records")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(raw_data)
            st.success(f"✅ Transformed into {len(metadata_df)} metadata, {len(media_df)} media, {len(colors_df)} color records")
        
        with st.spinner("Loading into database..."):
            batch_insert_artifacts(metadata_df, media_df, colors_df)
            st.success("✅ Data loaded successfully!")
        
        # Display sample
        st.subheader("Sample Data")
        st.dataframe(metadata_df.head(10))
```

## Common Patterns

### Complete ETL Workflow

```python
def run_complete_etl(num_records=500):
    """Execute full ETL pipeline"""
    # Extract
    print("Step 1: Extracting data from API...")
    raw_data = collect_all_artifacts(total_records=num_records)
    
    # Transform
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Step 3: Loading to database...")
    batch_insert_artifacts(metadata_df, media_df, colors_df)
    
    print("✅ ETL Pipeline Complete!")
    return metadata_df, media_df, colors_df
```

### Error Handling

```python
def safe_api_fetch(page, size, retries=3):
    """API fetch with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(page=page, size=size)
        except Exception as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Rate Limiting**: Harvard API allows 2500 requests/day. Add `time.sleep(1)` between requests.

**Database Connection Issues**: Verify TiDB/MySQL host, port, and credentials. Use `DB_PORT=4000` for TiDB Cloud.

**Missing Data**: Not all artifacts have images or colors. Use `WHERE` clauses to filter NULL values in queries.

**Memory Issues**: Process data in smaller batches (100-500 records) instead of loading thousands at once.

**Duplicate Keys**: Use `INSERT IGNORE` for metadata table or handle updates with `ON DUPLICATE KEY UPDATE`.
