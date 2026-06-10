---
name: harvard-artifacts-etl-analytics
description: Build ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - how do I build an ETL pipeline with Harvard Art Museums API
  - create a data engineering project with Harvard artifacts
  - set up analytics dashboard for museum artifacts
  - extract and transform Harvard Art Museums data
  - build SQL analytics for art collections
  - implement artifact data visualization with Streamlit
  - create museum data pipeline with Python
  - analyze Harvard art collection with SQL queries
---

# Harvard Artifacts ETL & Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics solution for the Harvard Art Museums API. It demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive visualization using Streamlit.

## What It Does

- **Extracts** artifact data from Harvard Art Museums API with pagination and rate limiting
- **Transforms** nested JSON into normalized relational tables
- **Loads** structured data into MySQL/TiDB databases with proper foreign keys
- **Analyzes** artifacts using 20+ predefined SQL queries
- **Visualizes** results through interactive Plotly charts in a Streamlit dashboard

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

### Getting Harvard API Key

1. Visit [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api)
2. Request an API key (free for non-commercial use)
3. Add to your `.env` file

## Database Schema

The project uses three main tables:

```sql
-- Artifact Metadata
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
    medium VARCHAR(500),
    dimensions VARCHAR(500),
    creditline TEXT,
    accession_number VARCHAR(100),
    url VARCHAR(500)
);

-- Artifact Media
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);

-- Artifact Colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
);
```

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
        raise Exception(f"API Error: {response.status_code}")

# Example usage
data = fetch_artifacts(page=1, size=50)
print(f"Total records: {data['info']['totalrecords']}")
print(f"Fetched: {len(data['records'])} artifacts")
```

### 2. ETL Pipeline

```python
import pandas as pd
import mysql.connector
from mysql.connector import Error

def extract_artifact_metadata(records):
    """Extract and transform artifact metadata"""
    artifacts = []
    
    for record in records:
        artifact = {
            'id': record.get('id'),
            'title': record.get('title', 'Untitled'),
            'culture': record.get('culture'),
            'century': record.get('century'),
            'classification': record.get('classification'),
            'department': record.get('department'),
            'dated': record.get('dated'),
            'period': record.get('period'),
            'technique': record.get('technique'),
            'medium': record.get('medium'),
            'dimensions': record.get('dimensions'),
            'creditline': record.get('creditline'),
            'accession_number': record.get('accessionyear'),
            'url': record.get('url')
        }
        artifacts.append(artifact)
    
    return pd.DataFrame(artifacts)

def extract_media_data(records):
    """Extract media/image data from artifacts"""
    media_list = []
    
    for record in records:
        artifact_id = record.get('id')
        images = record.get('images', [])
        
        for img in images:
            media_list.append({
                'artifact_id': artifact_id,
                'image_url': img.get('baseimageurl'),
                'caption': img.get('caption')
            })
    
    return pd.DataFrame(media_list)

def extract_color_data(records):
    """Extract color palette data"""
    colors = []
    
    for record in records:
        artifact_id = record.get('id')
        color_data = record.get('colors', [])
        
        for color in color_data:
            colors.append({
                'artifact_id': artifact_id,
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent')
            })
    
    return pd.DataFrame(colors)
```

### 3. Database Loading

```python
def get_db_connection():
    """Create database connection"""
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

def load_artifacts(df_artifacts, connection):
    """Load artifact metadata into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmetadata 
    (id, title, culture, century, classification, department, 
     dated, period, technique, medium, dimensions, creditline, 
     accession_number, url)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
    ON DUPLICATE KEY UPDATE 
    title=VALUES(title), culture=VALUES(culture)
    """
    
    # Batch insert
    data_tuples = [tuple(row) for row in df_artifacts.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} artifacts")
    cursor.close()

def load_media(df_media, connection):
    """Load media data into database"""
    cursor = connection.cursor()
    
    insert_query = """
    INSERT INTO artifactmedia (artifact_id, image_url, caption)
    VALUES (%s, %s, %s)
    """
    
    data_tuples = [tuple(row) for row in df_media.values]
    cursor.executemany(insert_query, data_tuples)
    connection.commit()
    
    print(f"Inserted {cursor.rowcount} media records")
    cursor.close()
