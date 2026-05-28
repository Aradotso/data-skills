---
name: harvard-art-museums-etl-analytics
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with the Harvard Art Museums API
  - set up a data engineering project with Harvard artifacts data
  - create analytics dashboard for museum collection data
  - extract and transform Harvard Art Museums API data
  - build a Streamlit app for artifact analytics
  - configure SQL database for Harvard museum artifacts
  - visualize museum collection data with Plotly
  - implement data pipeline for art museum collections
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates ETL pipelines, SQL analytics, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Data Collection**: Fetch artifact metadata from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON into relational SQL tables
- **SQL Analytics**: 20+ predefined analytical queries for artifact insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts
- **Database Design**: Proper relational schema with foreign key relationships

**Architecture**: API → ETL → SQL → Analytics → Visualization

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
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
```

### Get Harvard API Key

1. Visit https://harvardartmuseums.org/collections/api
2. Register for a free API key
3. Add to `.env` file

### Database Setup

```python
import mysql.connector
from dotenv import load_dotenv
import os

load_dotenv()

# Create database connection
conn = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    port=int(os.getenv('DB_PORT', 3306)),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)

cursor = conn.cursor()

# Create database
cursor.execute(f"CREATE DATABASE IF NOT EXISTS {os.getenv('DB_NAME')}")
cursor.execute(f"USE {os.getenv('DB_NAME')}")

# Create tables
cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    technique VARCHAR(300),
    period VARCHAR(200),
    dated VARCHAR(200),
    url TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactmedia (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    media_type VARCHAR(100),
    baseimageurl TEXT,
    iiifbaseuri TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS artifactcolors (
    id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    hue VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
)
""")

conn.commit()
cursor.close()
conn.close()
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    api_key = os.getenv('HARVARD_API_KEY')
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        'apikey': api_key,
        'page': page,
        'size': size,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API request failed: {response.status_code}")

# Fetch multiple pages with pagination
def fetch_all_artifacts(total_records=1000):
    """Fetch artifacts with pagination"""
    all_artifacts = []
    page_size = 100
    total_pages = (total_records // page_size) + 1
    
    for page in range(1, total_pages + 1):
        print(f"Fetching page {page}/{total_pages}")
        data = fetch_artifacts(page=page, size=page_size)
        all_artifacts.extend(data.get('records', []))
        
        # Respect rate limits
        import time
        time.sleep(0.5)
    
    return all_artifacts[:total_records]
```

### Transform: Process JSON to DataFrames

```python
import pandas as pd

def transform_artifact_data(artifacts):
    """Transform artifact JSON into relational dataframes"""
    
    # Metadata table
    metadata_records = []
    media_records = []
    color_records = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata_records.append({
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:200],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:200],
            'department': artifact.get('department', '')[:200],
            'technique': artifact.get('technique', '')[:300],
            'period': artifact.get('period', '')[:200],
            'dated': artifact.get('dated', '')[:200],
            'url': artifact.get('url', '')
        })
        
        # Extract media
        primary_image = artifact.get('primaryimageurl')
        if primary_image:
            media_records.append({
                'artifact_id': artifact.get('id'),
                'media_type': 'primary_image',
                'baseimageurl': primary_image,
                'iiifbaseuri': artifact.get('iiifbaseuri', '')
            })
        
        # Extract colors
        colors = artifact.get('colors', [])
        for color in colors:
            color_records.append({
                'artifact_id': artifact.get('id'),
                'color': color.get('color', ''),
                'spectrum': color.get('spectrum', ''),
                'hue': color.get('hue', ''),
                'percent': color.get('percent', 0.0)
            })
    
    df_metadata = pd.DataFrame(metadata_records)
    df_media = pd.DataFrame(media_records)
    df_colors = pd.DataFrame(color_records)
    
    return df_metadata, df_media, df_colors
```

### Load: Insert into SQL Database

```python
def load_to_database(df_metadata, df_media, df_colors):
    """Load dataframes into MySQL database"""
    import mysql.connector
    from dotenv import load_dotenv
    import os
    
    load_dotenv()
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    cursor = conn.cursor()
    
    # Insert metadata (with ON DUPLICATE KEY UPDATE for idempotency)
    for _, row in df_metadata.iterrows():
        cursor.execute("""
            INSERT INTO artifactmetadata 
            (id, title, culture, century, classification, department, technique, period, dated, url)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON DUPLICATE KEY UPDATE
            title=VALUES(title), culture=VALUES(culture)
        """, tuple(row))
    
    # Insert media
    for _, row in df_media.iterrows():
        cursor.execute("""
            INSERT INTO artifactmedia (artifact_id, media_type, baseimageurl, iiifbaseuri)
            VALUES (%s, %s, %s, %s)
        """, tuple(row))
    
    # Insert colors
    for _, row in df_colors.iterrows():
        cursor.execute("""
            INSERT INTO artifactcolors (artifact_id, color, spectrum, hue, percent)
            VALUES (%s, %s, %s, %s, %s)
        """, tuple(row))
    
    conn.commit()
    cursor.close()
    conn.close()
    
    print(f"Loaded {len(df_metadata)} artifacts, {len(df_media)} media, {len(df_colors)} colors")
```

## SQL Analytics Queries

### Common Analytical Queries

```python
ANALYTICAL_QUERIES = {
    "artifacts_by_culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL AND culture != ''
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 15
    """,
    
    "artifacts_by_century": """
        SELECT century, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != ''
        GROUP BY century
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "artifacts_by_department": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "color_distribution": """
        SELECT color, COUNT(*) as frequency
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 20
    """,
    
    "artifacts_with_images": """
        SELECT 
            CASE WHEN m.baseimageurl IS NOT NULL THEN 'With Image' ELSE 'No Image' END as image_status,
            COUNT(*) as count
        FROM artifactmetadata a
        LEFT JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY image_status
    """,
    
    "top_techniques": """
        SELECT technique, COUNT(*) as usage_count
        FROM artifactmetadata
        WHERE technique IS NOT NULL AND technique != ''
        GROUP BY technique
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "dominant_colors_by_culture": """
        SELECT a.culture, c.color, COUNT(*) as color_count
        FROM artifactmetadata a
        JOIN artifactcolors c ON a.id = c.artifact_id
        WHERE a.culture IS NOT NULL AND a.culture != ''
        GROUP BY a.culture, c.color
        ORDER BY a.culture, color_count DESC
    """
}

