---
name: harvard-artifacts-collection-data-engineering-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard with Harvard artifacts API
  - set up data engineering pipeline with museum collections
  - visualize Harvard Art Museums data with Streamlit
  - query and analyze art museum collection data
  - extract and transform Harvard artifacts metadata
  - build SQL analytics for museum artifact data
  - create interactive visualization for art collections
---

# Harvard Artifacts Collection Data Engineering Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive data visualization with Streamlit.

## What This Project Does

The Harvard Artifacts Collection app provides a complete data pipeline from API to visualization:

1. **Data Collection**: Fetches artifact metadata from Harvard Art Museums API with pagination and rate limiting
2. **ETL Pipeline**: Transforms nested JSON into structured relational tables
3. **SQL Storage**: Stores data in MySQL/TiDB with proper schema design and foreign keys
4. **Analytics**: Executes 20+ predefined SQL queries for insights
5. **Visualization**: Displays results through interactive Plotly charts in Streamlit

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Set up environment variables
export HARVARD_API_KEY="your_api_key_here"
export DB_HOST="your_database_host"
export DB_USER="your_database_user"
export DB_PASSWORD="your_database_password"
export DB_NAME="your_database_name"
```

**Required packages:**
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

# The app will be available at http://localhost:8501
```

## Database Schema

The application uses three main tables with relational integrity:

```sql
-- Artifact Metadata Table
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    dated VARCHAR(200),
    accession_number VARCHAR(100),
    url VARCHAR(500)
);

-- Artifact Media Table
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    media_type VARCHAR(100),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors Table
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## ETL Pipeline Pattern

### Extract: Fetching Data from Harvard API

```python
import requests
import os

def fetch_artifacts(api_key, page=1, size=100):
    """Fetch artifacts from Harvard Art Museums API"""
    base_url = "https://api.harvardartmuseums.org/object"
    
    params = {
        "apikey": api_key,
        "page": page,
        "size": size
    }
    
    response = requests.get(base_url, params=params)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f"API Error: {response.status_code}")

# Usage
api_key = os.getenv("HARVARD_API_KEY")
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data.get("records", [])
```

### Transform: Processing Nested JSON

```python
import pandas as pd

def transform_artifact_metadata(artifacts):
    """Transform artifact data into structured format"""
    metadata_list = []
    
    for artifact in artifacts:
        metadata = {
            "id": artifact.get("id"),
            "title": artifact.get("title", "Unknown"),
            "culture": artifact.get("culture", "Unknown"),
            "century": artifact.get("century", "Unknown"),
            "classification": artifact.get("classification", "Unknown"),
            "department": artifact.get("department", "Unknown"),
            "dated": artifact.get("dated", "Unknown"),
            "accession_number": artifact.get("accessionyear", "Unknown"),
            "url": artifact.get("url", "")
        }
        metadata_list.append(metadata)
    
    return pd.DataFrame(metadata_list)

def transform_artifact_media(artifacts):
    """Extract media/image data"""
    media_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        images = artifact.get("images", [])
        
        for image in images:
            media = {
                "artifact_id": artifact_id,
                "image_url": image.get("baseimageurl", ""),
                "media_type": "image"
            }
            media_list.append(media)
    
    return pd.DataFrame(media_list)

def transform_artifact_colors(artifacts):
    """Extract color data"""
    color_list = []
    
    for artifact in artifacts:
        artifact_id = artifact.get("id")
        colors = artifact.get("colors", [])
        
        for color in colors:
            color_data = {
                "artifact_id": artifact_id,
                "color_hex": color.get("hex", ""),
                "color_percent": color.get("percent", 0.0)
            }
            color_list.append(color_data)
    
    return pd.DataFrame(color_list)
```

### Load: Inserting into SQL Database

```python
import mysql.connector
from mysql.connector import Error

