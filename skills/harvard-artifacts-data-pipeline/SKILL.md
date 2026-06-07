---
name: harvard-artifacts-data-pipeline
description: Build ETL pipelines and analytics dashboards using Harvard Art Museums API with Streamlit, MySQL, and Plotly
triggers:
  - how do I build a data pipeline with the Harvard Art Museums API
  - show me how to use the Harvard artifacts analytics app
  - help me create an ETL pipeline for museum data
  - how to visualize Harvard Art Museums data with Streamlit
  - build a SQL analytics dashboard for art museum collections
  - integrate Harvard Art Museums API into a data engineering project
  - create interactive visualizations for museum artifact data
  - set up a complete data pipeline from API to dashboard
---

# Harvard Artifacts Data Pipeline

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides an end-to-end data engineering and analytics application that extracts artifact data from the Harvard Art Museums API, performs ETL operations, stores data in MySQL/TiDB Cloud, and visualizes insights through a Streamlit dashboard.

## What It Does

The Harvard Artifacts Collection Data Engineering Analytics App:
- Fetches artifact data from Harvard Art Museums API with pagination and rate limiting
- Transforms nested JSON into relational database tables
- Loads data into MySQL with proper foreign key relationships
- Executes 20+ analytical SQL queries for insights
- Visualizes results using interactive Plotly charts in Streamlit

**Architecture**: API → ETL → SQL → Analytics → Visualization

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt
```

Required packages:
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

Create a `.env` file or use Streamlit secrets:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# MySQL/TiDB Connection
DB_HOST=your_db_host
DB_PORT=3306
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

### Streamlit Secrets (`.streamlit/secrets.toml`)

```toml
[api]
harvard_api_key = "your_api_key_here"

[database]
host = "your_db_host"
port = 3306
user = "your_db_user"
password = "your_db_password"
database = "harvard_artifacts"
```

Get your free API key from: https://www.harvardartmuseums.org/collections/api

## Database Schema

The application creates three main tables:

```sql
-- Artifact metadata
CREATE TABLE artifactmetadata (
    artifact_id INT PRIMARY KEY,
    title VARCHAR(500),
    culture VARCHAR(200),
    century VARCHAR(100),
    classification VARCHAR(200),
    department VARCHAR(200),
    object_number VARCHAR(100),
    dated VARCHAR(200),
    medium VARCHAR(500),
    technique VARCHAR(500)
);

-- Artifact media/images
CREATE TABLE artifactmedia (
    media_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    image_url VARCHAR(1000),
    base_image_url VARCHAR(1000),
    caption TEXT,
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);

-- Artifact colors
CREATE TABLE artifactcolors (
    color_id INT AUTO_INCREMENT PRIMARY KEY,
    artifact_id INT,
    color_hex VARCHAR(10),
    color_percent FLOAT,
    color_spectrum VARCHAR(50),
    FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
);
```

## Key Components

### 1. API Integration

```python
import requests
import pandas as pd

def fetch_harvard_artifacts(api_key, num_records=100):
    """
    Fetch artifacts from Harvard Art Museums API with pagination
    """
    base_url = "https://api.harvardartmuseums.org/object"
    artifacts = []
    page = 1
    per_page = 100
    
    while len(artifacts) < num_records:
        params = {
            "apikey": api_key,
            "size": per_page,
            "page": page,
            "hasimage": 1  # Only artifacts with images
        }
        
        response = requests.get(base_url, params=params)
        
        if response.status_code == 200:
            data = response.json()
            records = data.get("records", [])
            
            if not records:
                break
                
            artifacts.extend(records)
            page += 1
        else:
            print(f"Error: {response.status_code}")
            break
    
    return artifacts[:num_records]
```

### 2. ETL Pipeline

```python
import mysql.connector
from mysql.connector import Error