def execute_query(query_name):
    """Execute analytical query and return results"""
    import mysql.connector
    import pandas as pd
    
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )
    
    df = pd.read_sql(ANALYTICAL_QUERIES[query_name], conn)
    conn.close()
    
    return df
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px
import pandas as pd

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("> End-to-end ETL and Analytics Application")
    
    # Sidebar navigation
    menu = st.sidebar.selectbox(
        "Select Module",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if menu == "Data Collection":
        st.header("📥 ETL Pipeline")
        
        col1, col2 = st.columns(2)
        with col1:
            num_records = st.number_input("Number of artifacts to fetch", 100, 5000, 500)
        with col2:
            if st.button("Run ETL Pipeline"):
                with st.spinner("Fetching data from API..."):
                    artifacts = fetch_all_artifacts(num_records)
                
                with st.spinner("Transforming data..."):
                    df_meta, df_media, df_colors = transform_artifact_data(artifacts)
                
                with st.spinner("Loading to database..."):
                    load_to_database(df_meta, df_media, df_colors)
                
                st.success(f"✅ ETL Complete! Loaded {len(df_meta)} artifacts")
    
    elif menu == "SQL Analytics":
        st.header("📊 SQL Query Analytics")
        
        query_name = st.selectbox(
            "Select Analysis",
            list(ANALYTICAL_QUERIES.keys())
        )
        
        if st.button("Execute Query"):
            df = execute_query(query_name)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Auto-generate visualization
            if len(df.columns) == 2:
                fig = px.bar(
                    df, 
                    x=df.columns[0], 
                    y=df.columns[1],
                    title=f"Analysis: {query_name.replace('_', ' ').title()}"
                )
                st.plotly_chart(fig, use_container_width=True)
    
    elif menu == "Visualizations":
        st.header("📈 Data Visualizations")
        
        # Culture distribution
        df_culture = execute_query("artifacts_by_culture")
        fig1 = px.bar(df_culture, x='culture', y='artifact_count', 
                      title="Artifacts by Culture")
        st.plotly_chart(fig1, use_container_width=True)
        
        # Color distribution
        df_colors = execute_query("color_distribution")
        fig2 = px.pie(df_colors, names='color', values='frequency',
                      title="Color Distribution in Artifacts")
        st.plotly_chart(fig2, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Run the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Batch Processing with Rate Limiting

```python
import time

def batch_fetch_with_limit(total_records, batch_size=100, delay=0.5):
    """Fetch data in batches respecting API rate limits"""
    all_data = []
    
    for i in range(0, total_records, batch_size):
        batch = fetch_artifacts(page=i//batch_size + 1, size=batch_size)
        all_data.extend(batch.get('records', []))
        time.sleep(delay)  # Rate limiting
    
    return all_data
```

### Error Handling for ETL

```python
def safe_etl_pipeline(num_records):
    """ETL pipeline with error handling"""
    try:
        # Extract
        artifacts = fetch_all_artifacts(num_records)
        if not artifacts:
            raise ValueError("No artifacts fetched")
        
        # Transform
        df_meta, df_media, df_colors = transform_artifact_data(artifacts)
        
        # Validate
        assert len(df_meta) > 0, "No metadata records"
        
        # Load
        load_to_database(df_meta, df_media, df_colors)
        
        return True, f"Success: {len(df_meta)} records"
    
    except Exception as e:
        return False, f"Error: {str(e)}"
```

## Troubleshooting

### API Key Issues
```python
# Verify API key is loaded
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.getenv('HARVARD_API_KEY')

if not api_key:
    raise ValueError("HARVARD_API_KEY not found in .env file")

# Test API connection
response = requests.get(
    "https://api.harvardartmuseums.org/object",
    params={'apikey': api_key, 'size': 1}
)
print(f"API Status: {response.status_code}")
```

### Database Connection Issues
```python
# Test database connection
try:
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        port=int(os.getenv('DB_PORT', 3306)),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    print("✅ Database connection successful")
    conn.close()
except Exception as e:
    print(f"❌ Database connection failed: {e}")
```

### Handle Missing Data
```python
def safe_get(dictionary, key, default=''):
    """Safely extract values from nested JSON"""
    value = dictionary.get(key, default)
    return value if value is not None else default

# Use in transformation
metadata_records.append({
    'id': artifact.get('id'),
    'title': safe_get(artifact, 'title', 'Untitled')[:500],
    'culture': safe_get(artifact, 'culture')[:200]
})
```

This skill provides comprehensive guidance for building ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit.
