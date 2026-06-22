---
name: harvard-art-museums-data-pipeline
description: Build end-to-end data engineering pipelines with the Harvard Art Museums API using Python, ETL workflows, SQL analytics, and Streamlit visualization
triggers:
  - "set up Harvard Art Museums data pipeline"
  - "create ETL pipeline for museum artifact data"
  - "build analytics dashboard with Harvard API"
  - "extract and analyze Harvard Art Museums data"
  - "setup artifact collection database and visualization"
  - "implement museum data engineering workflow"
  - "create Streamlit app for art museum analytics"
  - "configure Harvard Art Museums API integration"
---

# Harvard Art Museums Data Pipeline Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build complete data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard-Artifacts-Collection-Data-Engineering-Analytics-App provides:

- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads nested JSON into relational database tables
- **SQL Database**: Structured schema with artifact metadata, media, and color data
- **Analytics Queries**: 20+ predefined SQL queries for artifact analysis
- **Interactive Visualization**: Streamlit dashboard with Plotly charts

The architecture follows: `API → ETL → SQL → Analytics → Visualization`

## Installation

### Prerequisites

- Python 3.8+
- MySQL or TiDB Cloud database
- Harvard Art Museums API key (get from https://www.harvardartmuseums.org/collections/api)

### Setup

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="harvard_artifacts"
```

### Dependencies

```txt
streamlit
pandas
requests
mysql-connector-python
plotly
python-dotenv
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Database Schema Design

### Table: artifactmetadata

```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(100),
    classification VARCHAR(255),
    department VARCHAR(255),
    dated VARCHAR(255),
    period VARCHAR(255),
    technique VARCHAR(500),
    description TEXT,
    provenance TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Table: artifactmedia

```sql
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    base_image_url VARCHAR(1000),
    primary_image_url VARCHAR(1000),
    thumbnail_url VARCHAR(1000),
    total_images INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### Table: artifactcolors

```sql
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(7),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Implementation

### Extract: Fetch Data from API

```python
import requests
import os
from typing import List, Dict

def fetch_artifacts(api_key: str, size: int = 100, page: int = 1) -> Dict:
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key from environment
        size: Number of records per page
        page: Page number for pagination
    
    Returns:
        JSON response with artifact data
    """
    base_url = "https://api.harvardartmuseums.org/object"
    params = {
        'apikey': api_key,
        'size': size,
        'page': page,
        'hasimage': 1  # Only artifacts with images
    }
    
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    return response.json()

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, size=50, page=1)
artifacts = data.get('records', [])
```

### Transform: Clean and Structure Data

```python
import pandas as pd
from typing import List, Tuple

def transform_artifacts(artifacts: List[Dict]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """
    Transform raw artifact JSON into three normalized DataFrames
    
    Returns:
        Tuple of (metadata_df, media_df, colors_df)
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in artifacts:
        # Extract metadata
        metadata = {
            'id': artifact.get('id'),
            'title': artifact.get('title', '')[:500],
            'culture': artifact.get('culture', '')[:255],
            'century': artifact.get('century', '')[:100],
            'classification': artifact.get('classification', '')[:255],
            'department': artifact.get('department', '')[:255],
            'dated': artifact.get('dated', '')[:255],
            'period': artifact.get('period', '')[:255],
            'technique': artifact.get('technique', '')[:500],
            'description': artifact.get('description', ''),
            'provenance': artifact.get('provenance', '')
        }
        metadata_list.append(metadata)
        
        # Extract media information
        media = {
            'artifact_id': artifact.get('id'),
            'base_image_url': artifact.get('baseimageurl', ''),
            'primary_image_url': artifact.get('primaryimageurl', ''),
            'thumbnail_url': artifact.get('thumbnailurl', ''),
            'total_images': artifact.get('totalpageviews', 0)
        }
        media_list.append(media)
        
        # Extract color data
        colors = artifact.get('colors', [])
        for color in colors:
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex', ''),
                'color_percent': color.get('percent', 0.0)
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )
```

### Load: Insert into SQL Database

```python
import mysql.connector
from mysql.connector import Error
import pandas as pd

def get_db_connection():
    """Create database connection using environment variables"""
    return mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD'),
        database=os.getenv('DB_NAME')
    )

