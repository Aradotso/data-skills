---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build a data pipeline with the Harvard Art Museums API
  - create an ETL pipeline for museum artifact data
  - analyze Harvard Art Museums collection using SQL
  - build a Streamlit dashboard for museum artifacts
  - extract and transform data from Harvard Art Museums API
  - set up analytics for art museum collections
  - query Harvard artifacts database with SQL
  - visualize museum artifact data with Plotly
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates a complete data engineering workflow: extracting artifact data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases, running analytics queries, and visualizing results with interactive Streamlit dashboards.

## What It Does

- **Collects** artifact metadata, media, and color data from Harvard Art Museums API
- **Transforms** nested JSON into normalized relational tables
- **Loads** data into MySQL/TiDB Cloud with proper foreign key relationships
- **Analyzes** using 20+ predefined SQL queries
- **Visualizes** results with interactive Plotly charts in Streamlit

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

### API Key Setup

Get your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

Create a `.env` file:
```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Schema

The pipeline creates three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    division VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    creditline TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    mediatype VARCHAR(100),
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## ETL Pipeline Implementation

### Extract Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    size = 100  # Max per request
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page,
            'hasimage': 1  # Only get artifacts with images
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
    
    return artifacts[:num_records]
```

### Transform Data

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform nested JSON into relational dataframes"""
    
    # Metadata DataFrame
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'period': artifact.get('period', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'division': artifact.get('division', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'creditline': artifact.get('creditline', ''),
            'url': artifact.get('url', '')[:500]
        }
        metadata_records.append(metadata)
        
        # Extract media
        images = artifact.get('images', [])
        for img in images:
            media = {
                'objectid': artifact.get('objectid'),
                'mediatype': 'image',
                'baseimageurl': img.get('baseimageurl', '')[:500],
                'iiifbaseuri': img.get('iiifbaseuri', '')[:500]
            }
            media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'objectid': artifact.get('objectid'),
                'color': color.get('color', '')[:50],
                'spectrum': color.get('spectrum', '')[:50],
                'percentage': color.get('percent', 0.0)
            }
            color_records.append(color_data)
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load Data into Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_to_database(df_metadata, df_media, df_colors):
    """Batch insert dataframes into MySQL"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (objectid, title, culture, period, century, classification, 
             division, department, dated, creditline, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        cursor.executemany(metadata_query, df_metadata.values.tolist())
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia 
            (objectid, mediatype, baseimageurl, iiifbaseuri)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_query, df_media.values.tolist())
        
        # Insert colors
        color_query = """
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, percentage)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(color_query, df_colors.values.tolist())
        
        connection.commit()
        print(f"Loaded {len(df_metadata)} artifacts successfully")
        return True
        
    except Error as e:
        print(f"Database insert error: {e}")
        connection.rollback()
        return False
        
    finally:
        cursor.close()
        connection.close()
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Sample queries for the analytics dashboard
ANALYTICS_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as color_count, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY color_count DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT m.objectid, m.title, COUNT(a.id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia a ON m.objectid = a.objectid
        GROUP BY m.objectid, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND department != ''
        GROUP BY department
        ORDER BY count DESC
    """
}

def execute_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query execution error: {e}")
        return None
    finally:
        connection.close()
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Collection Analytics")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("📥 Collect Artifact Data")
        
        num_records = st.number_input(
            "Number of records to fetch",
            min_value=10,
            max_value=1000,
            value=100,
            step=10
        )
        
        if st.button("Fetch and Load Data"):
            with st.spinner("Fetching artifacts..."):
                artifacts = fetch_artifacts(num_records)
                st.success(f"Fetched {len(artifacts)} artifacts")
                
            with st.spinner("Transforming data..."):
                df_meta, df_media, df_colors = transform_artifacts(artifacts)
                st.success("Data transformed")
                
            with st.spinner("Loading to database..."):
                success = load_to_database(df_meta, df_media, df_colors)
                if success:
                    st.success("✅ Data loaded successfully!")
                    
                    # Show sample data
                    st.subheader("Sample Metadata")
                    st.dataframe(df_meta.head())
    
    elif page == "SQL Analytics":
        st.header("📊 SQL Analytics Dashboard")
        
        query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Run Query"):
            query = ANALYTICS_QUERIES[query_name]
            
            st.code(query, language="sql")
            
            with st.spinner("Executing query..."):
                df_result = execute_query(query)
                
            if df_result is not None and not df_result.empty:
                st.dataframe(df_result)
                
                # Auto-generate visualization
                if len(df_result.columns) >= 2:
                    fig = px.bar(
                        df_result,
                        x=df_result.columns[0],
                        y=df_result.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Handling API Rate Limits

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch data with exponential backoff"""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params)
            if response.status_code == 429:  # Rate limit
                wait_time = 2 ** attempt
                time.sleep(wait_time)
                continue
            return response
        except requests.RequestException as e:
            print(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(2 ** attempt)
    
    return None
```

### Data Validation

```python
def validate_artifact_data(df):
    """Validate required fields before loading"""
    required_columns = ['objectid', 'title']
    
    for col in required_columns:
        if col not in df.columns:
            raise ValueError(f"Missing required column: {col}")
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['objectid'])
    
    # Remove null objectids
    df = df[df['objectid'].notna()]
    
    return df
```

## Troubleshooting

**Connection errors:** Verify database credentials in `.env` and ensure the database is accessible.

**API key invalid:** Check that `HARVARD_API_KEY` is correctly set and valid at https://www.harvardartmuseums.org/collections/api

**Duplicate key errors:** The pipeline uses `ON DUPLICATE KEY UPDATE` for metadata to handle re-runs safely.

**Missing data in visualizations:** Ensure queries return non-empty results and column names match expected format.

**Streamlit not starting:** Check that all dependencies are installed and port 8501 is available.
