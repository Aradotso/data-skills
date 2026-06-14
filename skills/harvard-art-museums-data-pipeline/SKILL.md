---
name: harvard-art-museums-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with SQL and Streamlit
triggers:
  - how do I build a data pipeline with Harvard Art Museums API
  - help me create an ETL workflow for museum artifacts
  - show me how to analyze Harvard museum data with SQL
  - build a Streamlit dashboard for art museum analytics
  - integrate Harvard Art Museums API into my data pipeline
  - create analytical queries for museum artifact data
  - visualize museum collection data with Python
  - extract and transform Harvard API data into SQL
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill helps you build end-to-end data engineering pipelines using the Harvard Art Museums API. The project demonstrates ETL workflows, SQL analytics, and interactive visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:
- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load artifact metadata, media, and color data into relational databases
- **SQL Analytics**: Pre-built analytical queries for insights on artifacts, cultures, departments, and media
- **Interactive Dashboard**: Streamlit-based UI for data collection, query execution, and visualization
- **Data Visualization**: Plotly charts for exploring query results

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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306
```

### Get Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key (free)
3. Add it to your `.env` file

### Database Setup

The app supports MySQL and TiDB Cloud. Create the database:

```sql
CREATE DATABASE harvard_artifacts;
USE harvard_artifacts;
```

The app will auto-create these tables:
- `artifactmetadata`: Core artifact information
- `artifactmedia`: Media files and images
- `artifactcolors`: Color palette data

## Running the Application

```bash
streamlit run app.py
```

Access the dashboard at `http://localhost:8501`

## Architecture & Data Flow

```
Harvard API → Extract → Transform → Load → SQL Database → Analytics → Visualization
```

1. **Extract**: Fetch data from Harvard API with pagination
2. **Transform**: Parse JSON, normalize nested structures, create relational records
3. **Load**: Batch insert into SQL tables with proper relationships
4. **Analytics**: Execute SQL queries for insights
5. **Visualize**: Render interactive charts with Plotly

## Key Code Patterns

### 1. API Data Extraction

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    url = 'https://api.harvardartmuseums.org/object'
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Fetch multiple pages
def fetch_all_artifacts(total_records=500, page_size=100):
    all_artifacts = []
    total_pages = (total_records + page_size - 1) // page_size
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(page=page, size=page_size)
        all_artifacts.extend(data.get('records', []))
        
        # Rate limiting
        import time
        time.sleep(0.5)
    
    return all_artifacts
```

### 2. Data Transformation

```python
import pandas as pd

def transform_artifact_data(raw_artifacts):
    """Transform API JSON into relational dataframes"""
    
    # Metadata table
    metadata_records = []
    for artifact in raw_artifacts:
        metadata_records.append({
            'objectid': artifact.get('objectid'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'period': artifact.get('period'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'dimensions': artifact.get('dimensions')
        })
    
    metadata_df = pd.DataFrame(metadata_records)
    
    # Media table
    media_records = []
    for artifact in raw_artifacts:
        object_id = artifact.get('objectid')
        images = artifact.get('images', [])
        
        for img in images:
            media_records.append({
                'objectid': object_id,
                'baseimageurl': img.get('baseimageurl'),
                'iiifbaseuri': img.get('iiifbaseuri'),
                'format': img.get('format'),
                'width': img.get('width'),
                'height': img.get('height')
            })
    
    media_df = pd.DataFrame(media_records)
    
    # Colors table
    color_records = []
    for artifact in raw_artifacts:
        object_id = artifact.get('objectid')
        colors = artifact.get('colors', [])
        
        for color in colors:
            color_records.append({
                'objectid': object_id,
                'color': color.get('color'),
                'spectrum': color.get('spectrum'),
                'hue': color.get('hue'),
                'percent': color.get('percent')
            })
    
    colors_df = pd.DataFrame(color_records)
    
    return metadata_df, media_df, colors_df
```

### 3. Database Loading

```python
import mysql.connector
from mysql.connector import Error

def create_database_connection():
    """Create MySQL connection"""
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

def create_tables(connection):
    """Create database tables if they don't exist"""
    cursor = connection.cursor()
    
    # Metadata table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            objectid INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(200),
            period VARCHAR(200),
            century VARCHAR(100),
            classification VARCHAR(200),
            department VARCHAR(200),
            dated VARCHAR(200),
            medium TEXT,
            dimensions VARCHAR(500)
        )
    """)
    
    # Media table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            format VARCHAR(50),
            width INT,
            height INT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    # Colors table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            id INT AUTO_INCREMENT PRIMARY KEY,
            objectid INT,
            color VARCHAR(100),
            spectrum VARCHAR(100),
            hue VARCHAR(100),
            percent FLOAT,
            FOREIGN KEY (objectid) REFERENCES artifactmetadata(objectid)
        )
    """)
    
    connection.commit()
    cursor.close()