def get_db_connection():
    """Create database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv("DB_HOST"),
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD"),
            database=os.getenv("DB_NAME")
        )
        return connection
    except Error as e:
        print(f"Database connection error: {e}")
        return None

def load_metadata(df_metadata):
    """Batch insert artifact metadata"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmetadata 
        (id, title, culture, century, classification, department, dated, accession_number, url)
        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
        ON DUPLICATE KEY UPDATE
        title=VALUES(title), culture=VALUES(culture)
    """
    
    try:
        data_tuples = [tuple(row) for row in df_metadata.values]
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        return True
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()

def load_media(df_media):
    """Batch insert media data"""
    connection = get_db_connection()
    if not connection:
        return False
    
    cursor = connection.cursor()
    
    insert_query = """
        INSERT INTO artifactmedia (artifact_id, image_url, media_type)
        VALUES (%s, %s, %s)
    """
    
    try:
        data_tuples = [tuple(row) for row in df_media.values]
        cursor.executemany(insert_query, data_tuples)
        connection.commit()
        return True
    except Error as e:
        print(f"Insert error: {e}")
        connection.rollback()
        return False
    finally:
        cursor.close()
        connection.close()
```

## SQL Analytics Queries

### Example Analytical Queries

```python
# Query 1: Artifact count by culture
query_by_culture = """
    SELECT culture, COUNT(*) as artifact_count
    FROM artifactmetadata
    GROUP BY culture
    ORDER BY artifact_count DESC
    LIMIT 10
"""

# Query 2: Artifacts with media availability
query_media_availability = """
    SELECT 
        m.department,
        COUNT(DISTINCT m.id) as total_artifacts,
        COUNT(DISTINCT media.artifact_id) as artifacts_with_media
    FROM artifactmetadata m
    LEFT JOIN artifactmedia media ON m.id = media.artifact_id
    GROUP BY m.department
"""

# Query 3: Color distribution analysis
query_color_distribution = """
    SELECT 
        color_hex,
        COUNT(*) as usage_count,
        AVG(color_percent) as avg_percent
    FROM artifactcolors
    GROUP BY color_hex
    ORDER BY usage_count DESC
    LIMIT 15
"""

# Query 4: Century-wise artifact distribution
query_by_century = """
    SELECT 
        century,
        classification,
        COUNT(*) as count
    FROM artifactmetadata
    WHERE century IS NOT NULL AND century != 'Unknown'
    GROUP BY century, classification
    ORDER BY count DESC
