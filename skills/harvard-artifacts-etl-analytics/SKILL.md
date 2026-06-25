---
name: harvard-artifacts-etl-analytics
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API, featuring ETL operations, SQL analytics, and Streamlit visualization.
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering app with museum artifacts
  - extract and analyze Harvard museum collection data
  - set up a Streamlit dashboard for art museum analytics
  - query Harvard Art Museums API and store in SQL
  - build an analytics pipeline for cultural heritage data
  - create a museum artifacts data warehouse
  - visualize Harvard museum data with SQL queries
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates production-grade ETL pipelines, relational database design, SQL analytics, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Secure data collection from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media details, and color information
- **SQL Database**: Structured storage with proper relational design (artifacts, media, colors tables)
- **Analytics Queries**: 20+ predefined SQL queries for insights on culture, century, media, colors
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations

Architecture: `API → ETL → SQL → Analytics → Visualization`

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

### 1. API Key Setup

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

Create a `.env` file:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### 2. Database Setup

Create the MySQL/TiDB database schema:

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    url TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

CREATE TABLE IF NOT EXISTS artifactmedia (
    artifact_id INT,
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    primaryimageurl TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

```bash
# Launch the Streamlit app
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Core Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_pages=5):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,  # Max results per page
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """Transform raw API data into structured DataFrames"""
    
    # Metadata transformation
    metadata = []
    media_data = []
    color_data = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown'),
            'culture': artifact.get('culture', 'Unknown'),
            'century': artifact.get('century', 'Unknown'),
            'classification': artifact.get('classification', 'Unknown'),
            'department': artifact.get('department', 'Unknown'),
            'dated': artifact.get('dated', 'Unknown'),
            'url': artifact.get('url', ''),
            'totalpageviews': artifact.get('totalpageviews', 0),
            'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
        })
        
        # Extract media information
        if 'primaryimageurl' in artifact or 'images' in artifact:
            media_data.append({
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl', ''),
                'iiifbaseuri': artifact.get('iiifbaseuri', ''),
                'primaryimageurl': artifact.get('primaryimageurl', '')
            })
        
        # Extract color information
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_data.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', 'Unknown'),
                    'spectrum': color.get('spectrum', 'Unknown'),
                    'hue': color.get('hue', 'Unknown'),
                    'percent': color.get('percent', 0.0)
                })
    
    return (
        pd.DataFrame(metadata),
        pd.DataFrame(media_data),
        pd.DataFrame(color_data)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        cursor = connection.cursor()
        
        # Insert metadata (batch insert for performance)
        metadata_insert = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, dated, url, totalpageviews, totaluniquepageviews)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """
        cursor.executemany(metadata_insert, metadata_df.values.tolist())
        
        # Insert media data
        media_insert = """
            INSERT INTO artifactmedia (artifact_id, baseimageurl, iiifbaseuri, primaryimageurl)
            VALUES (%s, %s, %s, %s)
        """
        cursor.executemany(media_insert, media_df.values.tolist())
        
        # Insert color data
        color_insert = """
            INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(color_insert, colors_df.values.tolist())
        
        connection.commit()
        print(f"Successfully loaded {len(metadata_df)} artifacts")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if connection.is_connected():
            cursor.close()
            connection.close()
```

### 3. SQL Analytics Queries

```python
def get_analytics_queries():
    """Predefined analytical SQL queries"""
    
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture != 'Unknown'
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 15
        """,
        
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century != 'Unknown'
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "Most Viewed Artifacts": """
            SELECT title, culture, totalpageviews
            FROM artifactmetadata
            ORDER BY totalpageviews DESC
            LIMIT 20
        """,
        
        "Color Distribution": """
            SELECT color, COUNT(*) as count, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY count DESC
            LIMIT 10
        """,
        
        "Department Statistics": """
            SELECT department, 
                   COUNT(*) as artifact_count,
                   AVG(totalpageviews) as avg_views
            FROM artifactmetadata
            WHERE department != 'Unknown'
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        
        "Artifacts with Images": """
            SELECT am.classification, COUNT(DISTINCT am.id) as count
            FROM artifactmetadata am
            INNER JOIN artifactmedia med ON am.id = med.artifact_id
            WHERE med.primaryimageurl != ''
            GROUP BY am.classification
            ORDER BY count DESC
            LIMIT 15
        """,
        
        "Color Spectrum Analysis": """
            SELECT spectrum, color, COUNT(*) as occurrences
            FROM artifactcolors
            GROUP BY spectrum, color
            ORDER BY occurrences DESC
            LIMIT 20
        """
    }
    
    return queries

def execute_query(query_name, query_sql):
    """Execute SQL query and return results as DataFrame"""
    
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        df = pd.read_sql(query_sql, connection)
        return df
        
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        if connection.is_connected():
            connection.close()
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    menu = st.sidebar.selectbox(
        "Select Operation",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Data Collection":
        st.header("📥 Data Collection from Harvard API")
        
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
        
        if st.button("Fetch and Load Data"):
            with st.spinner("Fetching data from API..."):
                raw_data = fetch_artifacts(num_pages)
                st.success(f"Fetched {len(raw_data)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df, media_df, colors_df = transform_artifacts(raw_data)
                st.success("Data transformation complete")
            
            with st.spinner("Loading to database..."):
                load_to_database(metadata_df, media_df, colors_df)
                st.success("Data loaded successfully!")
    
    elif menu == "SQL Analytics":
        st.header("📊 SQL Analytics Queries")
        
        queries = get_analytics_queries()
        selected_query = st.selectbox("Select Query", list(queries.keys()))
        
        if st.button("Run Query"):
            with st.spinner("Executing query..."):
                result_df = execute_query(selected_query, queries[selected_query])
                
                if result_df is not None and not result_df.empty:
                    st.dataframe(result_df)
                    
                    # Auto-generate visualization
                    if len(result_df.columns) == 2:
                        fig = px.bar(
                            result_df,
                            x=result_df.columns[0],
                            y=result_df.columns[1],
                            title=selected_query
                        )
                        st.plotly_chart(fig, use_container_width=True)
                else:
                    st.warning("No data returned")
    
    elif menu == "Visualizations":
        st.header("📈 Interactive Visualizations")
        
        viz_type = st.selectbox(
            "Select Visualization",
            ["Culture Distribution", "Century Timeline", "Color Analysis"]
        )
        
        # Custom visualizations based on selection
        # (Implementation would fetch data and create Plotly charts)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Pattern 1: Full ETL Pipeline Execution

```python
def run_full_etl_pipeline(num_pages=10):
    """Execute complete ETL pipeline"""
    
    # Extract
    print("Step 1: Extracting data from API...")
    raw_data = fetch_artifacts(num_pages)
    
    # Transform
    print("Step 2: Transforming data...")
    metadata_df, media_df, colors_df = transform_artifacts(raw_data)
    
    # Load
    print("Step 3: Loading to database...")
    load_to_database(metadata_df, media_df, colors_df)
    
    print("ETL pipeline complete!")
    return len(metadata_df)
```

### Pattern 2: Incremental Data Updates

```python
def incremental_update():
    """Fetch only new artifacts since last update"""
    
    # Get latest artifact ID from database
    query = "SELECT MAX(id) as max_id FROM artifactmetadata"
    connection = mysql.connector.connect(...)
    cursor = connection.cursor()
    cursor.execute(query)
    last_id = cursor.fetchone()[0] or 0
    
    # Fetch newer artifacts
    params = {
        'apikey': os.getenv('HARVARD_API_KEY'),
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': last_id
    }
    
    response = requests.get('https://api.harvardartmuseums.org/object', params=params)
    new_artifacts = response.json().get('records', [])
    
    # Process and load
    if new_artifacts:
        metadata_df, media_df, colors_df = transform_artifacts(new_artifacts)
        load_to_database(metadata_df, media_df, colors_df)
```

### Pattern 3: Custom Analytics Function

```python
def analyze_cultural_trends(culture_name):
    """Analyze artifacts from specific culture"""
    
    query = f"""
        SELECT 
            am.century,
            am.classification,
            COUNT(*) as count,
            AVG(am.totalpageviews) as avg_views,
            GROUP_CONCAT(DISTINCT ac.color) as common_colors
        FROM artifactmetadata am
        LEFT JOIN artifactcolors ac ON am.id = ac.artifact_id
        WHERE am.culture = %s
        GROUP BY am.century, am.classification
        ORDER BY count DESC
    """
    
    connection = mysql.connector.connect(...)
    df = pd.read_sql(query, connection, params=(culture_name,))
    
    return df
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_artifacts_with_retry(num_pages, delay=1):
    """Fetch with rate limiting and retry logic"""
    
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        max_retries = 3
        for attempt in range(max_retries):
            try:
                response = requests.get(base_url, params=params)
                
                if response.status_code == 429:  # Too many requests
                    wait_time = int(response.headers.get('Retry-After', 60))
                    print(f"Rate limited. Waiting {wait_time} seconds...")
                    time.sleep(wait_time)
                    continue
                
                response.raise_for_status()
                data = response.json()
                all_artifacts.extend(data.get('records', []))
                break
                
            except requests.exceptions.RequestException as e:
                if attempt == max_retries - 1:
                    print(f"Failed after {max_retries} attempts: {e}")
                time.sleep(delay * (attempt + 1))
        
        time.sleep(delay)  # Polite delay between requests
    
    return all_artifacts
```

### Database Connection Issues

```python
def get_database_connection(max_retries=3):
    """Get database connection with retry logic"""
    
    for attempt in range(max_retries):
        try:
            connection = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                connection_timeout=10
            )
            return connection
        except Error as e:
            if attempt == max_retries - 1:
                raise Exception(f"Database connection failed: {e}")
            time.sleep(2 ** attempt)
