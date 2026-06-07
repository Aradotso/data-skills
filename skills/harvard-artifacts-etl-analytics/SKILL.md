---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - how do I fetch artifacts from Harvard Art Museums API
  - build an ETL pipeline for museum data
  - create analytics dashboard with Streamlit and SQL
  - process Harvard Art Museums collection data
  - set up data engineering pipeline for artifacts
  - query and visualize museum artifact data
  - integrate Harvard API with MySQL database
  - design relational schema for museum collections
---

# Harvard Artifacts ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates ETL pipeline development, SQL database design, and interactive visualization with Streamlit.

## What It Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:
- Dynamic data collection from Harvard Art Museums API
- ETL (Extract, Transform, Load) operations for artifact metadata
- Relational SQL database storage (MySQL/TiDB Cloud)
- Predefined analytical SQL queries
- Interactive Streamlit dashboards with Plotly visualizations

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Required Dependencies:**
```
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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

## Database Schema

The application uses three main tables with relational design:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    department VARCHAR(200),
    classification VARCHAR(200),
    dated VARCHAR(200),
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    iiifbaseuri VARCHAR(500),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core ETL Pipeline

### Extract Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()
        data = response.json()
        return data
    except requests.exceptions.RequestException as e:
        print(f"Error fetching data: {e}")
        return None

# Fetch multiple pages with rate limiting
import time

def fetch_all_artifacts(num_pages=5):
    """
    Fetch multiple pages of artifacts with rate limiting
    """
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        
        if data and 'records' in data:
            all_artifacts.extend(data['records'])
            time.sleep(1)  # Rate limiting: 1 second between requests
        else:
            break
    
    return all_artifacts
```

### Transform Data

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """
    Transform raw API data into structured metadata
    """
    metadata = []
    
    for artifact in artifacts:
        record = {
            'id': artifact.get('id'),
            'title': artifact.get('title', 'Unknown')[:500],
            'culture': artifact.get('culture', 'Unknown')[:200],
            'century': artifact.get('century', 'Unknown')[:100],
            'department': artifact.get('department', 'Unknown')[:200],
            'classification': artifact.get('classification', 'Unknown')[:200],
            'dated': artifact.get('dated', 'Unknown')[:200],
            'medium': artifact.get('medium', 'Unknown')[:500],
            'dimensions': artifact.get('dimensions', 'Unknown')[:500],
            'creditline': artifact.get('creditline', 'Unknown'),
            'url': artifact.get('url', '')[:500]
        }
        metadata.append(record)
    
    return pd.DataFrame(metadata)

def transform_artifact_media(artifacts):
    """
    Extract media/image data from artifacts
    """
    media_records = []
    
    for artifact in artifacts:
        if 'primaryimageurl' in artifact or 'images' in artifact:
            media = {
                'artifact_id': artifact.get('id'),
                'baseimageurl': artifact.get('baseimageurl', ''),
                'primaryimageurl': artifact.get('primaryimageurl', ''),
                'iiifbaseuri': artifact.get('iiifbaseuri', '')
            }
            media_records.append(media)
    
    return pd.DataFrame(media_records)

def transform_artifact_colors(artifacts):
    """
    Extract color palette data from artifacts
    """
    color_records = []
    
    for artifact in artifacts:
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                record = {
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', ''),
                    'spectrum': color.get('spectrum', ''),
                    'percentage': color.get('percent', 0.0)
                }
                color_records.append(record)
    
    return pd.DataFrame(color_records)
