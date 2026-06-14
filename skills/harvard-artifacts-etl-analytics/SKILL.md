---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline for Harvard Art Museums API
  - show me how to create analytics dashboards with museum artifact data
  - help me set up a data engineering project with Harvard artifacts
  - how to extract and transform Harvard Art Museums API data
  - build a Streamlit app for museum artifact analytics
  - create SQL queries for Harvard Art Museums collection data
  - how to visualize museum artifact data with Plotly
  - set up a complete data pipeline from API to dashboard
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualizations with Streamlit.

## What This Project Does

The Harvard Artifacts Collection application:
- Fetches artifact data from the Harvard Art Museums API with pagination
- Performs ETL operations to transform nested JSON into relational tables
- Stores structured data in MySQL/TiDB Cloud databases
- Executes analytical SQL queries on artifact metadata, media, and colors
- Visualizes insights through interactive Plotly charts in a Streamlit dashboard

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Requirements (typical dependencies):**
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

```env
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Database Setup

The application uses three main tables with foreign key relationships:

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    technique VARCHAR(500),
    dated VARCHAR(255),
    accessionyear INT,
    objectnumber VARCHAR(100)
);

CREATE TABLE artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

CREATE TABLE artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API with pagination"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    response.raise_for_status()
    
    data = response.json()
    return data['records'], data['info']

def collect_all_artifacts(max_records=1000):
    """Collect artifacts with pagination handling"""
    all_artifacts = []
    page = 1
    size = 100
    
    while len(all_artifacts) < max_records:
        records, info = fetch_artifacts(page=page, size=size)
        all_artifacts.extend(records)
        
        if info['next'] is None:
            break
        
        page += 1
    
    return all_artifacts[:max_records]
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector

def extract_metadata(artifacts):
    """Extract artifact metadata into DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', ''),
            'century': artifact.get('century', ''),
            'classification': artifact.get('classification', ''),
            'department': artifact.get('department', ''),
            'division': artifact.get('division', ''),
            'technique': artifact.get('technique', ''),
            'dated': artifact.get('dated', ''),
            'accessionyear': artifact.get('accessionyear'),
            'objectnumber': artifact.get('objectnumber', '')
        })
    
    return pd.DataFrame(metadata)

def extract_media(artifacts):
    """Extract media URLs into DataFrame"""
    media = []
    
    for artifact in artifacts:
        if artifact.get('images'):
            for image in artifact['images']:
                media.append({
                    'artifact_id': artifact.get('id'),
                    'baseimageurl': image.get('baseimageurl', ''),
                    'primaryimageurl': artifact.get('primaryimageurl', '')
                })
    
    return pd.DataFrame(media)

def extract_colors(artifacts):
    """Extract color data into DataFrame"""
    colors = []
    
    for artifact in artifacts:
        if artifact.get('colors'):
            for color in artifact['colors']:
                colors.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'percent': color.get('percent', 0.0)
                })
    
    return pd.DataFrame(colors)

def load_to_database(df, table_name, connection):
    """Load DataFrame into SQL database using batch insert"""
    cursor = connection.cursor()
    
    # Prepare column names and placeholders
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
    
    # Batch insert
    data = [tuple(row) for row in df.values]
    cursor.executemany(insert_query, data)
    connection.commit()
    
    cursor.close()
```

### 3. Database Connection

```python
def get_db_connection():
    """Create and return database connection"""
    connection = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    return connection

def execute_etl_pipeline():
    """Complete ETL pipeline execution"""
    # Extract
    artifacts = collect_all_artifacts(max_records=1000)
    
    # Transform
    df_metadata = extract_metadata(artifacts)
    df_media = extract_media(artifacts)
    df_colors = extract_colors(artifacts)
    
    # Load
    conn = get_db_connection()
    
    load_to_database(df_metadata, 'artifactmetadata', conn)
    load_to_database(df_media, 'artifactmedia', conn)
    load_to_database(df_colors, 'artifactcolors', conn)
    
    conn.close()
    
    return len(artifacts)
```