```

### Handling Missing Data

```python
def safe_transform(artifact, field, default='Unknown'):
    """Safely extract field with default value"""
    return artifact.get(field, default) or default

def transform_artifacts_safely(raw_data):
    """Transform with comprehensive error handling"""
    
    metadata = []
    
    for artifact in raw_data:
        try:
            metadata.append({
                'id': artifact.get('id'),
                'title': safe_transform(artifact, 'title', 'Untitled'),
                'culture': safe_transform(artifact, 'culture'),
                'century': safe_transform(artifact, 'century'),
                'classification': safe_transform(artifact, 'classification'),
                'department': safe_transform(artifact, 'department'),
                'dated': safe_transform(artifact, 'dated'),
                'url': artifact.get('url', ''),
                'totalpageviews': int(artifact.get('totalpageviews', 0) or 0),
                'totaluniquepageviews': int(artifact.get('totaluniquepageviews', 0) or 0)
            })
        except Exception as e:
            print(f"Error processing artifact {artifact.get('id')}: {e}")
            continue
    
    return pd.DataFrame(metadata)
```

## Advanced Usage

### Custom Query Builder

```python
class QueryBuilder:
    """Build dynamic SQL queries for artifact analysis"""
    
    def __init__(self, connection):
        self.connection = connection
    
    def filter_by_criteria(self, culture=None, century=None, classification=None, min_views=0):
        """Build filtered query dynamically"""
        
        conditions = ["1=1"]
        params = []
        
        if culture:
            conditions.append("culture = %s")
            params.append(culture)
        
        if century:
            conditions.append("century = %s")
            params.append(century)
        
        if classification:
            conditions.append("classification = %s")
            params.append(classification)
        
        if min_views > 0:
            conditions.append("totalpageviews >= %s")
            params.append(min_views)
        
        query = f"""
            SELECT * FROM artifactmetadata
            WHERE {' AND '.join(conditions)}
            ORDER BY totalpageviews DESC
        """
        
        return pd.read_sql(query, self.connection, params=params)
```

This skill enables you to build production-grade data engineering pipelines for cultural heritage data, with comprehensive ETL operations, SQL analytics, and interactive visualizations.
