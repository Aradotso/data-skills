---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for museum data
  - analyze Harvard Art Museums collection
  - create a data engineering pipeline with Streamlit
  - set up museum artifact analytics dashboard
  - extract and transform Harvard API data
  - build SQL analytics for art collection data
  - visualize museum collection insights
  - create artifact data warehouse
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

This project provides a complete data engineering and analytics solution for the Harvard Art Museums collection. It demonstrates:

- **API Integration**: Fetching artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracting, transforming, and loading nested JSON data into relational SQL tables
- **SQL Analytics**: 20+ predefined analytical queries for collection insights
- **Interactive Dashboards**: Streamlit-based visualization with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly sqlalchemy
```

## Configuration

### API Key Setup

Get your free API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

```python
# Store in environment variable
export HARVARD_API_KEY="your_api_key_here"

# Or configure in Streamlit app
# The app typically has a sidebar input for API key configuration
```

### Database Connection

```python
import mysql.connector
import os

# Database configuration
db_config = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'user': os.getenv('DB_USER', 'your_username'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME', 'harvard_artifacts'),
    'port': int(os.getenv('DB_PORT', 3306))
}

# Create connection
conn = mysql.connector.connect(**db_config)
cursor = conn.cursor()
```

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    period VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    totalpageviews INT,
    totaluniquepageviews INT
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    baseimageurl VARCHAR(1000),
    iiifbaseuri VARCHAR(1000),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    spectrum VARCHAR(50),
    percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Key Components

### 1. ETL Pipeline

```python
import requests
import pandas as pd
import time

class HarvardArtETL:
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def extract_artifacts(self, num_pages=10, page_size=100):
        """Extract artifact data with pagination"""
        artifacts = []
        
        for page in range(1, num_pages + 1):
            params = {
                'apikey': self.api_key,
                'size': page_size,
                'page': page
            }
            
            try:
                response = requests.get(self.base_url, params=params)
                response.raise_for_status()
                data = response.json()
                
                artifacts.extend(data.get('records', []))
                
                # Rate limiting
                time.sleep(0.5)
                
            except requests.exceptions.RequestException as e:
                print(f"Error on page {page}: {e}")
                break
        
        return artifacts
    
    def transform_metadata(self, artifacts):
        """Transform artifact data into metadata dataframe"""
        metadata_list = []
        
        for artifact in artifacts:
            metadata = {
                'id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'description': artifact.get('description'),
                'totalpageviews': artifact.get('totalpageviews', 0),
                'totaluniquepageviews': artifact.get('totaluniquepageviews', 0)
            }
            metadata_list.append(metadata)
        
        return pd.DataFrame(metadata_list)
    
    def transform_media(self, artifacts):
        """Transform media data into separate dataframe"""
        media_list = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            primary_image = artifact.get('primaryimageurl')
            
            if primary_image:
                media_list.append({
                    'artifact_id': artifact_id,
                    'baseimageurl': primary_image,
                    'iiifbaseuri': artifact.get('iiifbaseuri')
                })
        
        return pd.DataFrame(media_list)
    
    def transform_colors(self, artifacts):
        """Transform color data into separate dataframe"""
        color_list = []
        
        for artifact in artifacts:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_list.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'percent': color.get('percent')
                })
        
        return pd.DataFrame(color_list)
    
    def load_to_sql(self, df, table_name, conn):
        """Load dataframe to SQL table with batch insert"""
        cursor = conn.cursor()
        
        # Create insert query
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        insert_query = f"INSERT IGNORE INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        # Batch insert
        data = [tuple(row) for row in df.values]
        cursor.executemany(insert_query, data)
        conn.commit()
        
        return cursor.rowcount
