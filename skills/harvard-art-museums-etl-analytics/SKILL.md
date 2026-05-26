---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - create a data pipeline for art museum artifacts
  - build an ETL pipeline with Harvard Art Museums API
  - set up analytics dashboard for museum collection data
  - query and visualize Harvard art museum data
  - create Streamlit app for artifact analytics
  - implement SQL analytics on museum API data
  - design ETL workflow for cultural heritage data
  - build interactive art collection dashboard
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides a complete data pipeline that:

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON responses into normalized relational tables
- **Loads** structured data into MySQL/TiDB databases with optimized batch inserts
- **Analyzes** data using 20+ predefined SQL analytical queries
- **Visualizes** insights through interactive Plotly charts in a Streamlit dashboard

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

```bash
# Python 3.8+ required
python --version

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```txt
streamlit>=1.25.0
pandas>=2.0.0
mysql-connector-python>=8.0.33
requests>=2.31.0
plotly>=5.15.0
python-dotenv>=1.0.0
```

### Database Setup

You need a MySQL or TiDB Cloud database instance. Create the following schema:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    description TEXT,
    url VARCHAR(500)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Getting API Access

1. Register at [Harvard Art Museums API](https://docs.google.com/forms/d/e/1FAIpQLSfkmEBqH76HLMMiCC-GPPnhcvHC9aJS86E32dOd0Z8MpY2rvQ/viewform)
2. Receive API key via email
3. Store in environment variable

## Core Usage Patterns

### 1. Running the Streamlit Application

```bash
# Start the dashboard
streamlit run app.py

# Access at http://localhost:8501
```

### 2. ETL Pipeline Implementation

```python
import requests
import pandas as pd
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

# API Configuration
API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'

def extract_artifacts(num_pages=5, per_page=100):
    """Extract artifacts from Harvard API with pagination"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': API_KEY,
            'size': per_page,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(BASE_URL, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data.get('records', []))
            print(f"Fetched page {page}: {len(data.get('records', []))} artifacts")
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts

def transform_artifacts(raw_artifacts):
    """Transform nested JSON to relational structure"""
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_artifacts:
        artifact_id = artifact.get('id')
        
        # Metadata
        metadata = {
            'id': artifact_id,
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'department': artifact.get('department', 'Unknown')[:200],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'description': artifact.get('description', '')[:5000],
            'url': artifact.get('url', '')[:500]
        }
        metadata_list.append(metadata)
        
        # Media/Images
        images = artifact.get('images', [])
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl', '')[:1000],
                'media_type': 'image'
            })
        
        # Colors
        colors = artifact.get('colors', [])
        for color in colors:
            colors_list.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """Load transformed data into MySQL database"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Insert metadata (batch insert)
    metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, description, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    cursor.executemany(metadata_query, metadata_df.values.tolist())
    
    # Insert media
    media_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(media_query, media_df.values.tolist())
    
    # Insert colors
    colors_query = """
        INSERT INTO artifactcolors (artifact_id, color_hex, color_percent)
        VALUES (%s, %s, %s)
    """
    cursor.executemany(colors_query, colors_df.values.tolist())
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(metadata_df)} artifacts to database")

# Execute ETL pipeline
if __name__ == "__main__":
    raw_data = extract_artifacts(num_pages=5)
    metadata, media, colors = transform_artifacts(raw_data)
    load_to_database(metadata, media, colors)
```

### 3. SQL Analytics Queries

```python
# Sample analytical queries for the dashboard