```

### 4. SQL Analytics Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
    "Artifacts by Culture": """
        SELECT culture, COUNT(*) as count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY count DESC
        LIMIT 20
    """,
    
    "Artifacts by Century": """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        GROUP BY department
        ORDER BY artifact_count DESC
    """,
    
    "Most Common Colors": """
        SELECT color_hex, COUNT(*) as frequency,
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_hex
        ORDER BY frequency DESC
        LIMIT 15
    """,
    
    "Artifacts with Most Images": """
        SELECT a.id, a.title, a.culture, COUNT(m.media_id) as image_count
        FROM artifactmetadata a
        JOIN artifactmedia m ON a.id = m.artifact_id
        GROUP BY a.id, a.title, a.culture
        ORDER BY image_count DESC
        LIMIT 20
    """,
    
    "Classification Analysis": """
        SELECT classification, COUNT(*) as count,
               COUNT(DISTINCT department) as dept_count
        FROM artifactmetadata
        WHERE classification IS NOT NULL
        GROUP BY classification
        ORDER BY count DESC
    """
}

def execute_analytics_query(query_name, connection):
    """Execute analytical query and return results"""
    cursor = connection.cursor(dictionary=True)
    query = ANALYTICS_QUERIES.get(query_name)
    
    cursor.execute(query)
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar for configuration
    st.sidebar.header("Configuration")
    
    # Database connection
    connection = get_db_connection()
    
    if connection is None:
        st.error("Failed to connect to database")
        return
    
    # ETL Section
    st.header("📥 Data Collection")
    
    col1, col2 = st.columns(2)
    
    with col1:
        num_pages = st.number_input("Pages to fetch", min_value=1, max_value=50, value=5)
    
    with col2:
        page_size = st.selectbox("Records per page", [50, 100])
    
    if st.button("Run ETL Pipeline"):
        with st.spinner("Fetching data from API..."):
            all_records = []
            
            for page in range(1, num_pages + 1):
                data = fetch_artifacts(page=page, size=page_size)
                all_records.extend(data['records'])
                st.progress(page / num_pages)
            
            st.success(f"Fetched {len(all_records)} artifacts")
        
        with st.spinner("Transforming and loading data..."):
            # Extract
            df_artifacts = extract_artifact_metadata(all_records)
            df_media = extract_media_data(all_records)
            df_colors = extract_color_data(all_records)
            
            # Load
            load_artifacts(df_artifacts, connection)
            load_media(df_media, connection)
            
            st.success("ETL pipeline completed successfully!")
    
    # Analytics Section
    st.header("📊 Analytics Dashboard")
    
    query_choice = st.selectbox(
        "Select Analysis",
        list(ANALYTICS_QUERIES.keys())
    )
    
    if st.button("Run Analysis"):
        df_results = execute_analytics_query(query_choice, connection)
        
        st.subheader("Results")
        st.dataframe(df_results, use_container_width=True)
        
        # Auto-generate visualization
        if len(df_results.columns) >= 2:
            fig = px.bar(
                df_results.head(20),
                x=df_results.columns[0],
                y=df_results.columns[1],
                title=query_choice
            )
            st.plotly_chart(fig, use_container_width=True)
    
    connection.close()

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# The app will open in your browser at http://localhost:8501
```

## Common Patterns

### Full ETL Workflow

```python
def full_etl_pipeline(num_pages=10, page_size=100):
    """Complete ETL pipeline execution"""
    connection = get_db_connection()
    all_records = []
    
    # Extract
    print("Starting data extraction...")
    for page in range(1, num_pages + 1):
        print(f"Fetching page {page}/{num_pages}")
        data = fetch_artifacts(page=page, size=page_size)
        all_records.extend(data['records'])
    
    print(f"Extracted {len(all_records)} total records")
    
    # Transform
    print("Transforming data...")
    df_artifacts = extract_artifact_metadata(all_records)
    df_media = extract_media_data(all_records)
    df_colors = extract_color_data(all_records)
    
    # Load
    print("Loading data to database...")
    load_artifacts(df_artifacts, connection)
    load_media(df_media, connection)
    
    if not df_colors.empty:
        load_colors(df_colors, connection)
    
    connection.close()
    print("ETL pipeline completed successfully!")

# Execute
full_etl_pipeline(num_pages=5, page_size=50)
```

### Custom Query Execution

```python
def run_custom_query(sql_query, connection):
    """Execute custom SQL query and visualize"""
    cursor = connection.cursor(dictionary=True)
    cursor.execute(sql_query)
    results = cursor.fetchall()
    cursor.close()
    
    df = pd.DataFrame(results)
    return df

# Example
custom_sql = """
SELECT a.century, a.classification, COUNT(*) as count
FROM artifactmetadata a
WHERE a.century IS NOT NULL
GROUP BY a.century, a.classification
ORDER BY count DESC
LIMIT 30
"""

connection = get_db_connection()
results = run_custom_query(custom_sql, connection)
print(results)
```

## Troubleshooting

### API Rate Limiting

If you encounter rate limit errors:

```python
import time

def fetch_artifacts_with_retry(page=1, size=100, max_retries=3):
    """Fetch with retry logic"""
    for attempt in range(max_retries):
        try:
            return fetch_artifacts(page, size)
        except Exception as e:
            if "429" in str(e):  # Rate limit
                wait_time = (attempt + 1) * 5
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
def test_db_connection():
    """Test database connectivity"""
    try:
        conn = get_db_connection()
        if conn and conn.is_connected():
            print("✅ Database connection successful")
            cursor = conn.cursor()
            cursor.execute("SELECT DATABASE()")
            db_name = cursor.fetchone()[0]
            print(f"Connected to database: {db_name}")
            cursor.close()
            conn.close()
            return True
        else:
            print("❌ Failed to connect to database")
            return False
    except Error as e:
        print(f"❌ Connection error: {e}")
        return False

# Run test
test_db_connection()
```

### Handling Missing Data

```python
def safe_extract_metadata(record):
    """Extract with null-safe handling"""
    return {
        'id': record.get('id'),
        'title': record.get('title') or 'Untitled',
        'culture': record.get('culture') or 'Unknown',
        'century': record.get('century') or 'Unknown',
        'classification': record.get('classification') or 'Unclassified',
        'department': record.get('department') or 'Unknown',
        # Add default values for all fields
    }
```

## Additional Resources

- [Harvard Art Museums API Documentation](https://github.com/harvardartmuseums/api-docs)
- [Streamlit Documentation](https://docs.streamlit.io)
- [Plotly Documentation](https://plotly.com/python/)
- [MySQL Python Connector](https://dev.mysql.com/doc/connector-python/en/)
