---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifacts
  - create analytics dashboard with Streamlit and SQL
  - process Harvard Art Museums API with pagination
  - transform nested JSON artifacts into relational database
  - visualize museum collection data with Plotly
  - set up TiDB Cloud for artifact data
  - query Harvard artifacts database
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering & Analytics App:
- **Extracts** artifact data from the Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables (metadata, media, colors)
- **Loads** data into MySQL/TiDB Cloud databases with optimized batch inserts
- **Analyzes** data using 20+ predefined SQL queries for insights
- **Visualizes** results through interactive Plotly charts in a Streamlit dashboard

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required packages:**
```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### API Key Setup

1. Get a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=artifacts_db
```

### Database Schema

The project uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    technique VARCHAR(500),
    url TEXT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    renditionnumber VARCHAR(50),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent DECIMAL(5,2),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Key Components

### 1. API Data Extraction

**Fetch artifacts with pagination:**

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100, page_size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    
    Args:
        num_records: Total number of records to fetch
        page_size: Records per API call (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    all_artifacts = []
    page = 1
    
    while len(all_artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': page_size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code != 200:
            print(f"Error: {response.status_code}")
            break
            
        data = response.json()
        records = data.get('records', [])
        
        if not records:
            break
            
        all_artifacts.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts[:num_records]
```

### 2. ETL Pipeline - Transform

**Extract nested data into separate tables:**

```python
import pandas as pd

def transform_artifacts(artifacts):
    """
    Transform raw API data into normalized DataFrames
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'technique': artifact.get('technique', '')[:500],
            'url': artifact.get('url', '')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        if artifact.get('images'):
            for img in artifact['images']:
                media = {
                    'objectid': artifact.get('objectid'),
                    'baseimageurl': img.get('baseimageurl', ''),
                    'iiifbaseuri': img.get('iiifbaseuri', ''),
                    'renditionnumber': img.get('renditionnumber', '')
                }
                media_list.append(media)
        
        # Extract color data
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_entry = {
                    'objectid': artifact.get('objectid'),
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'percent': color.get('percent', 0.0)
                }
                colors_list.append(color_entry)
    
    metadata_df = pd.DataFrame(metadata_list)
    media_df = pd.DataFrame(media_list)
    colors_df = pd.DataFrame(colors_list)
    
    return metadata_df, media_df, colors_df
```

### 3. Load Data to SQL

**Batch insert with MySQL connector:**

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load transformed data into MySQL/TiDB Cloud database
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata
        metadata_query = """
            INSERT IGNORE INTO artifactmetadata 
            (objectid, title, culture, century, classification, department, dated, technique, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        """
        metadata_values = metadata_df.values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia (objectid, baseimageurl, iiifbaseuri, renditionnumber)
            VALUES (%s, %s, %s, %s)
        """
        media_values = media_df.values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        colors_query = """
            INSERT INTO artifactcolors (objectid, color, spectrum, percent)
            VALUES (%s, %s, %s, %s)
        """
        colors_values = colors_df.values.tolist()
        cursor.executemany(colors_query, colors_values)
        
        connection.commit()
        print(f"Inserted {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 4. Analytics Queries

**Sample analytical queries:**

```python
ANALYTICAL_QUERIES = {
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
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Most Common Colors": """
        SELECT color, COUNT(*) as usage_count, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Media": """
        SELECT am.title, am.culture, am.century, COUNT(media.id) as image_count
        FROM artifactmetadata am
        JOIN artifactmedia media ON am.objectid = media.objectid
        GROUP BY am.objectid, am.title, am.culture, am.century
        HAVING image_count > 0
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Color Diversity by Artifact": """
        SELECT am.title, am.culture, COUNT(DISTINCT ac.color) as unique_colors
        FROM artifactmetadata am
        JOIN artifactcolors ac ON am.objectid = ac.objectid
        GROUP BY am.objectid, am.title, am.culture
        ORDER BY unique_colors DESC
        LIMIT 15
    """
}

def run_query(query):
    """Execute SQL query and return results as DataFrame"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(query, connection)
    connection.close()
    return df
```

### 5. Streamlit Dashboard

**Main application structure:**

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Artifacts Analytics Dashboard")
    st.write("Explore Harvard Art Museums collection data")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Choose a page",
        ["Data Collection", "Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        st.header("📥 Collect Artifacts Data")
        
        num_records = st.number_input(
            "Number of records to fetch",
            min_value=10,
            max_value=1000,
            value=100
        )
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_artifacts(num_records)
                
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(artifacts)
                
            with st.spinner("Loading to database..."):
                load_to_database(metadata_df, media_df, colors_df)
                
            st.success(f"Successfully loaded {len(metadata_df)} artifacts!")
    
    elif page == "Analytics":
        st.header("📊 SQL Analytics")
        
        query_name = st.selectbox(
            "Select a query",
            list(ANALYTICAL_QUERIES.keys())
        )
        
        if st.button("Run Query"):
            query = ANALYTICAL_QUERIES[query_name]
            results_df = run_query(query)
            
            st.subheader("Query Results")
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) == 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(url, params, delay=0.5):
    """Fetch data with rate limiting to avoid API throttling"""
    response = requests.get(url, params=params)
    time.sleep(delay)
    return response
```

### Error Handling in ETL

```python
def safe_transform(artifact, field, default='', max_length=None):
    """Safely extract and truncate field from artifact"""
    value = artifact.get(field, default)
    if max_length and value:
        value = str(value)[:max_length]
    return value
```

### Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="artifact_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD'),
    database=os.getenv('DB_NAME')
)

def get_connection():
    return db_pool.get_connection()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open at http://localhost:8501
```

## Troubleshooting

**API Key Issues:**
- Ensure `HARVARD_API_KEY` is set in `.env`
- Verify key is valid at Harvard Art Museums API portal
- Check for rate limiting (max 2500 requests/day for free tier)

**Database Connection Errors:**
- Verify all `DB_*` environment variables are set correctly
- For TiDB Cloud, ensure port is 4000 (not 3306)
- Check firewall rules allow connections from your IP

**Missing Data in Tables:**
- Use `INSERT IGNORE` for metadata to handle duplicates
- Check foreign key constraints are properly set
- Verify data transformation didn't filter out records

**Streamlit Performance:**
- Cache database connections with `@st.cache_resource`
- Limit query result sizes for large datasets
- Use pagination for displaying large tables

**Memory Issues:**
- Fetch artifacts in smaller batches (50-100 at a time)
- Clear DataFrames after loading to database
- Use generator functions for large datasets
