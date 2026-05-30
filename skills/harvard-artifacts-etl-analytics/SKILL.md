---
name: harvard-artifacts-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering pipeline for museum artifacts
  - set up Harvard artifacts analytics dashboard
  - build Streamlit app with Harvard Art Museums data
  - extract and transform Harvard museum collection data
  - create SQL analytics for art museum artifacts
  - visualize Harvard Art Museums API data
  - implement ETL with museum artifact metadata
---

# Harvard Artifacts ETL Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables you to build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection app provides:
- **API Integration**: Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extracts, transforms, and loads artifact metadata, media, and color data into SQL databases
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Dashboards**: Streamlit-based visualizations using Plotly
- **Database Design**: Relational schema with proper foreign key relationships

## Installation

### Prerequisites

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

### Environment Setup

Create a `.env` file in the project root:

```bash
HARVARD_API_KEY=your_api_key_here
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema

The application uses three main tables:

### artifactmetadata
```sql
CREATE TABLE artifactmetadata (
    id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(255),
    century VARCHAR(255),
    classification VARCHAR(255),
    department VARCHAR(255),
    division VARCHAR(255),
    dated VARCHAR(255),
    description TEXT,
    url VARCHAR(500)
);
```

### artifactmedia
```sql
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(500),
    rank INT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

### artifactcolors
```sql
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color VARCHAR(50),
    percentage DECIMAL(5,2),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd
import os
from dotenv import load_dotenv

load_dotenv()

def fetch_artifacts(api_key, page=1, size=100):
    """
    Fetch artifacts from Harvard Art Museums API
    
    Args:
        api_key: Harvard API key
        page: Page number for pagination
        size: Number of records per page (max 100)
    
    Returns:
        JSON response with artifact data
    """
    url = f"https://api.harvardartmuseums.org/object"
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

# Usage
api_key = os.getenv('HARVARD_API_KEY')
data = fetch_artifacts(api_key, page=1, size=50)
artifacts = data.get('records', [])
```

### 2. ETL Pipeline

```python
import mysql.connector
from mysql.connector import Error

def connect_to_database():
    """Establish database connection"""
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

def extract_metadata(artifact):
    """Extract metadata from artifact JSON"""
    return {
        'id': artifact.get('id'),
        'title': artifact.get('title', 'Unknown')[:500],
        'culture': artifact.get('culture', 'Unknown')[:255],
        'century': artifact.get('century', 'Unknown')[:255],
        'classification': artifact.get('classification', 'Unknown')[:255],
        'department': artifact.get('department', 'Unknown')[:255],
        'division': artifact.get('division', 'Unknown')[:255],
        'dated': artifact.get('dated', 'Unknown')[:255],
        'description': artifact.get('description', '')[:5000],
        'url': artifact.get('url', '')[:500]
    }

def extract_media(artifact):
    """Extract media/image data from artifact"""
    media_list = []
    images = artifact.get('images', [])
    
    for img in images:
        media_list.append({
            'artifact_id': artifact.get('id'),
            'image_url': img.get('baseimageurl', '')[:500],
            'rank': img.get('rank', 0)
        })
    
    return media_list

def extract_colors(artifact):
    """Extract color data from artifact"""
    color_list = []
    colors = artifact.get('colors', [])
    
    for color in colors:
        color_list.append({
            'artifact_id': artifact.get('id'),
            'color': color.get('color', 'Unknown')[:50],
            'percentage': color.get('percent', 0.0)
        })
    
    return color_list

def load_metadata(connection, metadata_list):
    """Load metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, division, dated, description, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE
    title=VALUES(title), culture=VALUES(culture), century=VALUES(century)
    """
    
    values = [
        (m['id'], m['title'], m['culture'], m['century'], m['classification'],
         m['department'], m['division'], m['dated'], m['description'], m['url'])
        for m in metadata_list
    ]
    
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()

def load_media(connection, media_list):
    """Load media data into database"""
    if not media_list:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, rank)
    VALUES (%s, %s, %s)
    """
    
    values = [(m['artifact_id'], m['image_url'], m['rank']) for m in media_list]
    
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()

def load_colors(connection, color_list):
    """Load color data into database"""
    if not color_list:
        return
    
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactcolors (artifact_id, color, percentage)
    VALUES (%s, %s, %s)
    """
    
    values = [(c['artifact_id'], c['color'], c['percentage']) for c in color_list]
    
    cursor.executemany(insert_query, values)
    connection.commit()
    cursor.close()

# Complete ETL pipeline
def run_etl_pipeline(num_pages=5):
    """Run complete ETL pipeline"""
    api_key = os.getenv('HARVARD_API_KEY')
    connection = connect_to_database()
    
    if not connection:
        print("Failed to connect to database")
        return
    
    for page in range(1, num_pages + 1):
        print(f"Processing page {page}...")
        
        # Extract
        data = fetch_artifacts(api_key, page=page, size=100)
        artifacts = data.get('records', [])
        
        # Transform
        metadata_list = [extract_metadata(a) for a in artifacts]
        media_list = [item for a in artifacts for item in extract_media(a)]
        color_list = [item for a in artifacts for item in extract_colors(a)]
        
        # Load
        load_metadata(connection, metadata_list)
        load_media(connection, media_list)
        load_colors(connection, color_list)
        
        print(f"Loaded {len(metadata_list)} artifacts from page {page}")
    
    connection.close()
    print("ETL pipeline completed!")
```

### 3. SQL Analytics Queries