```

### Load Data to Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """
    Create database connection using environment variables
    """
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME'),
            port=int(os.getenv('DB_PORT', 3306))
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata_to_db(df_metadata):
    """
    Batch insert metadata into database
    """
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, department, classification, dated, medium, dimensions, creditline, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture)
    """
    
    try:
        # Convert DataFrame to list of tuples
        records = df_metadata.to_records(index=False).tolist()
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
        return True
    except Error as e:
        print(f"Error inserting metadata: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_media_to_db(df_media):
    """
    Batch insert media data into database
    """
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (artifact_id, baseimageurl, primaryimageurl, iiifbaseuri)
    VALUES (%s, %s, %s, %s)
    """
    
    try:
        records = df_media.to_records(index=False).tolist()
        cursor.executemany(insert_query, records)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
        return True
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## Streamlit Application

### Main App Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(
        page_title="Harvard Artifacts Analytics",
        page_icon="🏛️",
        layout="wide"
    )
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["ETL Pipeline", "SQL Analytics", "Visualizations"]
    )
    
    if page == "ETL Pipeline":
        show_etl_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    else:
        show_visualizations_page()

def show_etl_page():
    st.header("ETL Pipeline")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=10, value=3)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_all_artifacts(num_pages=num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata = transform_artifact_metadata(artifacts)
            df_media = transform_artifact_media(artifacts)
            df_colors = transform_artifact_colors(artifacts)
            st.success("Data transformation complete")
        
        with st.spinner("Loading to database..."):
            load_metadata_to_db(df_metadata)
            load_media_to_db(df_media)
            st.success("Data loaded to database")

if __name__ == "__main__":
    main()
```

## Analytical SQL Queries

### Sample Analytics Queries

```python
# Query 1: Artifacts by Department
query_by_department = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL AND department != 'Unknown'
GROUP BY department
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts by Culture
query_by_culture = """
SELECT culture, COUNT(*) as count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != 'Unknown'
GROUP BY culture
ORDER BY count DESC
LIMIT 15;
"""

# Query 3: Artifacts by Century
query_by_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL AND century != 'Unknown'
GROUP BY century
ORDER BY count DESC;
"""

# Query 4: Color Distribution
query_color_distribution = """
SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
FROM artifactcolors
WHERE color IS NOT NULL
GROUP BY color
ORDER BY usage_count DESC
LIMIT 10;
"""

# Query 5: Media Availability
query_media_availability = """
SELECT 
    COUNT(DISTINCT m.artifact_id) as artifacts_with_images,
    (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
    ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
FROM artifactmedia m;
"""

def execute_query(query):
    """
    Execute SQL query and return results as DataFrame
    """
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        st.error(f"Query error: {e}")
        return None
    finally:
        connection.close()
```

### Visualization with Plotly

```python
def show_analytics_page():
    st.header("SQL Analytics")
    
    query_options = {
        "Artifacts by Department": query_by_department,
        "Artifacts by Culture": query_by_culture,
        "Artifacts by Century": query_by_century,
        "Color Distribution": query_color_distribution,
        "Media Availability": query_media_availability
    }
    
    selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
    
    if st.button("Run Query"):
        query = query_options[selected_query]
        df_result = execute_query(query)
        
        if df_result is not None and not df_result.empty:
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result,
                    x=df_result.columns[0],
                    y=df_result.columns[1],
                    title=selected_query
                )
                st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Full ETL Workflow

```python
def run_complete_etl(num_pages=5):
    """
    Execute complete ETL pipeline
    """
    # Extract
    print("Starting extraction...")
    artifacts = fetch_all_artifacts(num_pages=num_pages)
    
    # Transform
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(artifacts)
    df_media = transform_artifact_media(artifacts)
    df_colors = transform_artifact_colors(artifacts)
    
    # Load
    print("Loading to database...")
    load_metadata_to_db(df_metadata)
    load_media_to_db(df_media)
    
    # Optional: Load colors (if you have the table)
    # load_colors_to_db(df_colors)
    
    print(f"ETL complete: {len(artifacts)} artifacts processed")
    return True
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time

for page in range(1, num_pages + 1):
    data = fetch_artifacts(page=page)
    time.sleep(1)  # Wait 1 second between requests
```

### Database Connection Issues
```python
# Test connection
def test_db_connection():
    conn = get_db_connection()
    if conn and conn.is_connected():
        print("Database connection successful")
        conn.close()
        return True
    else:
        print("Database connection failed")
        return False
```

### Handling Missing Data
```python
# Safe value extraction with defaults
def safe_get(dictionary, key, default='Unknown', max_length=None):
    value = dictionary.get(key, default)
    if max_length and value:
        value = str(value)[:max_length]
    return value
```

### Memory Management for Large Datasets
```python
# Process in batches
def load_in_batches(df, batch_size=1000):
    for i in range(0, len(df), batch_size):
        batch = df[i:i+batch_size]
        load_metadata_to_db(batch)
        print(f"Loaded batch {i//batch_size + 1}")
```
