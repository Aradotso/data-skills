---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data pipelines from Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualizations
triggers:
  - create an ETL pipeline for Harvard Art Museums data
  - build a data engineering app with Streamlit and SQL
  - fetch and analyze Harvard artifacts collection
  - set up analytics dashboard for museum API data
  - implement ETL workflow for art museums API
  - visualize Harvard Art Museums data with Plotly
  - query and analyze artifact metadata with SQL
  - build museum data pipeline with Python
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **SQL Analytics**: Execute predefined analytical queries on artifact metadata, media, and color data
- **Interactive Dashboards**: Visualize query results using Streamlit and Plotly
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

Required dependencies:
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

Create a `.env` file or configure environment variables:

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

1. Visit [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Store in environment variable

### Database Setup

The application supports MySQL or TiDB Cloud:

```python
import mysql.connector
import os

def get_db_connection():
    """Establish database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
```

## Key Components

### 1. API Data Collection

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        page: Page number for pagination
        size: Number of records per page
    
    Returns:
        dict: API response with artifact records
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Filter for artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=100)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Records fetched: {len(data['records'])}")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector

def extract_artifact_metadata(records):
    """Extract and transform artifact metadata"""
    metadata_list = []
    
    for record in records:
        metadata = {
            'object_id': record.get('objectid'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'dated': record.get('dated'),
            'period': record.get('period'),
            'technique': record.get('technique'),
            'accession_year': record.get('accessionyear'),
            'has_image': 1 if record.get('primaryimageurl') else 0
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def extract_artifact_media(records):
    """Extract media/image data"""
    media_list = []
    
    for record in records:
        if record.get('images'):
            for img in record['images']:
                media = {
                    'object_id': record.get('objectid'),
                    'image_id': img.get('imageid'),
                    'base_url': img.get('baseimageurl'),
                    'width': img.get('width'),
                    'height': img.get('height'),
                    'format': img.get('format'),
                    'copyright': img.get('copyright')
                }
                media_list.append(media)
    
    return pd.DataFrame(media_list)

def extract_artifact_colors(records):
    """Extract color data"""
    color_list = []
    
    for record in records:
        if record.get('colors'):
            for color in record['colors']:
                color_data = {
                    'object_id': record.get('objectid'),
                    'color_name': color.get('color'),
                    'hex_code': color.get('hex'),
                    'percentage': color.get('percent')
                }
                color_list.append(color_data)
    
    return pd.DataFrame(color_list)

# ETL Execution
def run_etl_pipeline(api_key, num_pages=5):
    """Run complete ETL pipeline"""
    all_records = []
    
    # Extract
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(api_key, page=page)
        all_records.extend(data['records'])
        print(f"Extracted page {page}")
    
    # Transform
    metadata_df = extract_artifact_metadata(all_records)
    media_df = extract_artifact_media(all_records)
    colors_df = extract_artifact_colors(all_records)
    
    # Load
    conn = get_db_connection()
    metadata_df.to_sql('artifactmetadata', conn, if_exists='append', index=False)
    media_df.to_sql('artifactmedia', conn, if_exists='append', index=False)
    colors_df.to_sql('artifactcolors', conn, if_exists='append', index=False)
    conn.close()
    
    return len(all_records)
```

### 3. Database Schema Creation

```python
def create_database_schema(conn):
    """Create SQL tables for artifact data"""
    cursor = conn.cursor()
    
    # Artifact Metadata Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            object_id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            division VARCHAR(200),
            dated VARCHAR(200),
            period VARCHAR(200),
            technique VARCHAR(500),
            accession_year INT,
            has_image TINYINT
        )
    """)
    
    # Artifact Media Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            image_id INT,
            base_url VARCHAR(500),
            width INT,
            height INT,
            format VARCHAR(50),
            copyright VARCHAR(500),
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    # Artifact Colors Table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            object_id INT,
            color_name VARCHAR(100),
            hex_code VARCHAR(10),
            percentage FLOAT,
            FOREIGN KEY (object_id) REFERENCES artifactmetadata(object_id)
        )
    """)
    
    conn.commit()
    cursor.close()
```

### 4. SQL Analytics Queries

```python
# Example analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
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
    
    "Color Distribution": """
        SELECT color_name, COUNT(*) as usage_count, 
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color_name
        ORDER BY usage_count DESC
        LIMIT 15
    """,
    
    "Media Format Analysis": """
        SELECT format, COUNT(*) as count,
               AVG(width) as avg_width,
               AVG(height) as avg_height
        FROM artifactmedia
        WHERE format IS NOT NULL
        GROUP BY format
        ORDER BY count DESC
    """,
    
    "Artifacts with Images": """
        SELECT has_image,
               COUNT(*) as count,
               ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) as percentage
        FROM artifactmetadata
        GROUP BY has_image
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    
    cursor.execute(ANALYTICS_QUERIES[query_name])
    results = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    st.sidebar.title("Navigation")
    page = st.sidebar.radio("Select Page", 
                            ["Data Collection", "Analytics", "Visualizations"])
    
    if page == "Data Collection":
        st.header("API Data Collection")
        
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        num_pages = st.slider("Number of pages to fetch", 1, 20, 5)
        
        if st.button("Start ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                records_count = run_etl_pipeline(api_key, num_pages)
                st.success(f"✅ Loaded {records_count} artifacts successfully!")
    
    elif page == "Analytics":
        st.header("SQL Analytics")
        
        query_name = st.selectbox("Select Query", list(ANALYTICS_QUERIES.keys()))
        
        if st.button("Execute Query"):
            df = execute_query(query_name)
            st.dataframe(df)
            st.code(ANALYTICS_QUERIES[query_name], language='sql')
    
    elif page == "Visualizations":
        st.header("Data Visualizations")
        
        # Culture Distribution
        df = execute_query("Artifacts by Culture")
        fig = px.bar(df, x='culture', y='count', 
                     title="Artifacts by Culture",
                     labels={'count': 'Number of Artifacts'})
        st.plotly_chart(fig)
        
        # Color Distribution
        df_colors = execute_query("Color Distribution")
        fig2 = px.bar(df_colors, x='color_name', y='usage_count',
                      title="Most Common Colors in Artifacts")
        st.plotly_chart(fig2)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Batch Processing with Pagination

```python
def fetch_all_artifacts(api_key, max_records=1000):
    """Fetch artifacts with automatic pagination"""
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data['records']
        
        if not records:
            break
        
        all_records.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_records[:max_records]
```

### Error Handling

```python
def safe_api_call(api_key, page, retries=3):
    """API call with retry logic"""
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except requests.exceptions.RequestException as e:
            if attempt == retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests using `time.sleep(0.5)`

**Database Connection Errors**: Verify environment variables and network connectivity

**Missing Data**: Check API response structure with `print(json.dumps(record, indent=2))`

**Large Datasets**: Use batch inserts with `executemany()` for better performance

**Streamlit Caching**: Use `@st.cache_data` decorator to cache query results

```python
@st.cache_data
def execute_query_cached(query_name):
    return execute_query(query_name)
```