def load_to_database(metadata_df: pd.DataFrame, 
                     media_df: pd.DataFrame, 
                     colors_df: pd.DataFrame):
    """
    Load DataFrames into SQL database using batch inserts
    """
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        # Insert metadata (batch insert for performance)
        metadata_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, 
         dated, period, technique, description, provenance)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
        """
        cursor.executemany(metadata_query, metadata_df.values.tolist())
        
        # Insert media
        media_query = """
        INSERT INTO artifactmedia 
        (artifact_id, base_image_url, primary_image_url, thumbnail_url, total_images)
        VALUES (%s, %s, %s, %s, %s)
        """
        cursor.executemany(media_query, media_df.values.tolist())
        
        # Insert colors
        if not colors_df.empty:
            colors_query = """
            INSERT INTO artifactcolors 
            (artifact_id, color_hex, color_percent)
            VALUES (%s, %s, %s)
            """
            cursor.executemany(colors_query, colors_df.values.tolist())
        
        conn.commit()
        print(f"Loaded {len(metadata_df)} artifacts successfully")
        
    except Error as e:
        print(f"Database error: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```

## Analytics SQL Queries

### Example Analytical Queries

```python
# Query 1: Top 10 cultures by artifact count
query_cultures = """
SELECT culture, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE culture IS NOT NULL AND culture != ''
GROUP BY culture
ORDER BY artifact_count DESC
LIMIT 10;
"""

# Query 2: Artifacts with images vs without
query_images = """
SELECT 
    CASE 
        WHEN primary_image_url IS NOT NULL AND primary_image_url != '' 
        THEN 'With Images' 
        ELSE 'Without Images' 
    END as image_status,
    COUNT(*) as count