def transform_artifacts(raw_data):
    """
    Transform nested JSON into relational dataframes
    """
    metadata_list = []
    media_list = []
    colors_list = []
    
    for artifact in raw_data:
        # Extract metadata
        metadata = {
            'artifact_id': artifact.get('id'),
            'title': artifact.get('title'),
            'culture': artifact.get('culture'),
            'century': artifact.get('century'),
            'classification': artifact.get('classification'),
            'department': artifact.get('department'),
            'object_number': artifact.get('objectnumber'),
            'dated': artifact.get('dated'),
            'medium': artifact.get('medium'),
            'technique': artifact.get('technique')
        }
        metadata_list.append(metadata)
        
        # Extract media/images
        for image in artifact.get('images', []):
            media = {
                'artifact_id': artifact.get('id'),
                'image_url': image.get('baseimageurl'),
                'base_image_url': image.get('iiifbaseuri'),
                'caption': image.get('caption')
            }
            media_list.append(media)
        
        # Extract colors
        for color in artifact.get('colors', []):
            color_data = {
                'artifact_id': artifact.get('id'),
                'color_hex': color.get('hex'),
                'color_percent': color.get('percent'),
                'color_spectrum': color.get('spectrum')
            }
            colors_list.append(color_data)
    
    return (
        pd.DataFrame(metadata_list),
        pd.DataFrame(media_list),
        pd.DataFrame(colors_list)
    )

def load_to_mysql(connection, df, table_name, batch_size=100):
    """
    Batch insert dataframe into MySQL table
    """
    cursor = connection.cursor()
    
    # Prepare column names and placeholders
    columns = ', '.join(df.columns)
    placeholders = ', '.join(['%s'] * len(df.columns))
    
    insert_query = f"""
        INSERT INTO {table_name} ({columns})
        VALUES ({placeholders})
        ON DUPLICATE KEY UPDATE {', '.join([f"{col}=VALUES({col})" for col in df.columns if col != 'artifact_id'])}
    """
    
    # Batch insert
    for i in range(0, len(df), batch_size):
        batch = df.iloc[i:i+batch_size]
        values = [tuple(row) for row in batch.values]
        cursor.executemany(insert_query, values)
        connection.commit()
    
    cursor.close()
```

### 3. Database Connection

```python
def create_db_connection(config):
    """
    Create MySQL database connection
    """
    try:
        connection = mysql.connector.connect(
            host=config['host'],
            port=config['port'],
            user=config['user'],
            password=config['password'],
            database=config['database']
        )
        
        if connection.is_connected():
            print("Successfully connected to database")
            return connection
    except Error as e:
        print(f"Error connecting to MySQL: {e}")
        return None

