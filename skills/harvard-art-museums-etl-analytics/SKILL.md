---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline using Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline with Harvard Art Museums API
  - create analytics dashboard for museum artifacts
  - set up data engineering project with Streamlit
  - fetch and analyze Harvard museum collection data
  - implement SQL analytics for art museum data
  - visualize artifact metadata with Python
  - create museum data warehouse pipeline
  - build art collection analytics application
---

# Harvard Art Museums ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides a complete data engineering and analytics solution for the Harvard Art Museums collection. It demonstrates ETL pipelines, SQL database design, analytical queries, and interactive dashboards using Streamlit.

## What It Does

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Transforms nested JSON into normalized relational tables
- **SQL Storage**: Stores data in MySQL/TiDB Cloud with proper schema design
- **Analytics**: Runs 20+ predefined SQL queries for insights
- **Visualization**: Interactive Plotly charts through Streamlit interface

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
requests
mysql-connector-python
plotly
python-dotenv
```

## Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit: https://www.harvardartmuseums.org/collections/api
2. Request an API key
3. Add to `.env` file

## Database Schema

The application creates three main tables:

```sql
-- Artifact metadata table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    technique VARCHAR(500),
    medium VARCHAR(500),
    dated VARCHAR(255),
    period VARCHAR(255),
    division VARCHAR(255)
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
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py
```

The app will be available at `http://localhost:8501`

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd
import os

def fetch_artifacts(api_key, num_pages=5):
    """
    Fetch artifacts from Harvard Art Museums API
    """
    base_url = "https://api.harvardartmuseums.org/object"
    all_artifacts = []
    
    for page in range(1, num_pages + 1):
        params = {
            'apikey': api_key,
            'page': page,
            'size': 100,  # Max per page
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            all_artifacts.extend(data['records'])
        else:
            print(f"Error fetching page {page}: {response.status_code}")
            break
    
    return all_artifacts
```

### 2. ETL Pipeline

```python
import mysql.connector
from dotenv import load_dotenv

def transform_artifacts(artifacts):
    """
    Transform nested JSON into structured dataframes
    """
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'medium': artifact.get('medium', '')[:500],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'division': artifact.get('division', '')[:255]
        })
        
        # Extract media
        if 'images' in artifact and artifact['images']:
            for image in artifact['images']:
                media_records.append({
                    'artifact_id': artifact.get('id'),
                    'image_url': image.get('baseimageurl', ''),
                    'media_type': 'image'
                })
        
        # Extract colors
        if 'colors' in artifact and artifact['colors']:
            for color in artifact['colors']:
                color_records.append({
                    'artifact_id': artifact.get('id'),
                    'color': color.get('color', ''),
                    'percentage': color.get('percent', 0)
                })
    
    return (
        pd.DataFrame(metadata_records),
        pd.DataFrame(media_records),
        pd.DataFrame(color_records)
    )

def load_to_database(metadata_df, media_df, colors_df):
    """
    Load dataframes into MySQL database
    """
    load_dotenv()
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    cursor = conn.cursor()
    
    # Insert metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, 
             technique, medium, dated, period, division)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE title=VALUES(title)
        """, tuple(row))
    
    # Insert media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, image_url, media_type)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, percentage)
            VALUES (%s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
```

### 3. Analytical Queries

```python
def execute_analytical_query(query_name):
    """
    Execute predefined analytical queries
    """
    queries = {
        'artifacts_by_culture': """
            SELECT culture, COUNT(*) as count
            FROM artifactmetadata
            WHERE culture IS NOT NULL AND culture != ''
            GROUP BY culture
            ORDER BY count DESC
            LIMIT 10
        """,
        
        'artifacts_by_century': """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL AND century != ''
            GROUP BY century
            ORDER BY count DESC
        """,
        
        'top_colors': """
            SELECT color, 
                   COUNT(*) as artifact_count,
                   AVG(percentage) as avg_percentage
            FROM artifactcolors
            GROUP BY color
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        'media_availability': """
            SELECT 
                COUNT(DISTINCT m.artifact_id) as with_media,
                (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
                ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
                      (SELECT COUNT(*) FROM artifactmetadata), 2) as percentage
            FROM artifactmedia m
        """,
        
        'department_distribution': """
            SELECT department, COUNT(*) as count
            FROM artifactmetadata
            WHERE department IS NOT NULL AND department != ''
            GROUP BY department
            ORDER BY count DESC
        """
    }
    
    load_dotenv()
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(queries[query_name], conn)
    conn.close()
    
    return df
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    api_key = st.sidebar.text_input("Harvard API Key", type="password")
    
    # ETL Section
    st.header("📥 Data Collection & ETL")
    num_pages = st.slider("Number of pages to fetch", 1, 10, 5)
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_artifacts(api_key, num_pages)
            st.success(f"Fetched {len(artifacts)} artifacts")
        
        with st.spinner("Transforming data..."):
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            st.success("Data transformed")
        
        with st.spinner("Loading to database..."):
            load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded successfully!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_option = st.selectbox(
        "Select Analysis",
        ['artifacts_by_culture', 'artifacts_by_century', 
         'top_colors', 'media_availability', 'department_distribution']
    )
    
    if st.button("Run Analysis"):
        df = execute_analytical_query(query_option)
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(df)
        
        # Visualization
        if len(df) > 0 and len(df.columns) >= 2:
            fig = px.bar(
                df, 
                x=df.columns[0], 
                y=df.columns[1],
                title=f"Analysis: {query_option.replace('_', ' ').title()}"
            )
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common Patterns

### Batch Processing for Large Collections

```python
def batch_insert(cursor, table, records, batch_size=1000):
    """
    Insert records in batches for performance
    """
    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        # Execute batch insert
        cursor.executemany(insert_query, batch)
```

### Error Handling for API Calls

```python
import time

def fetch_with_retry(url, params, max_retries=3):
    """
    Fetch with exponential backoff
    """
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(2 ** attempt)
```

## Troubleshooting

### API Rate Limiting
- Harvard API has rate limits; add delays between requests
- Use `time.sleep(0.5)` between pagination calls

### Database Connection Issues
- Verify `.env` file contains correct credentials
- Check firewall rules for MySQL port (3306)
- For TiDB Cloud, ensure IP whitelist is configured

### Data Type Mismatches
- Truncate long strings to match VARCHAR limits
- Handle NULL values before insertion
- Use `.get()` with defaults when accessing JSON keys

### Memory Issues with Large Datasets
- Process data in chunks instead of loading all at once
- Use database cursors for large result sets
- Clear dataframes after insertion: `del df`

## Advanced Usage

### Custom Query Builder

```python
def build_custom_query(filters):
    """
    Build dynamic SQL queries based on user filters
    """
    query = "SELECT * FROM artifactmetadata WHERE 1=1"
    
    if filters.get('culture'):
        query += f" AND culture = '{filters['culture']}'"
    if filters.get('century'):
        query += f" AND century = '{filters['century']}'"
    
    return query
```

### Export Results

```python
def export_to_csv(df, filename):
    """
    Export analysis results to CSV
    """
    df.to_csv(filename, index=False)
    st.download_button(
        label="Download CSV",
        data=df.to_csv(index=False),
        file_name=filename,
        mime='text/csv'
    )
```
