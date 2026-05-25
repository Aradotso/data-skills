---
name: harvard-art-museums-etl-analytics
description: Build end-to-end data pipelines with Harvard Art Museums API, ETL processing, SQL analytics, and Streamlit visualization
triggers:
  - how do I fetch data from Harvard Art Museums API
  - build an ETL pipeline for museum artifact data
  - create SQL analytics dashboard with Streamlit
  - process Harvard artifacts collection data
  - set up TiDB for art museum data
  - visualize museum data with Plotly
  - implement pagination for Harvard API
  - query artifact metadata with SQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates an end-to-end data engineering pipeline using the Harvard Art Museums API. It extracts artifact data, transforms it into relational structures, loads it into SQL databases (MySQL/TiDB), and provides interactive analytics through Streamlit dashboards with Plotly visualizations.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (artifacts, media, colors)
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes 20+ predefined analytical SQL queries
- **Visualization**: Interactive Plotly charts within Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_db_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"
```

### Dependencies

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

Create a `.env` file or set these variables:

```bash
# Harvard API
HARVARD_API_KEY=your_harvard_api_key

# Database (MySQL/TiDB)
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### Database Schema

The application uses three main tables:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    medium VARCHAR(500),
    technique VARCHAR(500)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    base_url TEXT,
    format VARCHAR(50),
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

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Key API Patterns

### Fetching Data from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
```

### Handling Pagination

```python
def fetch_all_artifacts(api_key, max_records=500):
    """
    Fetch multiple pages of artifacts with pagination
    """
    all_records = []
    page = 1
    size = 100
    
    while len(all_records) < max_records:
        data = fetch_artifacts(api_key, page=page, size=size)
        records = data.get('records', [])
        
        if not records:
            break
            
        all_records.extend(records)
        page += 1
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_records[:max_records]
```

## ETL Pipeline Implementation

### Extract

```python
import pandas as pd

