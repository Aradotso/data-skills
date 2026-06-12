---
name: harvard-artifacts-etl-analytics
description: Build end-to-end data pipelines with Harvard Art Museums API, ETL processing, SQL storage, and Streamlit analytics dashboards
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - set up data engineering project with museum artifacts
  - create analytics dashboard for Harvard Art collections
  - process and visualize Harvard museum data
  - build SQL analytics for art museum collections
  - implement ETL workflow for artifact metadata
  - query and analyze Harvard Art Museums database
  - create Streamlit app for museum data visualization
---

# Harvard Artifacts Collection Data Engineering & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project is an end-to-end data engineering and analytics application that demonstrates ETL pipelines, SQL analytics, and interactive visualization using the Harvard Art Museums API. It extracts artifact metadata, transforms nested JSON into relational structures, loads data into SQL databases, and provides analytical insights through a Streamlit dashboard.

## What This Project Does

- **Data Collection**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables (metadata, media, colors)
- **SQL Storage**: Stores structured data in MySQL/TiDB Cloud with proper relationships
- **Analytics**: Executes 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly charts and Streamlit dashboards

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

**Key dependencies**:
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

1. Register at https://www.harvardartmuseums.org/collections/api
2. Receive API key via email
3. Add to `.env` file

### Database Setup

The application supports MySQL and TiDB Cloud. Create three tables:

```sql
CREATE TABLE artifactmetadata (
    objectid INT PRIMARY KEY,
    title VARCHAR(500),
    dated VARCHAR(200),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    technique VARCHAR(500),
    medium VARCHAR(500),
    creditline TEXT,
    accessionyear INT
);

CREATE TABLE artifactmedia (
    mediaid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    iiifbaseuri VARCHAR(500),
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);

CREATE TABLE artifactcolors (
    colorid INT AUTO_INCREMENT PRIMARY KEY,
    objectid INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
);
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py
```

The app will open at `http://localhost:8501`

## Core Components