ANALYTICAL_QUERIES = {
    "Top 10 Cultures by Artifact Count": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture != 'Unknown'
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century Distribution": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Department-wise Classification": """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE department != 'Unknown' AND classification != 'Unknown'
        GROUP BY department, classification
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Most Common Colors": """
        SELECT color_hex, COUNT(*) as usage_count,
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Multiple Images": """
        SELECT a.id, a.title, COUNT(m.media_id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title
        HAVING image_count > 1
        ORDER BY image_count DESC
        LIMIT 15
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    query = ANALYTICAL_QUERIES[query_name]
    df = pd.read_sql(query, conn)
    conn.close()
    
    return df
```

### 4. Streamlit Dashboard Components

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit dashboard"""
    st.title("🎨 Harvard Art Museums Analytics")
    st.markdown("**ETL Pipeline & SQL Analytics Dashboard**")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Analysis",
        list(ANALYTICAL_QUERIES.keys())
    )
    
    # Execute selected query
    if st.button("Run Analysis"):
        with st.spinner("Executing query..."):
            results_df = execute_query(page)
            
            # Display results
            st.subheader(f"Results: {page}")
            st.dataframe(results_df)
            
            # Auto-generate visualization
            if len(results_df.columns) >= 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=page
                )
                st.plotly_chart(fig, use_container_width=True)
    
    # ETL Controls
    st.sidebar.markdown("---")
    st.sidebar.subheader("🔄 ETL Pipeline")
    
    num_pages = st.sidebar.number_input("Pages to fetch", 1, 20, 5)
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            raw_data = extract_artifacts(num_pages=num_pages)
            metadata, media, colors = transform_artifacts(raw_data)
            load_to_database(metadata, media, colors)
            st.sidebar.success(f"✅ Loaded {len(metadata)} artifacts")

if __name__ == "__main__":
    create_dashboard()
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time

def extract_with_rate_limit(num_pages, per_page=100, delay=0.5):
    """Extract with rate limiting to avoid API throttling"""
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        response = requests.get(BASE_URL, params={
            'apikey': API_KEY,
            'size': per_page,
            'page': page
        })
        
        if response.status_code == 200:
            all_artifacts.extend(response.json().get('records', []))
        elif response.status_code == 429:
            print("Rate limit hit, waiting 60 seconds...")
            time.sleep(60)
            continue
        
        time.sleep(delay)  # Respectful delay between requests
    
    return all_artifacts
```

### Handling Missing Data

```python
def safe_extract(artifact, key, default='Unknown', max_length=None):
    """Safely extract fields with defaults and truncation"""
    value = artifact.get(key, default)
    if value is None:
        value = default
    value = str(value)
    if max_length:
        value = value[:max_length]
    return value
```

### Incremental Loading

```python
def get_last_artifact_id(conn):
    """Get the most recent artifact ID in database"""
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    return result or 0

def incremental_etl():
    """Only load new artifacts not in database"""
    conn = get_db_connection()
    last_id = get_last_artifact_id(conn)
    
    # Fetch artifacts with ID greater than last_id
    params = {
        'apikey': API_KEY,
        'size': 100,
        'sort': 'id',
        'sortorder': 'asc',
        'after': last_id
    }
    # Continue with extraction...
```

## Troubleshooting

### API Key Issues

```python
# Test API connectivity
def test_api_connection():
    response = requests.get(
        f"{BASE_URL}?apikey={API_KEY}&size=1"
    )
    if response.status_code == 200:
        print("✅ API connection successful")
        return True
    else:
        print(f"❌ API error: {response.status_code}")
        print(f"Response: {response.text}")
        return False
```

### Database Connection Errors

```python
def test_database_connection():
    try:
        conn = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        print("✅ Database connection successful")
        conn.close()
        return True
    except Exception as e:
        print(f"❌ Database error: {e}")
        return False
```

### Handling Large Datasets

```python
def batch_insert(cursor, query, data, batch_size=1000):
    """Insert large datasets in batches to avoid memory issues"""
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(query, batch)
        cursor.connection.commit()
        print(f"Inserted batch {i//batch_size + 1}")
```

## Advanced Features

### Custom Query Builder

```python
def build_custom_query(table, columns, filters=None, limit=100):
    """Dynamically build SQL queries"""
    query = f"SELECT {', '.join(columns)} FROM {table}"
    
    if filters:
        conditions = " AND ".join([f"{k} = %s" for k in filters.keys()])
        query += f" WHERE {conditions}"
    
    query += f" LIMIT {limit}"
    return query
```

### Export Results

```python
def export_to_csv(df, filename):
    """Export query results to CSV"""
    df.to_csv(filename, index=False)
    st.download_button(
        label="Download CSV",
        data=df.to_csv(index=False),
        file_name=filename,
        mime="text/csv"
    )
```

This skill provides a complete foundation for building data engineering pipelines with museum APIs, SQL analytics, and interactive dashboards.
