---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum artifact data
  - create a data engineering workflow with Harvard Art Museums API
  - set up artifact collection analytics with Streamlit
  - implement museum data warehouse with SQL
  - build interactive art analytics dashboard
  - process and visualize Harvard museum artifacts
  - create end-to-end data pipeline for art collections
  - analyze museum artifact metadata with SQL
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations.

## What This Project Does

The Harvard Art Museums ETL Analytics application:
- Fetches artifact data from Harvard Art Museums API with pagination support
- Performs ETL operations to transform nested JSON into relational database tables
- Stores structured data in MySQL/TiDB Cloud databases
- Executes analytical SQL queries for insights
- Visualizes results through interactive Streamlit dashboards with Plotly charts

**Architecture Flow:** API → ETL → SQL → Analytics → Visualization

## Installation

### Prerequisites
```bash
# Python 3.8+
python --version

# MySQL or TiDB Cloud account
```

### Setup
```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Key dependencies include:
# - streamlit
# - pandas
# - requests
# - mysql-connector-python (or pymysql)
# - plotly
```

### Configuration

Create a `.env` file or configure environment variables:
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

Alternatively, configure within Streamlit's secrets management (`~/.streamlit/secrets.toml`):
```toml
[api]
harvard_api_key = "your_api_key_here"

[database]
host = "your_database_host"
port = 3306
user = "your_db_user"
password = "your_db_password"
database = "harvard_artifacts"
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Database Schema

The application uses three main tables with relational structure:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    objectnumber VARCHAR(255),
    title TEXT,
    dated VARCHAR(255),
    century VARCHAR(100),
    culture VARCHAR(255),
    classification VARCHAR(255),
    medium TEXT,
    department VARCHAR(255),
    division VARCHAR(255),
    technique TEXT,
    period VARCHAR(255)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(500),
    primaryimageurl VARCHAR(500),
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

## Core ETL Pipeline Pattern

### 1. Extract: Fetching Data from API

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, num_records=100):
    """
    Fetch artifact data from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    size = 100  # Max per page
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': min(size, num_records - len(artifacts)),
            'page': page
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get('records', [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]

# Usage
api_key = os.getenv('HARVARD_API_KEY')
artifacts = fetch_artifacts(api_key, num_records=500)
```

### 2. Transform: Processing Nested JSON

```python
def transform_artifacts(artifacts):
    """
    Transform raw API response into structured DataFrames
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'objectnumber': artifact.get('objectnumber'),
            'title': artifact.get('title'),
            'dated': artifact.get('dated'),
            'century': artifact.get('century'),
            'culture': artifact.get('culture'),
            'classification': artifact.get('classification'),
            'medium': artifact.get('medium'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'technique': artifact.get('technique'),
            'period': artifact.get('period')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'baseimageurl': artifact.get('baseimageurl'),
            'primaryimageurl': artifact.get('primaryimageurl')
        }
        media_list.append(media)
        
        # Extract color information
        colors = artifact.get('colors', [])
        for color in colors:
            color_entry = {
                'artifact_id': artifact.get('id'),
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'percentage': color.get('percent')
            }
            colors_list.append(color_entry)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### 3. Load: Batch Insert to Database

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection(host, port, user, password, database):
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=host,
            port=port,
            user=user,
            password=password,
            database=database
        )
        return connection
    except Error as e:
        print(f"Error connecting to database: {e}")
        return None

def batch_insert_metadata(connection, df_metadata):
    """
    Batch insert artifact metadata using executemany for performance
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, objectnumber, title, dated, century, culture, classification, 
     medium, department, division, technique, period)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), dated=VALUES(dated), century=VALUES(century)
    """
    
    # Convert DataFrame to list of tuples
    data = df_metadata.where(pd.notna(df_metadata), None).values.tolist()
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} metadata records")
    except Error as e:
        print(f"Error inserting data: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_media(connection, df_media):
    """
    Batch insert artifact media data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, baseimageurl, primaryimageurl)
    VALUES (%s, %s, %s)
    """
    
    data = df_media.where(pd.notna(df_media), None).values.tolist()
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} media records")
    except Error as e:
        print(f"Error inserting media: {e}")
        connection.rollback()
    finally:
        cursor.close()

def batch_insert_colors(connection, df_colors):
    """
    Batch insert artifact color data
    """
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, spectrum, percentage)
    VALUES (%s, %s, %s, %s)
    """
    
    data = df_colors.where(pd.notna(df_colors), None).values.tolist()
    
    try:
        cursor.executemany(insert_query, data)
        connection.commit()
        print(f"Inserted {cursor.rowcount} color records")
    except Error as e:
        print(f"Error inserting colors: {e}")
        connection.rollback()
    finally:
        cursor.close()
```

