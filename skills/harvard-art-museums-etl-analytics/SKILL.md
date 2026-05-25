---
name: harvard-art-museums-etl-analytics
description: End-to-end ETL pipeline and analytics app for Harvard Art Museums API with SQL storage and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifact data
  - extract and transform Harvard Art Museums API data
  - set up SQL database for art collection analytics
  - visualize museum artifact data with Streamlit
  - implement data engineering pipeline for art collections
  - query and analyze Harvard museum artifacts
  - build interactive dashboard for art museum data
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project demonstrates a production-grade data engineering workflow:
1. **Extract**: Fetch artifact data from Harvard Art Museums API
2. **Transform**: Clean and normalize nested JSON into relational tables
3. **Load**: Store in MySQL/TiDB Cloud with proper schema design
4. **Analyze**: Run SQL analytics queries
5. **Visualize**: Interactive Streamlit dashboard with Plotly charts

The application handles pagination, rate limiting, batch inserts, and foreign key relationships across multiple related tables.

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
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

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Register for a free API key
3. Add to your `.env` file

### Database Setup

The application supports MySQL and TiDB Cloud. Create the database:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;
```

The ETL pipeline will automatically create required tables:
- `artifactmetadata`: Main artifact information
- `artifactmedia`: Media files and URLs
- `artifactcolors`: Color palette data

## Running the Application

```bash
# Start Streamlit dashboard
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Key Components

### 1. API Data Collection

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = f"https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        data = response.json()
        return data['records'], data['info']
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch first page
artifacts, info = fetch_artifacts(page=1, size=100)
print(f"Total artifacts: {info['totalrecords']}")
print(f"Pages: {info['pages']}")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL database connection"""
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

def create_tables(connection):
    """Create database schema"""
    cursor = connection.cursor()
    
    # Artifact metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            classification VARCHAR(255),
            objectnumber VARCHAR(100),
            century VARCHAR(100),
            dated VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            technique VARCHAR(500),
            medium VARCHAR(500),
            description TEXT,
            provenance TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl VARCHAR(1000),
            iiifbaseuri VARCHAR(1000),
            imagetype VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent FLOAT,
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
    """)
    
    connection.commit()