```python
def execute_query(connection, query):
    """Execute SQL query and return results as DataFrame"""
    cursor = connection.cursor()
    cursor.execute(query)
    
    columns = [desc[0] for desc in cursor.description]
    results = cursor.fetchall()
    
    df = pd.DataFrame(results, columns=columns)
    cursor.close()
    
    return df

# Sample analytical queries
ANALYTICAL_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 10
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL AND century != 'Unknown'
        GROUP BY century
        ORDER BY count DESC
        LIMIT 15
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as total_artifacts
        FROM artifactmetadata
        GROUP BY department
        ORDER BY total_artifacts DESC
    """,
    
    "Top Colors Used": """
        SELECT color, COUNT(*) as usage_count, AVG(percentage) as avg_percentage
        FROM artifactcolors
        GROUP BY color
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.title, m.culture, COUNT(media.image_url) as image_count
        FROM artifactmetadata m
        LEFT JOIN artifactmedia media ON m.id = media.artifact_id
        GROUP BY m.id, m.title, m.culture
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Classification Breakdown": """
        SELECT classification, COUNT(*) as count
        FROM artifactmetadata
        GROUP BY classification
        ORDER BY count DESC
        LIMIT 10
    """
}

# Usage
connection = connect_to_database()
df = execute_query(connection, ANALYTICAL_QUERIES["Artifacts by Culture"])
print(df)
```

### 4. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("End-to-end ETL pipeline and analytics for Harvard Art Museums collection")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    connection = connect_to_database()
    
    if not connection:
        st.error("Failed to connect to database. Check your credentials.")
        return
    
    # ETL Section
    st.header("📥 Data Collection (ETL)")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Number of pages to fetch", min_value=1, max_value=50, value=5)
    
    with col2:
        if st.button("Run ETL Pipeline"):
            with st.spinner("Running ETL pipeline..."):
                run_etl_pipeline(num_pages)
                st.success(f"Successfully loaded {num_pages * 100} artifacts!")
    
    # Analytics Section
    st.header("📊 SQL Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICAL_QUERIES.keys()))
    
    if st.button("Execute Query"):
        with st.spinner("Executing query..."):
            query = ANALYTICAL_QUERIES[query_name]
            df = execute_query(connection, query)
            
            st.subheader("Query Results")
            st.dataframe(df)
            
            # Visualization
            if len(df) > 0:
                st.subheader("Visualization")
                
                # Auto-generate chart based on data
                if len(df.columns) >= 2:
                    fig = px.bar(
                        df,
                        x=df.columns[0],
                        y=df.columns[1],
                        title=query_name
                    )
                    st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit app
streamlit run app.py

# The app will open at http://localhost:8501
```

## Common Patterns

### Batch Processing with Pagination

```python
def batch_collect_artifacts(total_pages=10, batch_size=5):
    """Collect artifacts in batches to avoid rate limits"""
    import time
    
    for batch_start in range(1, total_pages + 1, batch_size):
        batch_end = min(batch_start + batch_size, total_pages + 1)
        
        print(f"Processing batch: pages {batch_start} to {batch_end - 1}")
        run_etl_pipeline_for_pages(batch_start, batch_end - 1)
        
        # Rate limiting - wait between batches
        time.sleep(2)
```

### Error Handling in ETL

```python
def safe_etl_pipeline(num_pages=5):
    """ETL pipeline with error handling"""
    api_key = os.getenv('HARVARD_API_KEY')
    connection = connect_to_database()
    
    failed_pages = []
    
    for page in range(1, num_pages + 1):
        try:
            data = fetch_artifacts(api_key, page=page, size=100)
            artifacts = data.get('records', [])
            
            # Transform and load
            metadata_list = [extract_metadata(a) for a in artifacts]
            load_metadata(connection, metadata_list)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            failed_pages.append(page)
            continue
    
    connection.close()
    
    if failed_pages:
        print(f"Failed pages: {failed_pages}")
    
    return failed_pages
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_with_retry(api_key, page, max_retries=3):
    """Fetch with exponential backoff"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(api_key, page)
        except Exception as e:
            if "429" in str(e):  # Rate limit
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_database_connection():
    """Test and debug database connection"""
    try:
        connection = mysql.connector.connect(
            host=os.getenv('DB_HOST'),
            port=int(os.getenv('DB_PORT', 3306)),
            user=os.getenv('DB_USER'),
            password=os.getenv('DB_PASSWORD'),
            database=os.getenv('DB_NAME')
        )
        
        if connection.is_connected():
            print("✅ Database connection successful")
            cursor = connection.cursor()
            cursor.execute("SELECT DATABASE();")
            db_name = cursor.fetchone()
            print(f"Connected to database: {db_name[0]}")
            cursor.close()
            connection.close()
            return True
            
    except Error as e:
        print(f"❌ Connection failed: {e}")
        return False
```

### Missing API Key

```python
def validate_env_vars():
    """Validate required environment variables"""
    required_vars = ['HARVARD_API_KEY', 'DB_HOST', 'DB_USER', 'DB_PASSWORD', 'DB_NAME']
    
    missing = []
    for var in required_vars:
        if not os.getenv(var):
            missing.append(var)
    
    if missing:
        raise ValueError(f"Missing environment variables: {', '.join(missing)}")
    
    print("✅ All environment variables configured")
```

## Best Practices

1. **Use environment variables** for all credentials
2. **Implement pagination** when fetching large datasets
3. **Add rate limiting** to respect API quotas
4. **Use batch inserts** for better database performance
5. **Validate data** before loading into database
6. **Log errors** for debugging ETL failures
7. **Index foreign keys** for faster joins
8. **Cache query results** in Streamlit for better UX

This skill provides everything needed to build production-grade ETL pipelines and analytics dashboards using the Harvard Art Museums API.
