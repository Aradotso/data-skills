---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards for Harvard Art Museums data using Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create analytics dashboard for museum artifact data
  - fetch and analyze Harvard art collection data
  - build streamlit app for museum data visualization
  - design SQL schema for artifact metadata
  - implement paginated API collection for Harvard museums
  - query and visualize museum artifact analytics
  - set up data engineering pipeline for art collections
---

# Harvard Artifacts Collection Data Engineering & Analytics App

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates end-to-end data engineering workflows: extracting data from the Harvard Art Museums API, transforming it into relational structures, loading into SQL databases, and building interactive analytics dashboards with Streamlit.

## What It Does

- **API Integration**: Collects artifact metadata, media, and color data from Harvard Art Museums API
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper foreign key relationships
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly charts in Streamlit dashboards

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"

# Run the Streamlit app
streamlit run app.py
```

## API Configuration

The Harvard Art Museums API requires authentication. Get your API key from: https://www.harvardartmuseums.org/collections/api

```python
import requests
import os

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = "https://api.harvardartmuseums.org/object"

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size
    }
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    dated VARCHAR(200),
    division VARCHAR(200),
    accessionyear INT,
    lastupdate DATETIME
);

-- Artifact media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(100),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline

### Extract

```python
import requests
import pandas as pd

def extract_artifacts(num_pages=5):
    """Extract artifact data from API"""
    all_records = []
    
    for page in range(1, num_pages + 1):
        data = fetch_artifacts(page=page, size=100)
        records = data.get('records', [])
        all_records.extend(records)
        
        # Rate limiting
        time.sleep(0.5)
    
    return all_records
```

### Transform

```python
def transform_metadata(records):
    """Transform raw API data into metadata DataFrame"""
    metadata = []
    
    for record in records:
        metadata.append({
            'id': record.get('id'),
            'title': record.get('title', '')[:500],
            'culture': record.get('culture', '')[:200],
            'century': record.get('century', '')[:100],
            'classification': record.get('classification', '')[:200],
            'department': record.get('department', '')[:200],
            'technique': record.get('technique', '')[:300],
            'dated': record.get('dated', '')[:200],
            'division': record.get('division', '')[:200],
            'accessionyear': record.get('accessionyear'),
            'lastupdate': record.get('lastupdate')
        })
    
    return pd.DataFrame(metadata)

def transform_media(records):
    """Transform media data"""
    media = []
    
    for record in records:
        if record.get('primaryimageurl') or record.get('baseimageurl'):
            media.append({
                'artifact_id': record.get('id'),
                'baseimageurl': record.get('baseimageurl', ''),
                'primaryimageurl': record.get('primaryimageurl', '')
            })
    
    return pd.DataFrame(media)

def transform_colors(records):
    """Transform color data"""
    colors = []
    
    for record in records:
        color_data = record.get('colors', [])
        for color in color_data:
            colors.append({
                'artifact_id': record.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'percent': color.get('percent', 0.0)
            })
    
    return pd.DataFrame(colors)
```

### Load

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

def load_metadata(df):
    """Load metadata into database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         technique, dated, division, accessionyear, lastupdate)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    for _, row in df.iterrows():
        cursor.execute(insert_query, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()

def batch_load(df, table_name, batch_size=1000):
    """Optimized batch loading"""
    conn = get_db_connection()
    
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        batch.to_sql(table_name, conn, if_exists='append', index=False)
    
    conn.close()
```

## Analytics Queries

```python
ANALYTICS_QUERIES = {
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
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY century
    """,
    
    "Media Availability": """
        SELECT 
            CASE WHEN primaryimageurl != '' THEN 'Has Image' ELSE 'No Image' END as status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY status
    """,
    
    "Top Colors": """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifacts DESC
    """
}

def execute_query(query_name):
    """Execute analytical query and return DataFrame"""
    conn = get_db_connection()
    query = ANALYTICS_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

## Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Artifacts Analytics Dashboard")
    
    # Sidebar for query selection
    query_name = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    # Execute query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            df = execute_query(query_name)
            
            # Display results
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Visualization
            if len(df.columns) >= 2:
                st.subheader("Visualization")
                fig = px.bar(df, x=df.columns[0], y=df.columns[1])
                st.plotly_chart(fig)
    
    # Data collection section
    st.sidebar.header("Data Collection")
    num_pages = st.sidebar.number_input("Pages to collect", 1, 10, 5)
    
    if st.sidebar.button("Collect & Load Data"):
        with st.spinner("Running ETL pipeline..."):
            # Extract
            records = extract_artifacts(num_pages)
            
            # Transform
            metadata_df = transform_metadata(records)
            media_df = transform_media(records)
            colors_df = transform_colors(records)
            
            # Load
            load_metadata(metadata_df)
            batch_load(media_df, 'artifactmedia')
            batch_load(colors_df, 'artifactcolors')
            
            st.success(f"Loaded {len(metadata_df)} artifacts!")

if __name__ == "__main__":
    main()
```

## Common Patterns

### Handling Pagination

```python
def collect_all_artifacts(max_records=1000):
    """Collect artifacts with automatic pagination"""
    records = []
    page = 1
    size = 100
    
    while len(records) < max_records:
        data = fetch_artifacts(page, size)
        batch = data.get('records', [])
        
        if not batch:
            break
            
        records.extend(batch)
        page += 1
        
        # Respect rate limits
        time.sleep(0.5)
    
    return records[:max_records]
```

### Error Handling

```python
def safe_api_call(page, max_retries=3):
    """API call with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page)
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff
```

## Troubleshooting

**API Rate Limiting**: Add delays between requests
```python
time.sleep(0.5)  # 500ms between requests
```

**Database Connection Issues**: Check credentials and network
```python
try:
    conn = get_db_connection()
    conn.ping(reconnect=True)
except Error as e:
    print(f"Database error: {e}")
```

**Memory Issues with Large Datasets**: Use batch processing
```python
for chunk in pd.read_sql(query, conn, chunksize=1000):
    process_chunk(chunk)
```

**Missing API Key**: Ensure environment variable is set
```python
if not os.getenv('HARVARD_API_KEY'):
    raise ValueError("HARVARD_API_KEY environment variable not set")
```