def transform_and_load(artifacts, connection):
    """Transform API data and load into database"""
    cursor = connection.cursor()
    
    for artifact in artifacts:
        # Extract metadata
        metadata = (
            artifact.get('id'),
            artifact.get('title', '')[:500],
            artifact.get('culture', ''),
            artifact.get('classification', ''),
            artifact.get('objectnumber', ''),
            artifact.get('century', ''),
            artifact.get('dated', ''),
            artifact.get('department', ''),
            artifact.get('division', ''),
            artifact.get('technique', ''),
            artifact.get('medium', ''),
            artifact.get('description', ''),
            artifact.get('provenance', '')
        )
        
        # Insert metadata
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, classification, objectnumber, century, 
             dated, department, division, technique, medium, description, provenance)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, metadata)
        
        # Extract and insert media
        if artifact.get('primaryimageurl'):
            media_data = (
                artifact['id'],
                artifact.get('primaryimageurl', ''),
                artifact.get('iiifbaseuri', ''),
                'primary',
                None,
                None
            )
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, iiifbaseuri, imagetype, width, height)
                VALUES (%s, %s, %s, %s, %s, %s)
            """, media_data)
        
        # Extract and insert colors
        if artifact.get('colors'):
            for color in artifact['colors']:
                color_data = (
                    artifact['id'],
                    color.get('color', ''),
                    color.get('spectrum', ''),
                    color.get('hue', ''),
                    color.get('percent', 0.0)
                )
                cursor.execute("""
                    INSERT INTO artifactcolors 
                    (artifact_id, color, spectrum, hue, percent)
                    VALUES (%s, %s, %s, %s, %s)
                """, color_data)
    
    connection.commit()
```

### 3. Batch Data Collection

```python
import time

def collect_all_artifacts(max_pages=10):
    """Collect artifacts with pagination and rate limiting"""
    connection = create_database_connection()
    create_tables(connection)
    
    total_collected = 0
    
    for page in range(1, max_pages + 1):
        try:
            artifacts, info = fetch_artifacts(page=page, size=100)
            transform_and_load(artifacts, connection)
            
            total_collected += len(artifacts)
            print(f"Page {page}/{max_pages}: Collected {len(artifacts)} artifacts")
            
            # Rate limiting
            time.sleep(1)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            break
    
    connection.close()
    print(f"Total artifacts collected: {total_collected}")

# Run collection
collect_all_artifacts(max_pages=10)
```

### 4. Analytics Queries

```python
def run_analytics_query(query_name, connection):
    """Execute predefined analytics queries"""
    
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 20
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'top_departments': """
            SELECT department, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY artifact_count DESC
        """,
        
        'color_distribution': """
            SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
            FROM artifactcolors
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """,
        
        'artifacts_with_images': """
            SELECT 
                COUNT(DISTINCT am.id) as total_artifacts,
                COUNT(DISTINCT media.artifact_id) as with_images,
                ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as percentage
            FROM artifactmetadata am
            LEFT JOIN artifactmedia media ON am.id = media.artifact_id
        """
    }
    
    cursor = connection.cursor()
    cursor.execute(queries[query_name])
    
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    return pd.DataFrame(results, columns=columns)

# Run analytics
connection = create_database_connection()
df = run_analytics_query('artifacts_by_culture', connection)
print(df)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

st.set_page_config(page_title="Harvard Art Museum Analytics", layout="wide")

st.title("🎨 Harvard Art Museums Analytics Dashboard")

# Sidebar navigation
page = st.sidebar.selectbox(
    "Select Page",
    ["Data Collection", "Analytics", "Visualizations"]
)

if page == "Data Collection":
    st.header("📥 Data Collection")
    
    num_pages = st.number_input("Number of pages to collect", 1, 50, 10)
    
    if st.button("Start Collection"):
        with st.spinner("Collecting artifacts..."):
            collect_all_artifacts(max_pages=num_pages)
            st.success(f"Successfully collected {num_pages * 100} artifacts!")

elif page == "Analytics":
    st.header("📊 SQL Analytics")
    
    query_option = st.selectbox(
        "Select Query",
        [
            "artifacts_by_culture",
            "artifacts_by_century",
            "top_departments",
            "color_distribution",
            "artifacts_with_images"
        ]
    )
    
    if st.button("Run Query"):
        connection = create_database_connection()
        df = run_analytics_query(query_option, connection)
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df) > 0 and len(df.columns) >= 2:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                         title=f"Visualization: {query_option}")
            st.plotly_chart(fig, use_container_width=True)

elif page == "Visualizations":
    st.header("📈 Interactive Visualizations")
    
    connection = create_database_connection()
    
    col1, col2 = st.columns(2)
    
    with col1:
        df_culture = run_analytics_query('artifacts_by_culture', connection)
        fig = px.pie(df_culture, values='count', names='culture', 
                     title='Artifacts by Culture')
        st.plotly_chart(fig, use_container_width=True)
    
    with col2:
        df_colors = run_analytics_query('color_distribution', connection)
        fig = px.bar(df_colors, x='color', y='frequency', 
                     title='Color Distribution')
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Error Handling in ETL

```python
def safe_transform_and_load(artifacts, connection):
    """ETL with error handling"""
    success_count = 0
    error_count = 0
    
    for artifact in artifacts:
        try:
            # Transform and load logic here
            transform_and_load([artifact], connection)
            success_count += 1
        except Exception as e:
            error_count += 1
            print(f"Error processing artifact {artifact.get('id')}: {e}")
    
    return success_count, error_count
```

### Incremental Data Loading

```python
def get_last_artifact_id(connection):
    """Get the last loaded artifact ID"""
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(id) FROM artifactmetadata")
    result = cursor.fetchone()
    return result[0] if result[0] else 0

def incremental_load(connection):
    """Load only new artifacts"""
    last_id = get_last_artifact_id(connection)
    
    # Fetch artifacts with ID > last_id
    # (API filtering logic would go here)
    pass
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time
time.sleep(1)  # 1 second between API calls
```

### Database Connection Issues
```python
# Test connection
try:
    connection = create_database_connection()
    print("Database connected successfully")
except Exception as e:
    print(f"Connection failed: {e}")
```

### Memory Issues with Large Datasets
```python
# Process in smaller batches
def collect_in_batches(total_pages, batch_size=5):
    for i in range(0, total_pages, batch_size):
        collect_all_artifacts(max_pages=min(batch_size, total_pages - i))
```

### Encoding Issues
```python
# Handle special characters
def clean_text(text):
    if text is None:
        return ''
    return text.encode('utf-8', 'ignore').decode('utf-8')[:500]
```

## Best Practices

1. **Always use environment variables** for sensitive data
2. **Implement batch processing** for large datasets
3. **Add proper error handling** at each ETL stage
4. **Use connection pooling** for better performance
5. **Log all operations** for debugging
6. **Test queries on small datasets** before full runs
7. **Back up your database** before major ETL runs
