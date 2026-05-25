---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit, SQL, and Plotly
triggers:
  - build a data pipeline from Harvard Art Museums API
  - create an ETL process for museum artifact data
  - set up analytics dashboard for Harvard artifacts
  - query Harvard Art Museums data with SQL
  - visualize museum collection data with Streamlit
  - extract and transform Harvard API data
  - build end-to-end data engineering pipeline
  - analyze art museum collection data
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for visualizing museum artifact data.

## What It Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Collects artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact data into relational SQL tables
- **Database Design**: Structures data into `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: Executes 20+ predefined analytical queries on artifact collections
- **Interactive Visualization**: Displays query results using Plotly charts in Streamlit dashboards

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

Get your Harvard Art Museums API key from https://www.harvardartmuseums.org/collections/api

Store credentials in environment variables or `.env` file:

```bash
# .env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Database Setup

Create MySQL/TiDB database with required tables:

```sql
CREATE DATABASE harvard_artifacts;

USE harvard_artifacts;

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
    description TEXT,
    accessionyear INT
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### ETL Pipeline

**Extract artifacts from API:**

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Collect multiple pages
def collect_all_artifacts(total_pages=5):
    """Paginate through API results"""
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(page=page)
        all_artifacts.extend(data.get('records', []))
        print(f"Collected page {page}: {len(data.get('records', []))} artifacts")
    
    return all_artifacts
```

**Transform nested JSON to relational format:**

```python
import pandas as pd

def transform_artifacts(artifacts):
    """Transform API response to structured DataFrames"""
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'dated': artifact.get('dated'),
            'description': artifact.get('description'),
            'accessionyear': artifact.get('accessionyear')
        }
        metadata_records.append(metadata)
        
        # Extract media
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': primary_image,
                'iiifbaseuri': artifact.get('iiifbaseuri')
            }
            media_records.append(media)
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_record = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            }
            color_records.append(color_record)
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )
```

**Load data into SQL:**

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Batch insert DataFrames into SQL tables"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, division, dated, description, accessionyear)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors (artifact_id, color, percentage)
                VALUES (%s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### Streamlit Analytics Dashboard

**Main application structure:**

```python
import streamlit as st
import plotly.express as px

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
st.sidebar.header("Analytics Queries")

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
        ORDER BY century
    """,
    "Department Distribution": """
        SELECT department, COUNT(*) as count 
        FROM artifactmetadata 
        GROUP BY department 
        ORDER BY count DESC
    """,
    "Artifacts with Images": """
        SELECT 
            CASE WHEN m.baseimageurl IS NOT NULL THEN 'With Image' 
                 ELSE 'No Image' END as has_image,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY has_image
    """,
    "Top Colors Used": """
        SELECT color, SUM(percentage) as total_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY total_percentage DESC
        LIMIT 10
    """
}

selected_query = st.sidebar.selectbox("Select Analysis", list(queries.keys()))

# Execute query
def run_query(query):
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df

if st.button("Run Analysis"):
    with st.spinner("Executing query..."):
        result_df = run_query(queries[selected_query])
        
        st.subheader("Query Results")
        st.dataframe(result_df)
        
        # Auto-generate visualization
        if len(result_df.columns) == 2:
            fig = px.bar(
                result_df, 
                x=result_df.columns[0], 
                y=result_df.columns[1],
                title=selected_query
            )
            st.plotly_chart(fig)
```

### Complete ETL Workflow

```python
def run_etl_pipeline():
    """Execute complete ETL pipeline"""
    st.header("🔄 ETL Pipeline")
    
    pages = st.number_input("Number of pages to collect", min_value=1, max_value=100, value=5)
    
    if st.button("Start ETL Process"):
        # Extract
        with st.spinner("Extracting data from API..."):
            artifacts = collect_all_artifacts(total_pages=pages)
            st.success(f"Extracted {len(artifacts)} artifacts")
        
        # Transform
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
            st.write(f"Metadata: {len(metadata_df)} rows")
            st.write(f"Media: {len(media_df)} rows")
            st.write(f"Colors: {len(colors_df)} rows")
        
        # Load
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded to database")
```

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Common Patterns

### Rate Limiting API Calls

```python
import time

def fetch_with_rate_limit(page, delay=0.5):
    """Add delay between API calls"""
    result = fetch_artifacts(page=page)
    time.sleep(delay)
    return result
```

### Error Handling

```python
def safe_api_call(page):
    """Handle API errors gracefully"""
    try:
        return fetch_artifacts(page=page)
    except requests.exceptions.RequestException as e:
        st.error(f"API error on page {page}: {e}")
        return None
    except Exception as e:
        st.error(f"Unexpected error: {e}")
        return None
```

## Troubleshooting

**API Key Issues:**
- Verify key is active at Harvard Art Museums portal
- Check `.env` file is loaded: `load_dotenv()`
- Ensure no extra whitespace in environment variables

**Database Connection Errors:**
- Confirm MySQL/TiDB service is running
- Check firewall rules for database port (default 3306)
- Verify credentials in environment variables

**Streamlit Performance:**
- Use `@st.cache_data` for query results
- Limit result set sizes with SQL LIMIT clauses
- Implement pagination for large datasets

**Memory Issues:**
- Process data in smaller batches
- Use SQL queries instead of loading full tables
- Clear unused DataFrames with `del df`