```

### 2. Streamlit Application Structure

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Art Analytics", layout="wide")
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("---")
    
    # Sidebar configuration
    with st.sidebar:
        st.header("Configuration")
        api_key = st.text_input("Harvard API Key", type="password", 
                                value=os.getenv('HARVARD_API_KEY', ''))
        
        db_host = st.text_input("Database Host", value="localhost")
        db_user = st.text_input("Database User")
        db_password = st.text_input("Database Password", type="password")
        db_name = st.text_input("Database Name", value="harvard_artifacts")
    
    # Main tabs
    tab1, tab2, tab3 = st.tabs(["📥 ETL Pipeline", "📊 SQL Analytics", "📈 Visualizations"])
    
    with tab1:
        st.header("Extract, Transform, Load")
        
        num_pages = st.slider("Number of pages to extract", 1, 50, 10)
        
        if st.button("Run ETL Pipeline"):
            with st.spinner("Extracting data from API..."):
                etl = HarvardArtETL(api_key)
                artifacts = etl.extract_artifacts(num_pages=num_pages)
                st.success(f"Extracted {len(artifacts)} artifacts")
            
            with st.spinner("Transforming data..."):
                metadata_df = etl.transform_metadata(artifacts)
                media_df = etl.transform_media(artifacts)
                colors_df = etl.transform_colors(artifacts)
                st.success("Transformation complete")
            
            with st.spinner("Loading to database..."):
                conn = get_db_connection(db_host, db_user, db_password, db_name)
                
                rows_metadata = etl.load_to_sql(metadata_df, 'artifactmetadata', conn)
                rows_media = etl.load_to_sql(media_df, 'artifactmedia', conn)
                rows_colors = etl.load_to_sql(colors_df, 'artifactcolors', conn)
                
                conn.close()
                
                st.success(f"Loaded {rows_metadata} metadata, {rows_media} media, {rows_colors} color records")
    
    with tab2:
        st.header("SQL Analytics Dashboard")
        
        query_options = {
            "Artifacts by Culture": """
                SELECT culture, COUNT(*) as count 
                FROM artifactmetadata 
                WHERE culture IS NOT NULL 
                GROUP BY culture 
                ORDER BY count DESC 
                LIMIT 15
            """,
            "Artifacts by Century": """
                SELECT century, COUNT(*) as count 
                FROM artifactmetadata 
                WHERE century IS NOT NULL 
                GROUP BY century 
                ORDER BY count DESC
            """,
            "Top Viewed Artifacts": """
                SELECT title, culture, totalpageviews 
                FROM artifactmetadata 
                ORDER BY totalpageviews DESC 
                LIMIT 20
            """,
            "Media Availability": """
                SELECT 
                    CASE WHEN baseimageurl IS NOT NULL THEN 'Has Image' ELSE 'No Image' END as status,
                    COUNT(*) as count
                FROM artifactmedia
                GROUP BY status
            """,
            "Color Distribution": """
                SELECT spectrum, COUNT(*) as count, AVG(percent) as avg_percent
                FROM artifactcolors
                GROUP BY spectrum
                ORDER BY count DESC
            """
        }
        
        selected_query = st.selectbox("Select Analysis", list(query_options.keys()))
        
        if st.button("Run Query"):
            conn = get_db_connection(db_host, db_user, db_password, db_name)
            df_result = pd.read_sql(query_options[selected_query], conn)
            conn.close()
            
            st.dataframe(df_result, use_container_width=True)
            
            # Auto-generate visualization
            if len(df_result.columns) >= 2:
                fig = px.bar(df_result, x=df_result.columns[0], y=df_result.columns[1],
                            title=selected_query)
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Common SQL Analytics Queries

```sql
-- 1. Most popular departments
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC;

-- 2. Artifacts with complete data
SELECT 
    COUNT(*) as total_artifacts,
    SUM(CASE WHEN description IS NOT NULL THEN 1 ELSE 0 END) as with_description,
    SUM(CASE WHEN culture IS NOT NULL THEN 1 ELSE 0 END) as with_culture
FROM artifactmetadata;

-- 3. Color analysis by culture
SELECT 
    am.culture,
    ac.spectrum,
    COUNT(*) as color_count,
    AVG(ac.percent) as avg_percentage
FROM artifactmetadata am
JOIN artifactcolors ac ON am.id = ac.artifact_id
WHERE am.culture IS NOT NULL
GROUP BY am.culture, ac.spectrum
ORDER BY color_count DESC
LIMIT 30;

-- 4. Image coverage statistics
SELECT 
    am.department,
    COUNT(DISTINCT am.id) as total_artifacts,
    COUNT(DISTINCT amd.artifact_id) as with_images,
    ROUND(COUNT(DISTINCT amd.artifact_id) * 100.0 / COUNT(DISTINCT am.id), 2) as coverage_percent
FROM artifactmetadata am
LEFT JOIN artifactmedia amd ON am.id = amd.artifact_id
GROUP BY am.department
HAVING total_artifacts > 10
ORDER BY coverage_percent DESC;
```

## Running the Application

```bash
# Set environment variables
export HARVARD_API_KEY="your_api_key"
export DB_HOST="localhost"
export DB_USER="your_db_user"
export DB_PASSWORD="your_db_password"
export DB_NAME="harvard_artifacts"

# Run Streamlit app
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Troubleshooting

### API Rate Limiting
```python
# Add delays between requests
import time

for page in range(num_pages):
    response = requests.get(url, params=params)
    time.sleep(0.5)  # 500ms delay
```

### Database Connection Issues
```python
# Test connection
try:
    conn = mysql.connector.connect(**db_config)
    print("Connection successful")
    conn.close()
except mysql.connector.Error as err:
    print(f"Error: {err}")
```

### Memory Management for Large Datasets
```python
# Use chunked processing
def process_in_chunks(artifacts, chunk_size=500):
    for i in range(0, len(artifacts), chunk_size):
        chunk = artifacts[i:i + chunk_size]
        # Process chunk
        yield chunk
```

### Missing Data Handling
```python
# Handle None values during transformation
def safe_get(artifact, key, default='Unknown'):
    value = artifact.get(key)
    return value if value is not None else default
```

## Best Practices

1. **Always use environment variables** for sensitive credentials
2. **Implement batch inserts** for better SQL performance
3. **Add proper error handling** around API calls
4. **Use connection pooling** for database operations in production
5. **Cache API responses** to avoid redundant calls during development
6. **Validate data** before loading to SQL to prevent constraint violations
