---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums API data using Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - extract and transform Harvard API data
  - set up artifact collection data pipeline
  - analyze Harvard museum data with SQL
  - visualize museum artifact insights
  - build Streamlit app for museum data
  - query Harvard Art Museums API
---

# Harvard Artifacts ETL & Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

Harvard-Artifacts-Collection-Data-Engineering-Analytics-App is a complete data pipeline that:
- Extracts artifact data from the Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables
- Loads structured data into MySQL/TiDB Cloud
- Provides 20+ predefined SQL analytics queries
- Visualizes results through interactive Plotly charts in a Streamlit dashboard

**Architecture:** `API → ETL → SQL → Analytics → Visualization`

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
DB_PORT=4000
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free for educational/research use)
3. Add to `.env` file

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
    classification VARCHAR(255),
    century VARCHAR(255),
    dated VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500),
    url TEXT
)
""")

# Artifact media table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl TEXT,
    format VARCHAR(50),
    height INT,
    width INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

# Artifact colors table
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
```

## Key Components

### 1. API Data Extraction

```python
import requests
import time
import os

def fetch_artifacts(api_key, num_pages=5):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'size': 100,  # Max results per page
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts = data.get('records', [])
            all_artifacts.extend(artifacts)
            print(f"Fetched page {page}: {len(artifacts)} artifacts")
            
            # Rate limiting
            time.sleep(1)
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_pages=10)
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector

def transform_artifacts(artifacts):
    """
    Transform nested JSON artifacts into relational dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'classification': artifact.get('classification', '')[:255],
            'century': artifact.get('century', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'department': artifact.get('department', '')[:255],
            'division': artifact.get('division', '')[:255],
            'medium': artifact.get('medium', '')[:500],
            'technique': artifact.get('technique', '')[:500],
            'url': artifact.get('url', '')
        }
        metadata_records.append(metadata)
        
        # Extract media/images
        images = artifact.get('images', [])
        for img in images:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': img.get('baseimageurl'),
                'format': img.get('format'),
                'height': img.get('height'),
                'width': img.get('width')
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

def load_to_database(df_metadata, df_media, df_colors, conn):
    """
    Load transformed data into SQL database
    """
    cursor = conn.cursor()
    
    # Load metadata
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, classification, century, dated, 
             department, division, medium, technique, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """, tuple(row))
    
    # Load media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (artifact_id, baseimageurl, format, height, width)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    print(f"Loaded {len(df_metadata)} artifacts to database")

# Full ETL execution
artifacts = fetch_artifacts(api_key, num_pages=5)
df_meta, df_media, df_colors = transform_artifacts(artifacts)
load_to_database(df_meta, df_media, df_colors, conn)
```

### 3. SQL Analytics Queries

```python
def get_analytics_queries():
    """
    Collection of analytical SQL queries
    """
    queries = {
        "Artifacts by Culture": """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        "Artifacts by Century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "Top Departments": """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        "Media Availability": """
            SELECT 
                CASE WHEN baseimageurl IS NOT NULL THEN 'Has Image' 
                     ELSE 'No Image' END as image_status,
                COUNT(*) as count
            FROM artifactmedia
            GROUP BY image_status
        """,
        
        "Top Colors in Collection": """
            SELECT color, 
                   COUNT(*) as artifact_count,
                   ROUND(AVG(percentage), 2) as avg_percentage
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY artifact_count DESC
            LIMIT 15
        """,
        
        "Artifacts by Classification": """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """
    }
    
    return queries

def execute_query(conn, query_name, query_sql):
    """
    Execute SQL query and return results as DataFrame
    """
    try:
        df = pd.read_sql(query_sql, conn)
        return df
    except Exception as e:
        print(f"Error executing {query_name}: {e}")
        return None
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums - Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        st.sidebar.success("✅ Database Connected")
    except Exception as e:
        st.sidebar.error(f"❌ Database Error: {e}")
        return
    
    # ETL Section
    st.header("📥 Data Collection")
    
    col1, col2 = st.columns(2)
    
    with col1:
        api_key = st.text_input("Harvard API Key", 
                                value=os.getenv('HARVARD_API_KEY', ''),
                                type="password")
    
    with col2:
        num_pages = st.number_input("Number of Pages", 
                                     min_value=1, 
                                     max_value=50, 
                                     value=5)
    
    if st.button("🚀 Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_meta, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(df_meta, df_media, df_colors, conn)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    queries = get_analytics_queries()
    query_name = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            result_df = execute_query(conn, query_name, queries[query_name])
            
            if result_df is not None and not result_df.empty:
                st.subheader("Query Results")
                st.dataframe(result_df, use_container_width=True)
                
                # Auto-generate visualization
                if len(result_df.columns) >= 2:
                    fig = px.bar(
                        result_df, 
                        x=result_df.columns[0], 
                        y=result_df.columns[1],
                        title=query_name,
                        labels={result_df.columns[0]: result_df.columns[0].title(),
                               result_df.columns[1]: result_df.columns[1].title()}
                    )
                    st.plotly_chart(fig, use_container_width=True)
    
    conn.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Batch Processing for Large Datasets

```python
def batch_insert(cursor, table, columns, data, batch_size=1000):
    """
    Insert data in batches for better performance
    """
    placeholders = ', '.join(['%s'] * len(columns))
    query = f"INSERT INTO {table} ({', '.join(columns)}) VALUES ({placeholders})"
    
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(query, batch)
    
    cursor.connection.commit()
```

### Error Handling for API Requests

```python
def safe_api_request(url, params, max_retries=3):
    """
    API request with retry logic
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Rate Limiting:**
- Add `time.sleep(1)` between requests
- Use smaller page sizes (`size=50` instead of 100)
- Check API quota at Harvard's developer portal

**Database Connection Issues:**
- Verify credentials in `.env`
- Check firewall rules for TiDB Cloud
- Ensure SSL/TLS settings if required

**Memory Issues with Large Datasets:**
- Process data in smaller batches
- Use `chunksize` parameter in `pd.read_sql()`
- Clear DataFrames after loading: `del df_meta`

**Missing Data in Queries:**
- Some artifacts may have NULL values
- Add `WHERE field IS NOT NULL` filters
- Use `COALESCE()` for default values

**Streamlit Performance:**
- Use `@st.cache_data` for expensive queries
- Implement pagination for large result sets
- Store connection in `st.session_state`