### 4. Analytical SQL Queries

```python
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as occurrences, AVG(percent) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 15
    """,
    
    "Artifacts with Images": """
        SELECT 
            CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'No Image' END as status,
            COUNT(*) as count
        FROM artifactmedia
        GROUP BY status
    """,
    
    "Accession Year Trends": """
        SELECT accessionyear, COUNT(*) as count
        FROM artifactmetadata
        WHERE accessionyear IS NOT NULL
        GROUP BY accessionyear
        ORDER BY accessionyear DESC
        LIMIT 30
    """
}

def execute_analytics_query(query_name):
    """Execute analytical query and return DataFrame"""
    conn = get_db_connection()
    df = pd.read_sql_query(ANALYTICS_QUERIES[query_name], conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar
    st.sidebar.header("Controls")
    
    # ETL Section
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL pipeline..."):
            count = execute_etl_pipeline()
            st.success(f"ETL completed! Loaded {count} artifacts.")
    
    # Analytics Section
    st.header("📊 Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        options=list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        df = execute_analytics_query(query_name)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualization
        if len(df) > 0 and len(df.columns) >= 2:
            st.subheader("Visualization")
            
            fig = px.bar(
                df,
                x=df.columns[0],
                y=df.columns[1],
                title=query_name
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py
```

The dashboard will be available at `http://localhost:8501`

## Common Patterns

### Rate Limiting API Requests

```python
import time

def fetch_with_rate_limit(page, size, delay=0.5):
    """Fetch data with rate limiting"""
    data, info = fetch_artifacts(page, size)
    time.sleep(delay)  # Avoid hitting API rate limits
    return data, info
```

### Error Handling in ETL

```python
def safe_extract_metadata(artifacts):
    """Extract metadata with error handling"""
    metadata = []
    errors = []
    
    for artifact in artifacts:
        try:
            metadata.append({
                'id': artifact.get('id'),
                'title': artifact.get('title', '')[:500],
                # ... other fields
            })
        except Exception as e:
            errors.append({'artifact_id': artifact.get('id'), 'error': str(e)})
    
    return pd.DataFrame(metadata), errors
```

### Incremental Data Loading

```python
def load_new_artifacts_only():
    """Load only artifacts not already in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    # Get existing IDs
    cursor.execute("SELECT id FROM artifactmetadata")
    existing_ids = set(row[0] for row in cursor.fetchall())
    
    # Fetch new artifacts
    all_artifacts = collect_all_artifacts()
    new_artifacts = [a for a in all_artifacts if a['id'] not in existing_ids]
    
    # Load only new ones
    if new_artifacts:
        df_metadata = extract_metadata(new_artifacts)
        load_to_database(df_metadata, 'artifactmetadata', conn)
    
    conn.close()
    return len(new_artifacts)
```

## Troubleshooting

### API Key Issues
- Ensure `HARVARD_API_KEY` is set in `.env`
- Get a free API key from: https://harvardartmuseums.org/collections/api

### Database Connection Errors
```python
# Test connection
try:
    conn = get_db_connection()
    print("Database connected successfully!")
    conn.close()
except mysql.connector.Error as e:
    print(f"Database error: {e}")
```

### Memory Issues with Large Datasets
```python
# Process in smaller batches
def batch_etl(batch_size=100, total_records=1000):
    for offset in range(0, total_records, batch_size):
        artifacts = collect_all_artifacts(max_records=batch_size)
        # Process and load this batch
        execute_etl_pipeline()
```

### Handling Missing Data
```python
# Clean data before loading
df_metadata = df_metadata.fillna({
    'culture': 'Unknown',
    'century': 'Unknown',
    'accessionyear': 0
})
```

This skill provides everything needed to build production-ready ETL pipelines and analytics dashboards for museum artifact data using modern data engineering tools.