"""

def execute_query(query):
    """Execute SQL query and return DataFrame"""
    connection = get_db_connection()
    if not connection:
        return None
    
    try:
        df = pd.read_sql(query, connection)
        return df
    except Error as e:
        print(f"Query error: {e}")
        return None
    finally:
        connection.close()
```

## Streamlit Dashboard Implementation

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for navigation
    page = st.sidebar.selectbox(
        "Select Page",
        ["Data Collection", "SQL Analytics", "Visualizations"]
    )
    
    if page == "Data Collection":
        show_data_collection_page()
    elif page == "SQL Analytics":
        show_analytics_page()
    elif page == "Visualizations":
        show_visualization_page()

def show_data_collection_page():
    st.header("📥 Data Collection from Harvard API")
    
    num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=20, value=5)
    
    if st.button("Fetch and Load Data"):
        api_key = os.getenv("HARVARD_API_KEY")
        
        with st.spinner("Fetching data..."):
            all_artifacts = []
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(api_key, page=page)
                all_artifacts.extend(data.get("records", []))
            
            # Transform
            df_metadata = transform_artifact_metadata(all_artifacts)
            df_media = transform_artifact_media(all_artifacts)
            df_colors = transform_artifact_colors(all_artifacts)
            
            # Load
            success_meta = load_metadata(df_metadata)
            success_media = load_media(df_media)
            
            if success_meta and success_media:
                st.success(f"✅ Successfully loaded {len(df_metadata)} artifacts!")
            else:
                st.error("❌ Error loading data")

def show_analytics_page():
    st.header("📊 SQL Analytics")
    
    queries = {
        "Artifacts by Culture": query_by_culture,
        "Media Availability by Department": query_media_availability,
        "Color Distribution": query_color_distribution,
        "Century-wise Distribution": query_by_century
    }
    
    selected_query = st.selectbox("Select Analysis", list(queries.keys()))
    
    if st.button("Run Query"):
        with st.spinner("Executing query..."):
            df_result = execute_query(queries[selected_query])
            
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
            else:
                st.warning("No data returned")

def show_visualization_page():
    st.header("📈 Interactive Visualizations")
    
    # Example: Culture distribution pie chart
    df_culture = execute_query(query_by_culture)
    
    if df_culture is not None:
        fig = px.pie(
            df_culture,
            names='culture',
            values='artifact_count',
            title='Artifact Distribution by Culture'
        )
        st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Complete ETL Workflow

```python
def run_complete_etl_pipeline(num_pages=5):
    """Complete ETL pipeline execution"""
    api_key = os.getenv("HARVARD_API_KEY")
    
    # EXTRACT
    print(f"Extracting {num_pages} pages of data...")
    all_artifacts = []
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            all_artifacts.extend(data.get("records", []))
            print(f"Page {page}: {len(data.get('records', []))} artifacts")
        except Exception as e:
            print(f"Error on page {page}: {e}")
            continue
    
    # TRANSFORM
    print("Transforming data...")
    df_metadata = transform_artifact_metadata(all_artifacts)
    df_media = transform_artifact_media(all_artifacts)
    df_colors = transform_artifact_colors(all_artifacts)
    
    print(f"Metadata records: {len(df_metadata)}")
    print(f"Media records: {len(df_media)}")
    print(f"Color records: {len(df_colors)}")
    
    # LOAD
    print("Loading into database...")
    success_meta = load_metadata(df_metadata)
    success_media = load_media(df_media)
    success_colors = load_colors(df_colors)  # Similar to load_media
    
    if all([success_meta, success_media, success_colors]):
        print("✅ ETL Pipeline completed successfully!")
        return True
    else:
        print("❌ ETL Pipeline failed")
        return False
```

## Configuration

Create a `.env` file:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_USER=your_database_user
DB_PASSWORD=your_database_password
DB_NAME=harvard_artifacts_db
DB_PORT=3306
```

Load environment variables:

```python
from dotenv import load_dotenv
load_dotenv()
```

## Troubleshooting

### API Rate Limiting

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with retry logic for rate limiting"""
    for attempt in range(max_retries):
        try:
            data = fetch_artifacts(api_key, page=page)
            return data
        except Exception as e:
            if "429" in str(e):  # Rate limit error
                wait_time = (attempt + 1) * 5
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    return None
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        connection = get_db_connection()
        if connection and connection.is_connected():
            print("✅ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE()")
            db_name = cursor.fetchone()
            print(f"Connected to: {db_name[0]}")
            cursor.close()
            connection.close()
            return True
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Handling Missing Data

```python
def safe_get(dictionary, key, default="Unknown"):
    """Safely extract values with fallback"""
    value = dictionary.get(key, default)
    return value if value else default

# Usage in transformation
metadata = {
    "id": artifact.get("id"),
    "title": safe_get(artifact, "title"),
    "culture": safe_get(artifact, "culture"),
    "century": safe_get(artifact, "century")
}
```

## Best Practices

1. **Always use environment variables** for sensitive data
2. **Implement pagination** for large datasets
3. **Use batch inserts** for better SQL performance
4. **Handle API rate limits** with exponential backoff
5. **Validate data** before database insertion
6. **Use transactions** for data integrity
7. **Create indexes** on frequently queried columns
8. **Cache results** for expensive queries in Streamlit

This skill provides comprehensive patterns for building production-grade data engineering pipelines with API integration, ETL processing, SQL analytics, and interactive visualization.