def extract_artifact_metadata(records):
    """
    Extract metadata from API records
    """
    metadata = []
    
    for record in records:
        artifact = {
            'id': record.get('id'),
            'title': record.get('title'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'division': record.get('division'),
            'dated': record.get('dated'),
            'period': record.get('period'),
            'medium': record.get('medium'),
            'technique': record.get('technique')
        }
        metadata.append(artifact)
    
    return pd.DataFrame(metadata)
```

### Transform

```python
def transform_media_data(records):
    """
    Transform nested media data into flat structure
    """
    media_data = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for image in images:
            media = {
                'artifact_id': artifact_id,
                'media_type': 'image',
                'base_url': image.get('baseimageurl'),
                'format': image.get('format')
            }
            media_data.append(media)
    
    return pd.DataFrame(media_data)

def transform_color_data(records):
    """
    Extract color information from artifacts
    """
    color_data = []
    
    for record in records:
        artifact_id = record.get('id')
        colors = record.get('colors', [])
        
        for color in colors:
            color_entry = {
                'artifact_id': artifact_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            }
            color_data.append(color_entry)
    
    return pd.DataFrame(color_data)
```

### Load

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection
    """
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_metadata(df):
    """
    Load artifact metadata into database
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         division, dated, period, medium, technique)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    for _, row in df.iterrows():
        cursor.execute(insert_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()

def batch_insert(df, table_name, columns):
    """
    Efficient batch insert for large datasets
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    placeholders = ', '.join(['%s'] * len(columns))
    column_names = ', '.join(columns)
    
    insert_query = f"""
        INSERT INTO {table_name} ({column_names})
        VALUES ({placeholders})
    """
    
    data = [tuple(row) for row in df[columns].values]
    cursor.executemany(insert_query, data)
    
    conn.commit()
    cursor.close()
    conn.close()
```

## SQL Analytics Queries

### Example Analytical Queries

```python
def get_artifacts_by_culture():
    """
    Count artifacts grouped by culture
    """
    query = """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """
    return execute_query(query)

def get_media_availability():
    """
    Analyze media availability across artifacts
    """
    query = """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as artifacts_with_media,
            COUNT(*) as total_media_items,
            AVG(media_per_artifact) as avg_media_per_artifact
        FROM artifactmedia m
        JOIN (
            SELECT artifact_id, COUNT(*) as media_per_artifact
            FROM artifactmedia
            GROUP BY artifact_id
        ) sub ON m.artifact_id = sub.artifact_id
    """
    return execute_query(query)

def get_color_distribution():
    """
    Analyze color usage across artifacts
    """
    query = """
        SELECT 
            color, 
            COUNT(DISTINCT artifact_id) as artifact_count,
            AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY artifact_count DESC
        LIMIT 15
    """
    return execute_query(query)

def execute_query(query):
    """
    Execute SQL query and return DataFrame
    """
    conn = get_db_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for query selection
    st.sidebar.header("Select Analysis")
    query_option = st.sidebar.selectbox(
        "Choose Query",
        [
            "Artifacts by Culture",
            "Media Availability",
            "Color Distribution",
            "Department Statistics",
            "Century Analysis"
        ]
    )
    
    # Execute selected query
    if query_option == "Artifacts by Culture":
        df = get_artifacts_by_culture()
        st.subheader("Top 10 Cultures by Artifact Count")
        st.dataframe(df)
        
        # Visualization
        fig = px.bar(
            df, 
            x='culture', 
            y='artifact_count',
            title='Artifacts by Culture',
            labels={'artifact_count': 'Number of Artifacts', 'culture': 'Culture'}
        )
        st.plotly_chart(fig, use_container_width=True)
    
    elif query_option == "Color Distribution":
        df = get_color_distribution()
        st.subheader("Color Usage Patterns")
        st.dataframe(df)
        
        fig = px.bar(
            df,
            x='color',
            y='artifact_count',
            color='avg_percent',
            title='Color Distribution Across Artifacts'
        )
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Complete ETL Workflow

```python
def run_etl_pipeline(num_records=500):
    """
    Complete ETL pipeline execution
    """
    # Extract
    api_key = os.getenv('HARVARD_API_KEY')
    records = fetch_all_artifacts(api_key, max_records=num_records)
    
    # Transform
    metadata_df = extract_artifact_metadata(records)
    media_df = transform_media_data(records)
    colors_df = transform_color_data(records)
    
    # Load
    load_metadata(metadata_df)
    batch_insert(media_df, 'artifactmedia', 
                 ['artifact_id', 'media_type', 'base_url', 'format'])
    batch_insert(colors_df, 'artifactcolors',
                 ['artifact_id', 'color', 'spectrum', 'hue', 'percent'])
    
    print(f"ETL completed: {len(metadata_df)} artifacts processed")
```

### Error Handling

```python
def safe_api_call(api_key, page, retries=3):
    """
    API call with retry logic
    """
    for attempt in range(retries):
        try:
            return fetch_artifacts(api_key, page=page)
        except Exception as e:
            if attempt < retries - 1:
                import time
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise Exception(f"Failed after {retries} attempts: {e}")
```

## Troubleshooting

### API Rate Limiting
- Add `time.sleep()` between requests
- Use smaller page sizes
- Implement exponential backoff for retries

### Database Connection Issues
```python
# Test connection
try:
    conn = get_db_connection()
    print("Database connection successful")
    conn.close()
except Error as e:
    print(f"Error: {e}")
```

### Missing Data Handling
```python
# Handle null values in transformation
df['culture'] = df['culture'].fillna('Unknown')
df['century'] = df['century'].fillna('Undated')
```

### Memory Management for Large Datasets
```python
# Process in chunks
chunk_size = 100
for i in range(0, len(records), chunk_size):
    chunk = records[i:i+chunk_size]
    process_and_load(chunk)
```

## Performance Optimization

- Use batch inserts instead of row-by-row
- Index foreign keys in database
- Cache frequently used queries
- Implement connection pooling for multiple requests
- Use pandas DataFrame operations for transformations
