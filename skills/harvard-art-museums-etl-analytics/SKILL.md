---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - create a data engineering project with museum artifacts
  - set up Harvard Art Museums data analytics application
  - build a Streamlit dashboard for art museum data
  - how to extract and analyze Harvard museum collection data
  - implement SQL analytics for Harvard Art Museums API
  - create an end-to-end data pipeline for museum artifacts
  - visualize Harvard Art Museums collection with Plotly
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates building an end-to-end data engineering and analytics application using the Harvard Art Museums API. It covers API integration, ETL pipeline development, SQL database design, analytical queries, and interactive visualization with Streamlit.

## What It Does

The application:
- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into normalized relational tables
- Loads data into SQL databases (MySQL/TiDB Cloud)
- Executes analytical SQL queries on the dataset
- Visualizes insights using Plotly charts in a Streamlit interface

**Architecture**: API → ETL → SQL → Analytics → Visualization

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
DB_PORT=3306
DB_USER=your_db_username
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key (free)
3. Add to your `.env` file

## Database Setup

### Create Database Schema

```sql
CREATE DATABASE IF NOT EXISTS harvard_artifacts;
USE harvard_artifacts;

-- Artifact metadata table
CREATE TABLE IF NOT EXISTS artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    dated VARCHAR(200),
    classification VARCHAR(200),
    department VARCHAR(200),
    accession_number VARCHAR(100),
    url VARCHAR(500),
    credit_line TEXT
);

-- Artifact media table
CREATE TABLE IF NOT EXISTS artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    image_url VARCHAR(500),
    base_image_url VARCHAR(500),
    thumbnail_url VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

-- Artifact colors table
CREATE TABLE IF NOT EXISTS artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Core Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only fetch artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params, timeout=30)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

def collect_all_artifacts(api_key, max_pages=10):
    """
    Collect multiple pages of artifact data
    """
    all_records = []
    
    for page in range(1, max_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(api_key, page=page)
        
        if data and 'records' in data:
            all_records.extend(data['records'])
            print(f"Collected {len(data['records'])} records from page {page}")
        else:
            break
    
    return all_records
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

class HarvardETL:
    def __init__(self, db_config):
        self.db_config = db_config
        self.connection = None
    
    def connect_db(self):
        """Establish database connection"""
        try:
            self.connection = mysql.connector.connect(**self.db_config)
            print("Database connection successful")
        except Error as e:
            print(f"Error connecting to database: {e}")
    
    def extract(self, api_key, max_pages=5):
        """Extract data from API"""
        print("Starting extraction...")
        records = collect_all_artifacts(api_key, max_pages)
        return records
    
    def transform(self, records):
        """Transform raw JSON into structured dataframes"""
        metadata_list = []
        media_list = []
        colors_list = []
        
        for record in records:
            object_id = record.get('objectid')
            
            # Transform metadata
            metadata = {
                'objectid': object_id,
                'title': record.get('title', '')[:500],
                'culture': record.get('culture', '')[:200],
                'century': record.get('century', '')[:100],
                'dated': record.get('dated', '')[:200],
                'classification': record.get('classification', '')[:200],
                'department': record.get('department', '')[:200],
                'accession_number': record.get('accessionyear', ''),
                'url': record.get('url', '')[:500],
                'credit_line': record.get('creditline', '')
            }
            metadata_list.append(metadata)
            
            # Transform media
            images = record.get('images', [])
            for img in images:
                media = {
                    'objectid': object_id,
                    'image_url': img.get('imageurl', '')[:500],
                    'base_image_url': img.get('baseimageurl', '')[:500],
                    'thumbnail_url': img.get('thumbnailurl', '')[:500]
                }
                media_list.append(media)
            
            # Transform colors
            colors = record.get('colors', [])
            for color in colors:
                color_data = {
                    'objectid': object_id,
                    'color': color.get('color', '')[:50],
                    'spectrum': color.get('spectrum', '')[:50],
                    'percentage': color.get('percent', 0.0)
                }
                colors_list.append(color_data)
        
        df_metadata = pd.DataFrame(metadata_list)
        df_media = pd.DataFrame(media_list)
        df_colors = pd.DataFrame(colors_list)
        
        print(f"Transformed {len(df_metadata)} metadata records")
        print(f"Transformed {len(df_media)} media records")
        print(f"Transformed {len(df_colors)} color records")
        
        return df_metadata, df_media, df_colors
    
    def load(self, df_metadata, df_media, df_colors):
        """Load dataframes into SQL database"""
        if not self.connection:
            self.connect_db()
        
        cursor = self.connection.cursor()
        
        # Load metadata
        for _, row in df_metadata.iterrows():
            sql = """
            INSERT INTO artifactmetadata 
            (objectid, title, culture, century, dated, classification, 
             department, accession_number, url, credit_line)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(sql, tuple(row))
        
        # Load media
        for _, row in df_media.iterrows():
            sql = """
            INSERT INTO artifactmedia 
            (objectid, image_url, base_image_url, thumbnail_url)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        # Load colors
        for _, row in df_colors.iterrows():
            sql = """
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, percentage)
            VALUES (%s, %s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        self.connection.commit()
        print("Data loaded successfully")
        cursor.close()
```