def load_data_to_sql(metadata_df, media_df, colors_df):
    """Load dataframes into SQL database"""
    connection = create_database_connection()
    if not connection:
        return False
    
    create_tables(connection)
    cursor = connection.cursor()
    
    # Load metadata
    for _, row in metadata_df.iterrows():
        cursor.execute("""
            INSERT IGNORE INTO artifactmetadata 
            (objectid, title, culture, period, century, classification, 
             department, dated, medium, dimensions)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Load media
    for _, row in media_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia 
            (objectid, baseimageurl, iiifbaseuri, format, width, height)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, tuple(row))
    
    # Load colors
    for _, row in colors_df.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors 
            (objectid, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    connection.commit()
    cursor.close()
    connection.close()
    
    return True
```

### 4. Analytics Queries

```python
def execute_analytics_query(query_name):
    """Execute predefined analytical queries"""
    
    queries = {
        "artifacts_by_culture": """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 10
        """,
        
        "artifacts_by_century": """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """,
        
        "top_departments": """
            SELECT department, COUNT(*) as total_artifacts
            FROM artifactmetadata
            WHERE department IS NOT NULL
            GROUP BY department
            ORDER BY total_artifacts DESC
            LIMIT 10
        """,
        
        "color_distribution": """
            SELECT color, COUNT(*) as usage_count
            FROM artifactcolors
            GROUP BY color
            ORDER BY usage_count DESC
            LIMIT 15
        """,
        
        "media_availability": """
            SELECT 
                COUNT(DISTINCT am.objectid) as total_artifacts,
                COUNT(DISTINCT med.objectid) as artifacts_with_media,
                ROUND(COUNT(DISTINCT med.objectid) * 100.0 / COUNT(DISTINCT am.objectid), 2) as media_percentage
            FROM artifactmetadata am
            LEFT JOIN artifactmedia med ON am.objectid = med.objectid
        """
    }
    
    connection = create_database_connection()
    cursor = connection.cursor()
    
    cursor.execute(queries.get(query_name, "SELECT 1"))
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    
    cursor.close()
    connection.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 5. Streamlit Dashboard Integration

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🎨 Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for API configuration
    with st.sidebar:
        st.header("Configuration")
        num_records = st.number_input("Records to fetch", 100, 1000, 500)
        
        if st.button("Fetch & Load Data"):
            with st.spinner("Fetching data from API..."):
                artifacts = fetch_all_artifacts(total_records=num_records)
                metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
                
                success = load_data_to_sql(metadata_df, media_df, colors_df)
                
                if success:
                    st.success(f"✅ Loaded {len(metadata_df)} artifacts!")
                else:
                    st.error("❌ Database loading failed")
    
    # Analytics section
    st.header("📊 Analytics Queries")
    
    query_options = [
        "Artifacts by Culture",
        "Artifacts by Century",
        "Top Departments",
        "Color Distribution",
        "Media Availability"
    ]
    
    selected_query = st.selectbox("Select Analysis", query_options)
    
    if st.button("Run Query"):
        query_map = {
            "Artifacts by Culture": "artifacts_by_culture",
            "Artifacts by Century": "artifacts_by_century",
            "Top Departments": "top_departments",
            "Color Distribution": "color_distribution",
            "Media Availability": "media_availability"
        }
        
        df = execute_analytics_query(query_map[selected_query])
        
        st.dataframe(df)
        
        # Auto-generate visualization
        if len(df.columns) == 2 and len(df) > 1:
            fig = px.bar(df, x=df.columns[0], y=df.columns[1], 
                        title=selected_query)
            st.plotly_chart(fig)

if __name__ == "__main__":
    main()
```

## Common Use Cases

### Build Complete ETL Pipeline

```python
# Full pipeline execution
artifacts = fetch_all_artifacts(total_records=1000)
metadata_df, media_df, colors_df = transform_artifact_data(artifacts)
load_data_to_sql(metadata_df, media_df, colors_df)
```

### Custom Analytics Query

```python
def custom_query(sql):
    connection = create_database_connection()
    df = pd.read_sql(sql, connection)
    connection.close()
    return df

# Example: Find artifacts with specific colors
result = custom_query("""
    SELECT am.title, ac.color, ac.percent
    FROM artifactmetadata am
    JOIN artifactcolors ac ON am.objectid = ac.objectid
    WHERE ac.color = 'blue'
    ORDER BY ac.percent DESC
    LIMIT 20
""")
```

## Troubleshooting

### API Rate Limiting

If you hit rate limits, add delays:

```python
import time
time.sleep(1)  # Wait 1 second between requests
```

### Database Connection Issues

Verify credentials:

```python
connection = create_database_connection()
if connection and connection.is_connected():
    print("✅ Database connected")
else:
    print("❌ Connection failed - check .env file")
```

### Empty API Results

Check API key validity and parameters:

```python
response = fetch_artifacts(page=1, size=10)
print(f"Total records available: {response.get('info', {}).get('totalrecords')}")
```

### Missing Data in Tables

Ensure foreign key relationships:

```python
# Check data integrity
cursor.execute("""
    SELECT COUNT(*) FROM artifactmedia
    WHERE objectid NOT IN (SELECT objectid FROM artifactmetadata)
""")
orphaned_records = cursor.fetchone()[0]
print(f"Orphaned media records: {orphaned_records}")
```

## Advanced Patterns

### Incremental Data Loading

```python
def get_last_loaded_id():
    connection = create_database_connection()
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(objectid) FROM artifactmetadata")
    result = cursor.fetchone()[0]
    cursor.close()
    connection.close()
    return result or 0

def incremental_load():
    last_id = get_last_loaded_id()
    # Fetch only new artifacts
    # Implementation depends on API filtering capabilities
```

### Export Analytics Results

```python
def export_query_results(query_name, output_format='csv'):
    df = execute_analytics_query(query_name)
    
    if output_format == 'csv':
        df.to_csv(f'{query_name}.csv', index=False)
    elif output_format == 'excel':
        df.to_excel(f'{query_name}.xlsx', index=False)
    
    return f"Exported {len(df)} rows"
```

This skill enables you to build production-ready data pipelines for cultural heritage data with proper ETL practices, SQL analytics, and interactive visualization.
