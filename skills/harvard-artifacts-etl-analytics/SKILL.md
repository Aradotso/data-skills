---
name: harvard-artifacts-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data engineering project with Harvard API
  - set up artifact collection analytics with Streamlit
  - extract and analyze Harvard museum artifacts
  - build a museum data pipeline with SQL
  - create Harvard Art Museums dashboard
  - implement artifact data visualization
  - design ETL for museum collections API
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering pipeline for collecting, transforming, storing, and analyzing artifact data from the Harvard Art Museums API. It demonstrates ETL workflows, SQL analytics, and interactive visualization using Streamlit.

## What This Project Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB with proper schema design
- **Analytics**: Executes 20+ predefined analytical queries
- **Visualization**: Interactive dashboards using Streamlit and Plotly

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

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
mysql-connector-python
requests
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
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Getting Harvard API Key

1. Visit [Harvard Art Museums API](https://harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to your `.env` file

## Database Schema

The project uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    url VARCHAR(500)
);

CREATE TABLE artifactmedia (
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Usage Patterns

### Running the Streamlit App

```bash
streamlit run app.py
```

The app will start on `http://localhost:8501`

### ETL Pipeline Components

#### 1. Extract Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    url = "https://api.harvardartmuseums.org/object"
    params = {
        "apikey": api_key,
        "page": page,
        "size": size
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=100)
```

#### 2. Transform Data

```python
import pandas as pd

def transform_artifacts(raw_data):
    """Transform nested JSON to relational format"""
    artifacts = []
    media_records = []
    color_records = []
    
    for record in raw_data.get('records', []):
        # Metadata
        artifact = {
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'dated': record.get('dated'),
            'accessionyear': record.get('accessionyear'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'url': record.get('url')
        }
        artifacts.append(artifact)
        
        # Media
        for media in record.get('images', []):
            media_records.append({
                'artifact_id': record.get('id'),
                'media_type': 'image',
                'baseimageurl': media.get('baseimageurl')
            })
        
        # Colors
        for color in record.get('colors', []):
            color_records.append({
                'artifact_id': record.get('id'),
                'color': color.get('color'),
                'percentage': color.get('percent')
            })
    
    return {
        'metadata': pd.DataFrame(artifacts),
        'media': pd.DataFrame(media_records),
        'colors': pd.DataFrame(color_records)
    }
```

#### 3. Load to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_artifacts(dataframes):
    """Batch insert data into MySQL"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata
        metadata_query = """
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             dated, accessionyear, technique, medium, dimensions, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """
        metadata_values = dataframes['metadata'].values.tolist()
        cursor.executemany(metadata_query, metadata_values)
        
        # Insert media
        media_query = """
            INSERT INTO artifactmedia (artifact_id, media_type, baseimageurl)
            VALUES (%s, %s, %s)
        """
        media_values = dataframes['media'].values.tolist()
        cursor.executemany(media_query, media_values)
        
        # Insert colors
        color_query = """
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """
        color_values = dataframes['colors'].values.tolist()
        cursor.executemany(color_query, color_values)
        
        conn.commit()
        print(f"Loaded {len(metadata_values)} artifacts successfully")
        
    except Error as e:
        print(f"Error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

### Sample Analytical Queries

#### Query 1: Artifacts by Culture

```python
def get_artifacts_by_culture(limit=10):
    """Get artifact count by culture"""
    conn = get_db_connection()
    query = """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT %s
    """
    df = pd.read_sql(query, conn, params=(limit,))
    conn.close()
    return df
```

#### Query 2: Media Availability Analysis

```python
def analyze_media_availability():
    """Check which artifacts have media"""
    conn = get_db_connection()
    query = """
        SELECT 
            CASE WHEN m.artifact_id IS NOT NULL 
                 THEN 'Has Media' 
                 ELSE 'No Media' 
            END as media_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY media_status
    """
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

#### Query 3: Color Distribution

```python
def get_color_distribution():
    """Analyze color usage across artifacts"""
    conn = get_db_connection()
    query = """
        SELECT color, 
               COUNT(DISTINCT artifact_id) as artifact_count,
               AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY artifact_count DESC
        LIMIT 15
    """
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### Streamlit Dashboard Example

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")

st.title("🏛️ Harvard Art Museums Analytics Dashboard")

# Sidebar for query selection
query_options = [
    "Artifacts by Culture",
    "Media Availability",
    "Color Distribution",
    "Artifacts by Century",
    "Department Analysis"
]

selected_query = st.sidebar.selectbox("Select Analysis", query_options)

# Execute and display
if selected_query == "Artifacts by Culture":
    st.header("📊 Artifacts by Culture")
    df = get_artifacts_by_culture(limit=15)
    
    # Display table
    st.dataframe(df)
    
    # Visualization
    fig = px.bar(df, x='culture', y='count', 
                 title='Top Cultures by Artifact Count',
                 labels={'count': 'Number of Artifacts', 'culture': 'Culture'})
    st.plotly_chart(fig, use_container_width=True)

elif selected_query == "Color Distribution":
    st.header("🎨 Color Distribution Analysis")
    df = get_color_distribution()
    
    st.dataframe(df)
    
    fig = px.bar(df, x='color', y='artifact_count',
                 title='Color Usage Across Artifacts',
                 color='avg_percentage',
                 labels={'artifact_count': 'Number of Artifacts'})
    st.plotly_chart(fig, use_container_width=True)
```

## Common Workflows

### Complete ETL Execution

```python
def run_etl_pipeline(num_pages=5):
    """Execute full ETL pipeline"""
    api_key = os.getenv("HARVARD_API_KEY")
    
    all_metadata = []
    all_media = []
    all_colors = []
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        raw_data = fetch_artifacts(api_key, page=page, size=100)
        
        # Transform
        transformed = transform_artifacts(raw_data)
        
        all_metadata.append(transformed['metadata'])
        all_media.append(transformed['media'])
        all_colors.append(transformed['colors'])
    
    # Combine all pages
    combined = {
        'metadata': pd.concat(all_metadata, ignore_index=True),
        'media': pd.concat(all_media, ignore_index=True),
        'colors': pd.concat(all_colors, ignore_index=True)
    }
    
    # Load
    load_artifacts(combined)
    
    print(f"ETL Complete: {len(combined['metadata'])} artifacts processed")

# Run pipeline
run_etl_pipeline(num_pages=10)
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt
                print(f"Retry in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

### Database Connection Issues

```python
def test_db_connection():
    """Verify database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        result = cursor.fetchone()
        cursor.close()
        conn.close()
        print("✅ Database connection successful")
        return True
    except Error as e:
        print(f"❌ Database connection failed: {e}")
        return False
```

### Handle Missing Data

```python
def safe_transform(record, field, default=None):
    """Safely extract field with fallback"""
    value = record.get(field, default)
    return value if value not in [None, '', 'null'] else default

# Usage in transform
artifact = {
    'id': record.get('id'),
    'title': safe_transform(record, 'title', 'Untitled'),
    'culture': safe_transform(record, 'culture', 'Unknown'),
    # ...
}
```

## Best Practices

1. **Always use environment variables** for API keys and database credentials
2. **Implement pagination** when fetching large datasets from the API
3. **Use batch inserts** for better database performance
4. **Handle NULL values** in transformations to avoid SQL errors
5. **Add error logging** for production deployments
6. **Cache query results** in Streamlit using `@st.cache_data`
7. **Close database connections** properly to avoid leaks