### 3. Running the ETL Pipeline

```python
import os
from dotenv import load_dotenv

load_dotenv()

# Database configuration
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Initialize ETL
etl = HarvardETL(db_config)

# Run ETL pipeline
api_key = os.getenv('HARVARD_API_KEY')
records = etl.extract(api_key, max_pages=5)
df_metadata, df_media, df_colors = etl.transform(records)
etl.load(df_metadata, df_media, df_colors)
```

## Analytical SQL Queries

### Common Analytics Patterns

```python
def execute_query(connection, query):
    """Execute SQL query and return DataFrame"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    return pd.DataFrame(results)

# Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10
"""

# Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC
"""

# Department distribution
query_departments = """
SELECT department, COUNT(*) as total_artifacts
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY total_artifacts DESC
"""

# Color analysis
query_colors = """
SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
FROM artifactcolors
GROUP BY color
ORDER BY usage_count DESC
LIMIT 15
"""

# Media availability
query_media = """
SELECT 
    COUNT(DISTINCT am.objectid) as artifacts_with_media,
    COUNT(DISTINCT m.objectid) as total_artifacts
FROM artifactmetadata m
LEFT JOIN artifactmedia am ON m.objectid = am.objectid
"""
```

## Streamlit Dashboard

### Main Application Structure

```python
import streamlit as st
import plotly.express as px
import mysql.connector
import pandas as pd
import os
from dotenv import load_dotenv

load_dotenv()

st.set_page_config(page_title="Harvard Art Museums Analytics", layout="wide")

# Sidebar configuration
st.sidebar.title("⚙️ Configuration")
api_key_input = st.sidebar.text_input("Harvard API Key", type="password", 
                                      value=os.getenv('HARVARD_API_KEY', ''))

# Database connection
@st.cache_resource
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

# Main app
st.title("🎨 Harvard Art Museums Analytics Dashboard")

tab1, tab2, tab3 = st.tabs(["📊 Analytics", "🔄 ETL Pipeline", "📈 Visualizations"])

with tab1:
    st.header("SQL Analytics")
    
    queries = {
        "Top 10 Cultures": query_cultures,
        "Artifacts by Century": query_century,
        "Department Distribution": query_departments,
        "Color Analysis": query_colors
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        df = execute_query(conn, queries[selected_query])
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

with tab2:
    st.header("ETL Pipeline")
    
    max_pages = st.number_input("Number of pages to fetch", 1, 20, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            etl = HarvardETL(db_config)
            records = etl.extract(api_key_input, max_pages)
            df_metadata, df_media, df_colors = etl.transform(records)
            etl.load(df_metadata, df_media, df_colors)
            st.success(f"ETL completed! Processed {len(df_metadata)} artifacts")

with tab3:
    st.header("Interactive Visualizations")
    # Add custom visualizations
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues
```python
def verify_db_connection(db_config):
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Connection failed: {e}")
        return False
```

### Handling Missing Data
```python
def safe_get(record, key, default='', max_len=None):
    """Safely extract value from record with length limit"""
    value = record.get(key, default)
    if max_len and value:
        return str(value)[:max_len]
    return value
```

## Best Practices

1. **Always use environment variables** for API keys and credentials
2. **Implement pagination** when fetching large datasets
3. **Use batch inserts** for better database performance
4. **Cache database connections** in Streamlit with `@st.cache_resource`
5. **Handle API rate limits** with retry logic and exponential backoff
6. **Validate data** before inserting into database
7. **Use foreign keys** to maintain referential integrity