FROM artifactmedia
GROUP BY image_status;
"""

# Query 3: Most common colors across all artifacts
query_colors = """
SELECT color_hex, COUNT(*) as usage_count, AVG(color_percent) as avg_percent
FROM artifactcolors
GROUP BY color_hex
ORDER BY usage_count DESC
LIMIT 15;
"""

# Query 4: Artifacts by century
query_century = """
SELECT century, COUNT(*) as count
FROM artifactmetadata
WHERE century IS NOT NULL
GROUP BY century
ORDER BY count DESC;
"""

# Query 5: Department distribution
query_departments = """
SELECT department, COUNT(*) as artifact_count
FROM artifactmetadata
WHERE department IS NOT NULL
GROUP BY department
ORDER BY artifact_count DESC;
"""
```

### Execute Queries and Return Results

```python
def execute_analytics_query(query: str) -> pd.DataFrame:
    """Execute SQL query and return results as DataFrame"""
    conn = get_db_connection()
    try:
        df = pd.read_sql(query, conn)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return pd.DataFrame()
    finally:
        conn.close()

# Usage
results = execute_analytics_query(query_cultures)
print(results.head())
```

## Streamlit Dashboard Implementation

### Basic App Structure

```python
import streamlit as st
import plotly.express as px
import plotly.graph_objects as go

def main():
    st.set_page_config(
        page_title="Harvard Art Museums Analytics",
        page_icon="🎨",
        layout="wide"
    )
    
    st.title("🎨 Harvard Art Museums Collection Analytics")
    st.markdown("---")
    
    # Sidebar navigation
    page = st.sidebar.selectbox(
        "Choose a Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        data_collection_page()
    elif page == "SQL Analytics":
        analytics_page()
    else:
        visualization_page()

if __name__ == "__main__":
    main()
```

### Data Collection Page

```python
def data_collection_page():
    st.header("📥 Data Collection from API")
    
    col1, col2 = st.columns(2)
    with col1:
        num_records = st.number_input("Number of records to fetch", 
                                       min_value=10, max_value=100, value=50)
    with col2:
        page_num = st.number_input("Page number", min_value=1, value=1)
    
    if st.button("Fetch and Load Data"):
        with st.spinner("Fetching data from API..."):
            api_key = os.getenv('HARVARD_API_KEY')
            data = fetch_artifacts(api_key, size=num_records, page=page_num)
            artifacts = data.get('records', [])
            
            st.success(f"Fetched {len(artifacts)} artifacts")
            
            # Transform
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
            # Load to database
            load_to_database(metadata_df, media_df, colors_df)
            
            st.success("✅ Data loaded successfully!")
            st.dataframe(metadata_df.head())
```

### Analytics Page with Dynamic Queries

```python
def analytics_page():
    st.header("📊 SQL Analytics Dashboard")
    
    queries = {
        "Top 10 Cultures": query_cultures,
        "Image Availability": query_images,
        "Popular Colors": query_colors,
        "Century Distribution": query_century,
        "Department Breakdown": query_departments
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            query = queries[selected_query]
            results = execute_analytics_query(query)
            
            if not results.empty:
                st.dataframe(results)
                
                # Auto-generate visualization
                if len(results.columns) == 2:
                    fig = px.bar(results, 
                                 x=results.columns[0], 
                                 y=results.columns[1],
                                 title=selected_query)
                    st.plotly_chart(fig, use_container_width=True)
```

### Visualization Page

```python
def visualization_page():
    st.header("📈 Interactive Visualizations")
    
    # Color distribution pie chart
    color_data = execute_analytics_query(query_colors)
    if not color_data.empty:
        fig = go.Figure(data=[go.Pie(
            labels=color_data['color_hex'],
            values=color_data['usage_count'],
            marker=dict(colors=color_data['color_hex'])
        )])
        fig.update_layout(title="Color Distribution in Artifacts")
        st.plotly_chart(fig, use_container_width=True)
    
    # Department comparison
    dept_data = execute_analytics_query(query_departments)
    if not dept_data.empty:
        fig = px.bar(dept_data, 
                     x='department', 
                     y='artifact_count',
                     title="Artifacts by Department",
                     color='artifact_count',
                     color_continuous_scale='Viridis')
        st.plotly_chart(fig, use_container_width=True)
```

## Common Patterns

### Rate Limiting for API Calls

```python
import time

def fetch_with_rate_limit(api_key: str, total_pages: int, delay: float = 1.0):
    """Fetch multiple pages with rate limiting"""
    all_artifacts = []
    
    for page in range(1, total_pages + 1):
        data = fetch_artifacts(api_key, size=100, page=page)
        all_artifacts.extend(data.get('records', []))
        
        if page < total_pages:
            time.sleep(delay)  # Respect API rate limits
    
    return all_artifacts
```

### Error Handling for ETL Pipeline

```python
def safe_etl_pipeline(api_key: str, num_records: int = 100):
    """ETL pipeline with comprehensive error handling"""
    try:
        # Extract
        data = fetch_artifacts(api_key, size=num_records)
        artifacts = data.get('records', [])
        
        if not artifacts:
            raise ValueError("No artifacts fetched from API")
        
        # Transform
        metadata_df, media_df, colors_df = transform_artifacts(artifacts)
        
        # Validate
        if metadata_df.empty:
            raise ValueError("Transformation produced empty DataFrame")
        
        # Load
        load_to_database(metadata_df, media_df, colors_df)
        
        return len(artifacts)
        
    except requests.RequestException as e:
        print(f"API request failed: {e}")
        return 0
    except Error as e:
        print(f"Database error: {e}")
        return 0
    except Exception as e:
        print(f"Unexpected error: {e}")
        return 0
```

## Configuration

### Environment Variables (.env file)

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts
DB_PORT=3306

# Optional: API Rate Limiting
API_DELAY_SECONDS=1.0
MAX_RETRIES=3
```

### Load Configuration in Code

```python
from dotenv import load_dotenv
import os

load_dotenv()

CONFIG = {
    'api_key': os.getenv('HARVARD_API_KEY'),
    'db_host': os.getenv('DB_HOST'),
    'db_user': os.getenv('DB_USER'),
    'db_password': os.getenv('DB_PASSWORD'),
    'db_name': os.getenv('DB_NAME'),
    'db_port': int(os.getenv('DB_PORT', 3306)),
    'api_delay': float(os.getenv('API_DELAY_SECONDS', 1.0))
}
```

## Troubleshooting

### API Connection Issues

```python
# Test API connectivity
def test_api_connection(api_key: str) -> bool:
    """Verify API key and connection"""
    try:
        response = requests.get(
            "https://api.harvardartmuseums.org/object",
            params={'apikey': api_key, 'size': 1},
            timeout=10
        )
        response.raise_for_status()
        return True
    except requests.RequestException as e:
        print(f"API connection failed: {e}")
        return False
```

### Database Connection Issues

```python
def test_database_connection() -> bool:
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.close()
        conn.close()
        return True
    except Error as e:
        print(f"Database connection failed: {e}")
        return False
```

### Handle Missing Data

```python
def safe_get(obj: dict, key: str, default='', max_length: int = None) -> str:
    """Safely extract and truncate string values"""
    value = obj.get(key, default)
    if value is None:
        return default
    value = str(value)
    if max_length and len(value) > max_length:
        return value[:max_length]
    return value
```

### Clear and Reset Database

```python
def reset_database():
    """Clear all tables (use with caution)"""
    conn = get_db_connection()
    cursor = conn.cursor()
    
    try:
        cursor.execute("SET FOREIGN_KEY_CHECKS = 0")
        cursor.execute("TRUNCATE TABLE artifactcolors")
        cursor.execute("TRUNCATE TABLE artifactmedia")
        cursor.execute("TRUNCATE TABLE artifactmetadata")
        cursor.execute("SET FOREIGN_KEY_CHECKS = 1")
        conn.commit()
        print("Database reset successfully")
    except Error as e:
        print(f"Reset failed: {e}")
        conn.rollback()
    finally:
        cursor.close()
        conn.close()
```