### 1. API Integration

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv('HARVARD_API_KEY')
BASE_URL = 'https://api.harvardartmuseums.org/object'

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts with pagination"""
    params = {
        'apikey': API_KEY,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(BASE_URL, params=params)
    response.raise_for_status()
    return response.json()

# Example: Fetch first 100 artifacts
data = fetch_artifacts(page=1, size=100)
artifacts = data['records']
total_pages = data['info']['pages']
```

### 2. ETL Pipeline

**Extract and Transform:**

```python
import pandas as pd

def extract_metadata(artifacts):
    """Extract metadata into DataFrame"""
    metadata = []
    
    for artifact in artifacts:
        metadata.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'dated': artifact.get('dated'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'creditline': artifact.get('creditline'),
            'accessionyear': artifact.get('accessionyear')
        })
    
    return pd.DataFrame(metadata)

def extract_media(artifacts):
    """Extract media information"""
    media = []
    
    for artifact in artifacts:
        if artifact.get('primaryimageurl'):
            media.append({
                'objectid': artifact.get('objectid'),
                'iiifbaseuri': artifact.get('images', [{}])[0].get('iiifbaseuri') if artifact.get('images') else None,
                'baseimageurl': artifact.get('images', [{}])[0].get('baseimageurl') if artifact.get('images') else None,
                'primaryimageurl': artifact.get('primaryimageurl')
            })
    
    return pd.DataFrame(media)

def extract_colors(artifacts):
    """Extract color information"""
    colors = []
    
    for artifact in artifacts:
        if artifact.get('colors'):
            for color in artifact.get('colors', []):
                colors.append({
                    'objectid': artifact.get('objectid'),
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
    
    return pd.DataFrame(colors)
```

**Load to Database:**

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

def load_metadata(df_metadata):
    """Batch insert metadata"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (objectid, title, dated, culture, century, classification, 
     department, division, technique, medium, creditline, accessionyear)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE objectid=objectid
    """
    
    data = [tuple(row) for row in df_metadata.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_media(df_media):
    """Batch insert media"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia 
    (objectid, iiifbaseuri, baseimageurl, primaryimageurl)
    VALUES (%s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()

def load_colors(df_colors):
    """Batch insert colors"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors 
    (objectid, color, spectrum, hue, percent)
    VALUES (%s, %s, %s, %s, %s)
    """
    
    data = [tuple(row) for row in df_colors.values]
    cursor.executemany(insert_query, data)
    conn.commit()
    
    cursor.close()
    conn.close()
```

### 3. Complete ETL Workflow

```python
def run_etl_pipeline(num_pages=5):
    """Complete ETL pipeline"""
    all_artifacts = []
    
    # Extract
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}...")
        data = fetch_artifacts(page=page, size=100)
        all_artifacts.extend(data['records'])
    
    # Transform
    df_metadata = extract_metadata(all_artifacts)
    df_media = extract_media(all_artifacts)
    df_colors = extract_colors(all_artifacts)
    
    # Load
    load_metadata(df_metadata)
    load_media(df_media)
    load_colors(df_colors)
    
    print(f"ETL complete: {len(all_artifacts)} artifacts processed")
    return len(all_artifacts)
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Top 10 Cultures": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY count DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as occurrences, 
               ROUND(AVG(percent), 2) as avg_percent
        FROM artifactcolors
        GROUP BY color
        ORDER BY occurrences DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Colors": """
        SELECT m.objectid, m.title, COUNT(c.colorid) as num_colors
        FROM artifactmetadata m
        JOIN artifactcolors c ON m.objectid = c.objectid
        GROUP BY m.objectid, m.title
        ORDER BY num_colors DESC
        LIMIT 10
    """,
    
    "Media Availability": """
        SELECT 
            COUNT(DISTINCT m.objectid) as total_artifacts,
            COUNT(DISTINCT med.objectid) as with_media,
            ROUND(COUNT(DISTINCT med.objectid) * 100.0 / COUNT(DISTINCT m.objectid), 2) as media_percentage
        FROM artifactmetadata m
        LEFT JOIN artifactmedia med ON m.objectid = med.objectid
    """
}

def execute_query(query_name):
    """Execute analytical query"""
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
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar
    st.sidebar.header("ETL Pipeline")
    
    if st.sidebar.button("Run ETL Pipeline"):
        with st.spinner("Running ETL..."):
            num_artifacts = run_etl_pipeline(num_pages=5)
            st.sidebar.success(f"Processed {num_artifacts} artifacts")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Execute Query"):
        with st.spinner("Running query..."):
            df_results = execute_query(query_name)
            
            # Display results
            st.dataframe(df_results)
            
            # Auto-generate visualization
            if len(df_results.columns) == 2:
                fig = px.bar(
                    df_results,
                    x=df_results.columns[0],
                    y=df_results.columns[1],
                    title=query_name
                )
                st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Incremental ETL

```python
def get_max_objectid():
    """Get latest objectid in database"""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return result[0] or 0

def incremental_etl():
    """Only fetch new artifacts"""
    last_id = get_max_objectid()
    # Fetch artifacts with objectid > last_id
    # Process and load
```

### Error Handling

```python
def safe_etl(num_pages=5):
    """ETL with error handling"""
    try:
        for page in range(1, num_pages + 1):
            try:
                data = fetch_artifacts(page=page)
                # Process...
            except requests.exceptions.RequestException as e:
                st.error(f"API error on page {page}: {e}")
                continue
            except Error as e:
                st.error(f"Database error on page {page}: {e}")
                continue
    except Exception as e:
        st.error(f"Unexpected error: {e}")
```

## Troubleshooting

**API Rate Limiting:**
```python
import time

def fetch_with_retry(page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

**Database Connection Issues:**
```python
# Test connection
try:
    conn = get_db_connection()
    print("Database connected successfully")
    conn.close()
except Error as e:
    print(f"Database connection failed: {e}")
```

**Missing Data Handling:**
```python
# Clean dataframe before insert
df_metadata = df_metadata.where(pd.notnull(df_metadata), None)
```

## Best Practices

1. **Use environment variables** for all credentials
2. **Batch inserts** for better performance (100-1000 rows)
3. **Handle NULL values** explicitly in transformations
4. **Add indexes** to frequently queried columns
5. **Log ETL progress** for debugging
6. **Validate data** before database insertion
7. **Use transactions** for atomic operations