## Streamlit Application Pattern

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection (from secrets or env)
    conn = create_database_connection(
        host=st.secrets["database"]["host"],
        port=st.secrets["database"]["port"],
        user=st.secrets["database"]["user"],
        password=st.secrets["database"]["password"],
        database=st.secrets["database"]["database"]
    )
    
    # ETL Section
    st.header("1. Data Collection (ETL)")
    
    num_records = st.number_input("Number of artifacts to fetch", 
                                  min_value=10, max_value=1000, value=100)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching artifacts..."):
            api_key = st.secrets["api"]["harvard_api_key"]
            artifacts = fetch_artifacts(api_key, num_records)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            df_metadata, df_media, df_colors = transform_artifacts(artifacts)
            st.success("Data transformed successfully")
        
        with st.spinner("Loading to database..."):
            batch_insert_metadata(conn, df_metadata)
            batch_insert_media(conn, df_media)
            batch_insert_colors(conn, df_colors)
            st.success("Data loaded to database")
    
    # Analytics Section
    st.header("2. SQL Analytics")
    
    queries = {
        "Top 10 Cultures by Artifact Count": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
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
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY count DESC
        """,
        "Top Colors Used": """
            SELECT color, COUNT(*) as usage_count, 
                   AVG(percentage) as avg_percentage
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 10
        """
    }
    
    selected_query = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Execute Query"):
        cursor = conn.cursor()
        cursor.execute(queries[selected_query])
        results = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        
        df_results = pd.DataFrame(results, columns=columns)
        
        st.dataframe(df_results)
        
        # Auto-generate visualization
        if len(df_results.columns) >= 2:
            fig = px.bar(df_results, 
                        x=df_results.columns[0], 
                        y=df_results.columns[1],
                        title=selected_query)
            st.plotly_chart(fig)
        
        cursor.close()

if __name__ == "__main__":
    main()
```

## Common Analytical Queries

```python
# Artifacts with images vs without
"""
SELECT 
    CASE WHEN primaryimageurl IS NOT NULL THEN 'With Image' ELSE 'No Image' END as image_status,
    COUNT(*) as count
FROM artifactmedia
GROUP BY image_status
"""

# Classification breakdown
"""
SELECT classification, COUNT(*) as count
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY count DESC
LIMIT 15
"""

# Color spectrum analysis
"""
SELECT spectrum, COUNT(*) as count, AVG(percentage) as avg_percentage
FROM artifactcolors
WHERE spectrum IS NOT NULL
GROUP BY spectrum
ORDER BY count DESC
"""

# Artifacts by period and culture
"""
SELECT period, culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE period IS NOT NULL AND culture IS NOT NULL
GROUP BY period, culture
ORDER BY artifact_count DESC
LIMIT 20
"""
```

## Troubleshooting

### API Rate Limiting
```python
import time

def fetch_artifacts_with_retry(api_key, num_records, retry_delay=1):
    """Fetch with rate limiting protection"""
    artifacts = []
    page = 1
    
    while len(artifacts) < num_records:
        try:
            response = requests.get(base_url, params=params)
            
            if response.status_code == 429:  # Rate limited
                print(f"Rate limited, waiting {retry_delay}s...")
                time.sleep(retry_delay)
                retry_delay *= 2  # Exponential backoff
                continue
            
            # Process response...
            time.sleep(0.1)  # Gentle rate limiting
            
        except Exception as e:
            print(f"Error: {e}")
            break
    
    return artifacts
```

### Database Connection Issues
```python
def safe_database_connection(max_retries=3):
    """Retry database connection"""
    for attempt in range(max_retries):
        try:
            conn = create_database_connection(...)
            if conn and conn.is_connected():
                return conn
        except Exception as e:
            print(f"Connection attempt {attempt + 1} failed: {e}")
            time.sleep(2)
    
    raise Exception("Could not connect to database")
```

### Handling Missing Data
```python
def safe_transform(artifact, field, default=None):
    """Safely extract nested fields"""
    return artifact.get(field, default)

# For nested structures
def get_nested(data, *keys, default=None):
    """Safely get nested dictionary values"""
    for key in keys:
        if isinstance(data, dict):
            data = data.get(key)
        else:
            return default
    return data if data is not None else default

# Usage
medium = get_nested(artifact, 'medium', 'name', default='Unknown')
```

### Memory Management for Large Datasets
```python
def chunked_insert(connection, df, table_name, chunk_size=1000):
    """Insert large DataFrames in chunks"""
    total_rows = len(df)
    
    for i in range(0, total_rows, chunk_size):
        chunk = df.iloc[i:i+chunk_size]
        batch_insert_metadata(connection, chunk)
        print(f"Processed {min(i+chunk_size, total_rows)}/{total_rows} rows")
```

## Best Practices

1. **Always use parameterized queries** to prevent SQL injection
2. **Batch inserts** for performance (use `executemany`)
3. **Handle NULL values** explicitly in transformations
4. **Use environment variables** for all credentials
5. **Implement retry logic** for API calls
6. **Close database connections** properly
7. **Validate data** before insertion
8. **Use indexes** on frequently queried columns (id, artifact_id, culture, century)

This skill enables comprehensive data engineering workflows for museum artifact analytics using modern Python data stack.
