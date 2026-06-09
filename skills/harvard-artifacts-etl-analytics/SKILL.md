---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums data
  - show me how to create analytics dashboard with museum artifacts
  - help me extract and transform Harvard API data
  - how to set up SQL database for museum collection data
  - build a Streamlit app for art museum analytics
  - create data engineering pipeline for artifact collections
  - analyze Harvard Art Museums data with SQL queries
  - visualize museum artifact data with Plotly
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What It Does

The Harvard Artifacts Collection application:
- Extracts artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results with interactive Plotly charts in Streamlit

**Architecture**: API → ETL → SQL → Analytics → Visualization

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

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to `.env` file

## Database Schema

The project uses three main tables with foreign key relationships:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    dated VARCHAR(200),
    people TEXT,
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    media_url VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

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

## Key Components

### 1. ETL Pipeline

**Extract data from API:**

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, pages=5, size=100):
    """
    Fetch artifact data from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        pages: Number of pages to fetch
        size: Records per page (max 100)
    
    Returns:
        List of artifact dictionaries
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, pages + 1):
        params = {
            'apikey': api_key,
            'size': size,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}/{pages}")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

**Transform nested JSON:**

```python
def transform_artifacts(raw_artifacts):
    """
    Transform raw API data into normalized dataframes
    
    Returns:
        tuple: (metadata_df, media_df, colors_df)
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in raw_artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'technique': artifact.get('technique'),
            'dated': artifact.get('dated'),
            'people': ', '.join([p.get('name', '') for p in artifact.get('people', [])]),
            'url': artifact.get('url')
        }
        metadata_records.append(metadata)
        
        # Extract media
        for image in artifact.get('images', []):
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'image',
                'media_url': image.get('baseimageurl')
            })
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_records.append({
                'artifact_id': artifact.get('id'),
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
```

**Load into database:**

```python
import mysql.connector
from mysql.connector import Error

def load_to_database(metadata_df, media_df, colors_df, db_config):
    """
    Load dataframes into MySQL database
    
    Args:
        metadata_df: Artifact metadata dataframe
        media_df: Media dataframe
        colors_df: Colors dataframe
        db_config: Dict with host, user, password, database
    """
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, period, century, classification, 
                 department, technique, dated, people, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Batch insert media
        media_values = [tuple(row) for _, row in media_df.iterrows()]
        cursor.executemany("""
            INSERT INTO artifactmedia (artifact_id, media_type, media_url)
            VALUES (%s, %s, %s)
        """, media_values)
        
        # Batch insert colors
        color_values = [tuple(row) for _, row in colors_df.iterrows()]
        cursor.executemany("""
            INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, color_values)
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
    finally:
        if conn.is_connected():
            cursor.close()
            conn.close()
```

### 2. Streamlit Application

**Main application structure:**

```python
import streamlit as st
import pandas as pd
import plotly.express as px
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

def main():
    st.title("🏛️ Harvard Art Museums - Data Analytics Dashboard")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Navigation",
        ["Home", "Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Home":
        show_home()
    elif menu == "Data Collection":
        show_data_collection()
    elif menu == "SQL Analytics":
        show_sql_analytics()
    elif menu == "Visualizations":
        show_visualizations()

def show_data_collection():
    st.header("📥 Data Collection from API")
    
    api_key = os.getenv('HARVARD_API_KEY')
    pages = st.number_input("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Fetch Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, pages=pages)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
            st.success(f"Fetched {len(metadata_df)} artifacts")
            st.dataframe(metadata_df.head(10))
            
            # Load to database
            db_config = {
                'host': os.getenv('DB_HOST'),
                'user': os.getenv('DB_USER'),
                'password': os.getenv('DB_PASSWORD'),
                'database': os.getenv('DB_NAME')
            }
            load_to_database(metadata_df, media_df, colors_df, db_config)

if __name__ == "__main__":
    main()
```

### 3. Analytical SQL Queries

**Example queries for insights:**

```python
ANALYTICAL_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Artifacts with Most Media": """
        SELECT am.title, am.culture, COUNT(m.id) as media_count
        FROM artifactmetadata am
        JOIN artifactmedia m ON am.id = m.artifact_id
        GROUP BY am.id, am.title, am.culture
        ORDER BY media_count DESC
        LIMIT 10
    """,
    
    "Color Distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Department Statistics": """
        SELECT department, 
               COUNT(*) as total_artifacts,
               COUNT(DISTINCT culture) as unique_cultures
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """
}

def execute_query(query, db_config):
    """Execute SQL query and return results as DataFrame"""
    conn = mysql.connector.connect(**db_config)
    df = pd.read_sql(query, conn)
    conn.close()
    return df

def visualize_results(df, chart_type='bar'):
    """Create Plotly visualization from query results"""
    if len(df.columns) >= 2:
        fig = px.bar(
            df, 
            x=df.columns[0], 
            y=df.columns[1],
            title=f"{df.columns[1]} by {df.columns[0]}"
        )
        return fig
    return None
```

### 4. Interactive Dashboard

```python
def show_sql_analytics():
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Run Query"):
        db_config = {
            'host': os.getenv('DB_HOST'),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
        
        query = ANALYTICAL_QUERIES[query_name]
        
        # Display query
        st.code(query, language='sql')
        
        # Execute and display results
        with st.spinner("Executing query..."):
            results_df = execute_query(query, db_config)
            
            st.subheader("Query Results")
            st.dataframe(results_df)
            
            # Visualize
            fig = visualize_results(results_df)
            if fig:
                st.plotly_chart(fig, use_container_width=True)
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# App will open at http://localhost:8501
```

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(api_key, pages, delay=0.5):
    """Fetch with delay between requests"""
    artifacts = []
    for page in range(1, pages + 1):
        response = requests.get(base_url, params={'apikey': api_key, 'page': page})
        if response.status_code == 200:
            artifacts.extend(response.json()['records'])
        time.sleep(delay)  # Respect API rate limits
    return artifacts
```

### Error Handling in ETL

```python
def safe_transform(artifacts):
    """Transform with error handling"""
    successful = []
    failed = []
    
    for artifact in artifacts:
        try:
            # Transform logic
            transformed = transform_single(artifact)
            successful.append(transformed)
        except Exception as e:
            failed.append({'id': artifact.get('id'), 'error': str(e)})
    
    return successful, failed
```

## Troubleshooting

**API Key Issues:**
- Verify key is active at https://www.harvardartmuseums.org/collections/api
- Check `.env` file format: `HARVARD_API_KEY=your_key_here`
- Ensure no quotes around the key value

**Database Connection:**
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Connected successfully")
except Error as e:
    print(f"Connection failed: {e}")
```

**Missing Data:**
- API returns NULL for many fields; handle with `.get()` and default values
- Use `WHERE field IS NOT NULL` in queries to filter incomplete records

**Streamlit Performance:**
- Cache expensive operations with `@st.cache_data`
- Limit query result sizes for faster rendering
- Use pagination for large datasets

```python
@st.cache_data(ttl=3600)
def cached_query(query_string, _db_config):
    return execute_query(query_string, _db_config)
```