# Usage
db_config = {
    'host': os.getenv('DB_HOST'),
    'port': int(os.getenv('DB_PORT', 3306)),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

connection = create_db_connection(db_config)
```

### 4. Analytical Queries

```python
# Sample analytical queries
ANALYTICS_QUERIES = {
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
        SELECT color_spectrum, COUNT(*) as usage_count, 
               AVG(color_percent) as avg_percent
        FROM artifactcolors
        GROUP BY color_spectrum
        ORDER BY usage_count DESC
        LIMIT 10
    """,
    
    "Artifacts with Most Images": """
        SELECT m.artifact_id, m.title, COUNT(media.media_id) as image_count
        FROM artifactmetadata m
        JOIN artifactmedia media ON m.artifact_id = media.artifact_id
        GROUP BY m.artifact_id, m.title
        ORDER BY image_count DESC
        LIMIT 10
    """,
    
    "Department Distribution": """
        SELECT department, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE department IS NOT NULL
        GROUP BY department
        ORDER BY artifact_count DESC
    """
}

def execute_query(connection, query):
    """
    Execute SQL query and return results as DataFrame
    """
    cursor = connection.cursor()
    cursor.execute(query)
    results = cursor.fetchall()
    columns = [desc[0] for desc in cursor.description]
    cursor.close()
    
    return pd.DataFrame(results, columns=columns)
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    
    # Sidebar configuration
    st.sidebar.header("Configuration")
    
    api_key = st.sidebar.text_input("Harvard API Key", 
                                     value=st.secrets.get("api", {}).get("harvard_api_key", ""),
                                     type="password")
    
    # Data collection
    if st.sidebar.button("Fetch New Data"):
        with st.spinner("Fetching artifacts..."):
            artifacts = fetch_harvard_artifacts(api_key, num_records=500)
            metadata_df, media_df, colors_df = transform_artifacts(artifacts)
            
            # Load to database
            connection = create_db_connection(st.secrets["database"])
            load_to_mysql(connection, metadata_df, "artifactmetadata")
            load_to_mysql(connection, media_df, "artifactmedia")
            load_to_mysql(connection, colors_df, "artifactcolors")
            connection.close()
            
            st.success(f"Loaded {len(artifacts)} artifacts!")
    
    # Analytics section
    st.header("📊 Analytics")
    
    query_name = st.selectbox("Select Analysis", list(ANALYTICS_QUERIES.keys()))
    
    if st.button("Run Query"):
        connection = create_db_connection(st.secrets["database"])
        query = ANALYTICS_QUERIES[query_name]
        
        results_df = execute_query(connection, query)
        connection.close()
        
        # Display results
        st.subheader("Query Results")
        st.dataframe(results_df)
        
        # Visualization
        if len(results_df.columns) >= 2:
            fig = px.bar(results_df, 
                        x=results_df.columns[0], 
                        y=results_df.columns[1],
                        title=query_name)
            st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start the Streamlit dashboard
streamlit run app.py

# Access at http://localhost:8501
```

## Common Patterns

### Pattern 1: Full ETL Pipeline

```python
# Complete ETL workflow
api_key = os.getenv('HARVARD_API_KEY')
db_config = {
    'host': os.getenv('DB_HOST'),
    'user': os.getenv('DB_USER'),
    'password': os.getenv('DB_PASSWORD'),
    'database': os.getenv('DB_NAME')
}

# Extract
artifacts = fetch_harvard_artifacts(api_key, num_records=1000)

# Transform
metadata_df, media_df, colors_df = transform_artifacts(artifacts)

# Load
connection = create_db_connection(db_config)
load_to_mysql(connection, metadata_df, "artifactmetadata")
load_to_mysql(connection, media_df, "artifactmedia")
load_to_mysql(connection, colors_df, "artifactcolors")
connection.close()
```

### Pattern 2: Custom Analytics Query

```python
def run_custom_analysis(connection, culture_filter):
    """
    Run custom analysis filtered by culture
    """
    query = f"""
        SELECT m.title, m.century, COUNT(c.color_id) as color_variety
        FROM artifactmetadata m
        LEFT JOIN artifactcolors c ON m.artifact_id = c.artifact_id
        WHERE m.culture = %s
        GROUP BY m.artifact_id, m.title, m.century
        ORDER BY color_variety DESC
        LIMIT 20
    """
    
    cursor = connection.cursor()
    cursor.execute(query, (culture_filter,))
    results = cursor.fetchall()
    cursor.close()
    
    return pd.DataFrame(results, columns=['title', 'century', 'color_variety'])
```

### Pattern 3: Incremental Data Loading

```python
def get_latest_artifact_id(connection):
    """
    Get the latest artifact ID to avoid duplicates
    """
    cursor = connection.cursor()
    cursor.execute("SELECT MAX(artifact_id) FROM artifactmetadata")
    result = cursor.fetchone()
    cursor.close()
    return result[0] if result[0] else 0

def fetch_new_artifacts_only(api_key, last_id):
    """
    Fetch only artifacts newer than last_id
    """
    # Implementation depends on API capabilities
    pass
```

## Troubleshooting

**API Rate Limiting**: The Harvard API has rate limits. Add delays between requests:
```python
import time
time.sleep(1)  # Wait 1 second between API calls
```

**Database Connection Errors**: Ensure your MySQL server is running and credentials are correct:
```python
# Test connection
connection = mysql.connector.connect(
    host=os.getenv('DB_HOST'),
    user=os.getenv('DB_USER'),
    password=os.getenv('DB_PASSWORD')
)
print(connection.is_connected())
```

**Missing Data**: Handle None/null values in transformations:
```python
metadata = {
    'artifact_id': artifact.get('id'),
    'title': artifact.get('title', 'Unknown'),
    'culture': artifact.get('culture') or 'Unknown'
}
```

**Streamlit Secrets Not Loading**: Verify `.streamlit/secrets.toml` exists and is properly formatted.

**Large Dataset Performance**: Use batch processing and indexing:
```sql
CREATE INDEX idx_culture ON artifactmetadata(culture);
CREATE INDEX idx_century ON artifactmetadata(century);
```
