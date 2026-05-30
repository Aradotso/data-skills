---
name: harvard-art-museums-data-pipeline
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build a data pipeline for museum artifacts
  - integrate Harvard Art Museums API
  - create ETL pipeline with streamlit dashboard
  - fetch and analyze art museum collection data
  - set up SQL database for artifact metadata
  - visualize museum collection analytics
  - build data engineering project with API
  - create analytics dashboard for art collections
---

# Harvard Art Museums Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application that extracts artifact data from the Harvard Art Museums API, transforms it into relational tables, loads it into a SQL database, and provides interactive analytics through a Streamlit dashboard.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON responses into structured relational data
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics**: Provides 20+ predefined analytical SQL queries
- **Visualization**: Interactive Plotly charts and tabular results via Streamlit

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

### Required Dependencies

```
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### 1. Harvard Art Museums API Key

Obtain a free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api).

### 2. Database Setup

Set up a MySQL or TiDB Cloud database with credentials.

### 3. Environment Variables

Create a `.env` file in the project root:

```bash
# Harvard API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_username
DB_PASSWORD=your_password
DB_NAME=harvard_artifacts
```

### 4. Database Schema

The application uses three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    period VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    division VARCHAR(200),
    accessionyear INT,
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(200)
);

-- Artifact media table
CREATE TABLE artifactmedia (
    artifact_id INT,
    media_type VARCHAR(50),
    media_url TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact colors table
CREATE TABLE artifactcolors (
    artifact_id INT,
    color VARCHAR(50),
    percentage FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
streamlit run app.py
```

The app will launch at `http://localhost:8501`

## Key Code Patterns

### API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(num_records=100):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            'apikey': api_key,
            'size': per_page,
            'page': page,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            artifacts.extend(data.get('records', []))
            
            if len(data.get('records', [])) < per_page:
                break  # No more records
            
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### ETL Pipeline - Transform

```python
import pandas as pd

def transform_artifacts(raw_data):
    """
    Transform raw API data into structured dataframes
    """
    metadata_list = []
    media_list = []
    color_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'division': artifact.get('division'),
            'accessionyear': artifact.get('accessionyear'),
            'technique': artifact.get('technique'),
            'medium': artifact.get('medium'),
            'dated': artifact.get('dated')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        if artifact.get('images'):
            for image in artifact['images']:
                media_list.append({
                    'artifact_id': artifact.get('id'),
                    'media_type': 'image',
                    'media_url': image.get('baseimageurl')
                })
        
        # Extract color information
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_list.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color'),
                    'percentage': color.get('percent')
                })
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(color_list)
    )
```

### Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_db_connection():
    """
    Create database connection
    """
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

def load_data_to_db(metadata_df, media_df, colors_df):
    """
    Load transformed data into database
    """
    connection = create_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    try:
        # Load metadata
        for _, row in metadata_df.iterrows():
            sql = """
            INSERT INTO artifactmetadata 
            (id, title, culture, period, century, classification, 
             department, division, accessionyear, technique, medium, dated)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
            """
            cursor.execute(sql, tuple(row))
        
        # Load media
        for _, row in media_df.iterrows():
            sql = """
            INSERT INTO artifactmedia (artifact_id, media_type, media_url)
            VALUES (%s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        # Load colors
        for _, row in colors_df.iterrows():
            sql = """
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
            """
            cursor.execute(sql, tuple(row))
        
        connection.commit()
        return True
        
    except Error as e:
        print(f"Error loading data: {e}")
        connection.rollback()
        return False
        
    finally:
        cursor.close()
        connection.close()
```

### Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for data collection
    st.sidebar.header("Data Collection")
    num_records = st.sidebar.number_input(
        "Number of Records", 
        min_value=10, 
        max_value=1000, 
        value=100
    )
    
    if st.sidebar.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            artifacts = fetch_artifacts(num_records)
            
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
        with st.spinner("Loading to database..."):
            success = load_data_to_db(metadata_df, media_df, colors_df)
            
        if success:
            st.success(f"Successfully loaded {len(metadata_df)} artifacts!")
    
    # Analytics section
    st.header("Analytics Queries")
    
    queries = {
        "Artifacts by Culture": """
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
        "Top Colors Used": """
            SELECT color, COUNT(*) as count, AVG(percentage) as avg_percentage
            FROM artifactcolors 
            GROUP BY color 
            ORDER BY count DESC 
            LIMIT 10
        """
    }
    
    query_choice = st.selectbox("Select Query", list(queries.keys()))
    
    if st.button("Run Query"):
        connection = create_db_connection()
        if connection:
            df_result = pd.read_sql(queries[query_choice], connection)
            st.dataframe(df_result)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(
                    df_result, 
                    x=df_result.columns[0], 
                    y=df_result.columns[1],
                    title=query_choice
                )
                st.plotly_chart(fig)
            
            connection.close()

if __name__ == "__main__":
    main()
```

## Common Analytics Queries

```sql
-- Most common artifact classifications
SELECT classification, COUNT(*) as count
FROM artifactmetadata
GROUP BY classification
ORDER BY count DESC
LIMIT 10;

-- Artifacts with images
SELECT 
    COUNT(DISTINCT a.id) as total_artifacts,
    COUNT(DISTINCT m.artifact_id) as with_images
FROM artifactmetadata a
LEFT JOIN artifactmedia m ON a.id = m.artifact_id;

-- Color distribution
SELECT color, SUM(percentage) as total_usage
FROM artifactcolors
GROUP BY color
ORDER BY total_usage DESC;

-- Department-wise distribution
SELECT department, division, COUNT(*) as count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department, division
ORDER BY count DESC;
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:
- Add delays between requests: `time.sleep(0.5)`
- Reduce batch size from 100 to 50
- Check your API key quota

### Database Connection Issues

```python
# Test connection
def test_connection():
    try:
        conn = create_db_connection()
        if conn and conn.is_connected():
            print("Connection successful!")
            conn.close()
            return True
    except Exception as e:
        print(f"Connection failed: {e}")
        return False
```

### Missing Data Handling

```python
# Handle None/null values during transform
def safe_get(dictionary, key, default=None):
    value = dictionary.get(key, default)
    return value if value != "" else default
```

### Performance Optimization

For large datasets, use batch inserts:

```python
# Batch insert example
def batch_insert(cursor, sql, data, batch_size=1000):
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        cursor.executemany(sql, batch)
```

## Best Practices

1. **Always use environment variables** for credentials
2. **Implement error handling** for API requests
3. **Use transactions** for database operations
4. **Cache API responses** to avoid redundant calls
5. **Validate data** before loading to database
6. **Index foreign keys** for query performance
7. **Use parameterized queries** to prevent SQL injection
