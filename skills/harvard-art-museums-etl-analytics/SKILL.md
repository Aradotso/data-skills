---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums data
  - create a data engineering pipeline with the Harvard Art Museums API
  - set up analytics dashboard for museum artifacts
  - extract and analyze Harvard museum collection data
  - build Streamlit app for art museum data visualization
  - query Harvard Art Museums API and load into SQL database
  - create end-to-end data pipeline for museum artifacts
  - analyze art museum data with Python and SQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load museum data into relational SQL databases
- **Database Design**: Structured schema with artifact metadata, media, and color tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboard**: Streamlit-based UI with Plotly visualizations

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
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Add it to your `.env` file

### Database Setup

The application supports MySQL or TiDB Cloud. Ensure your database is accessible and credentials are configured.

## Key Components

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Example: Fetch first page of artifacts
data = fetch_artifacts(page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Artifacts in this page: {len(data['records'])}")
```

### 2. ETL Pipeline - Extract and Transform

```python
import pandas as pd

def extract_artifact_metadata(api_response):
    """Extract and transform artifact metadata"""
    records = api_response.get('records', [])
    
    artifacts = []
    for record in records:
        artifact = {
            'object_id': record.get('objectid'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'department': record.get('department'),
            'division': record.get('division'),
            'dated': record.get('dated'),
            'period': record.get('period'),
            'provenance': record.get('provenance'),
            'url': record.get('url')
        }
        artifacts.append(artifact)
    
    return pd.DataFrame(artifacts)

def extract_artifact_media(api_response):
    """Extract media/images data"""
    records = api_response.get('records', [])
    
    media_list = []
    for record in records:
        object_id = record.get('objectid')
        images = record.get('images', [])
        
        for img in images:
            media = {
                'object_id': object_id,
                'image_id': img.get('imageid'),
                'base_url': img.get('baseimageurl'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height'),
                'iiif_url': img.get('iiifbaseuri')
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_artifact_colors(api_response):
    """Extract color data"""
    records = api_response.get('records', [])
    
    colors_list = []
    for record in records:
        object_id = record.get('objectid')
        colors = record.get('colors', [])
        
        for color in colors:
            color_entry = {
                'object_id': object_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'saturation': color.get('saturation'),
                'percent': color.get('percent')
            }
            colors_list.append(color_entry)
    
    return pd.DataFrame(colors_list)

# Example usage
data = fetch_artifacts(page=1, size=50)
metadata_df = extract_artifact_metadata(data)
media_df = extract_artifact_media(data)
colors_df = extract_artifact_colors(data)
```

### 3. ETL Pipeline - Load to SQL

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

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title TEXT,
            culture VARCHAR(255),
            century VARCHAR(100),
            classification VARCHAR(255),
            technique TEXT,
            medium TEXT,
            dimensions TEXT,
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            period VARCHAR(255),
            provenance TEXT,
            url TEXT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url TEXT,
            format VARCHAR(50),
            width INT,
            height INT,
            iiif_url TEXT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            saturation FLOAT,
            percent FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_metadata_to_db(df, connection):
    """Batch insert metadata"""
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (object_id, title, culture, century, classification, technique, 
         medium, dimensions, department, division, dated, period, provenance, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    data = df.values.tolist()
    cursor.executemany(insert_query, data)
    connection.commit()
    cursor.close()

# Example: Complete ETL workflow
conn = get_db_connection()
if conn:
    create_tables(conn)
    
    # Extract
    data = fetch_artifacts(page=1, size=100)
    metadata_df = extract_artifact_metadata(data)
    
    # Load
    load_metadata_to_db(metadata_df, conn)
    
    conn.close()
    print("ETL pipeline completed successfully")
```

### 4. SQL Analytics Queries

```python
def run_analytics_query(connection, query_number):
    """Execute predefined analytics queries"""
    
    queries = {
        1: """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        2: """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        3: """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            GROUP BY department
            ORDER BY total_artifacts DESC
        """,
        4: """
            SELECT classification, COUNT(*) as count
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 15
        """,
        5: """
            SELECT 
                COUNT(DISTINCT am.object_id) as artifacts_with_images,
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
                ROUND(COUNT(DISTINCT am.object_id) * 100.0 / 
                      (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
            FROM artifactmedia am
        """,
        6: """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 10
        """
    }
    
    cursor = connection.cursor(dictionary=True)
    cursor.execute(queries[query_number])
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)

# Example usage
conn = get_db_connection()
df_results = run_analytics_query(conn, 1)
print(df_results)
conn.close()
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # Data Collection
    st.header("📥 Data Collection")
    num_pages = st.slider("Number of pages to fetch", 1, 10, 1)
    
    if st.button("Fetch Artifacts"):
        with st.spinner("Fetching data from API..."):
            conn = get_db_connection()
            create_tables(conn)
            
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(page=page, size=100)
                metadata_df = extract_artifact_metadata(data)
                media_df = extract_artifact_media(data)
                colors_df = extract_artifact_colors(data)
                
                load_metadata_to_db(metadata_df, conn)
                # Load media and colors similarly
                
            conn.close()
            st.success(f"Loaded {num_pages * 100} artifacts successfully!")
    
    # Analytics Section
    st.header("📊 Analytics")
    
    query_options = {
        "Top 10 Cultures": 1,
        "Artifacts by Century": 2,
        "Department Distribution": 3,
        "Top Classifications": 4,
        "Image Availability": 5,
        "Popular Colors": 6
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        conn = get_db_connection()
        results_df = run_analytics_query(conn, query_options[selected_query])
        conn.close()
        
        # Display table
        st.dataframe(results_df)
        
        # Visualization
        if len(results_df) > 0:
            x_col = results_df.columns[0]
            y_col = results_df.columns[1]
            
            fig = px.bar(results_df, x=x_col, y=y_col, 
                        title=selected_query)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pagination for Large Datasets

```python
def fetch_all_artifacts(max_pages=10):
    """Fetch multiple pages of artifacts"""
    all_artifacts = []
    
    for page in range(1, max_pages + 1):
        try:
            data = fetch_artifacts(page=page, size=100)
            artifacts = extract_artifact_metadata(data)
            all_artifacts.append(artifacts)
            
            # Rate limiting
            import time
            time.sleep(0.5)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    return pd.concat(all_artifacts, ignore_index=True)
```

### Error Handling in ETL

```python
def safe_etl_pipeline(pages=5):
    """ETL with error handling"""
    conn = get_db_connection()
    
    if not conn:
        print("Failed to connect to database")
        return
    
    try:
        create_tables(conn)
        
        for page in range(1, pages + 1):
            try:
                data = fetch_artifacts(page=page)
                metadata_df = extract_artifact_metadata(data)
                
                if not metadata_df.empty:
                    load_metadata_to_db(metadata_df, conn)
                    print(f"Page {page} loaded successfully")
                    
            except Exception as e:
                print(f"Error processing page {page}: {e}")
                continue
                
    finally:
        conn.close()
```

## Troubleshooting

### API Rate Limiting

```python
import time
from functools import wraps

def rate_limited(max_per_second=2):
    """Decorator to rate limit API calls"""
    min_interval = 1.0 / max_per_second
    
    def decorator(func):
        last_called = [0.0]
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        
        return wrapper
    return decorator

@rate_limited(max_per_second=2)
def fetch_artifacts_limited(page=1, size=100):
    return fetch_artifacts(page, size)
```

### Database Connection Pooling

```python
from mysql.connector import pooling

db_pool = pooling.MySQLConnectionPool(
    pool_name="harvard_pool",
    pool_size=5,
    host=os.getenv('DB_HOST'),
    database=os.getenv('DB_NAME'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)

def get_pooled_connection():
    return db_pool.get_connection()
```

### Handling Missing Data

```python
def clean_artifact_data(df):
    """Clean and validate artifact data"""
    # Fill missing values
    df['culture'] = df['culture'].fillna('Unknown')
    df['century'] = df['century'].fillna('Unknown')
    
    # Remove duplicates
    df = df.drop_duplicates(subset=['object_id'])
    
    # Validate data types
    df['object_id'] = pd.to_numeric(df['object_id'], errors='coerce')
    df = df.dropna(subset=['object_id'])
    
    return df
```

This skill enables you to build complete data engineering pipelines for museum collection data, from API extraction to interactive analytics dashboards.
