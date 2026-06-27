---
name: harvard-art-museums-etl-analytics
description: Build end-to-end ETL pipelines and analytics dashboards using the Harvard Art Museums API with Python, SQL, and Streamlit
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create analytics dashboard for museum artifacts
  - integrate Harvard Art Museums API into data pipeline
  - visualize museum collection data with Streamlit
  - query and analyze Harvard artifacts database
  - set up museum data engineering workflow
  - extract and transform Harvard Art Museums metadata
  - build SQL analytics for art collection data
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI coding agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit visualizations for museum artifact data.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract, transform, and load nested JSON data into relational SQL tables
- **Database Design**: Store artifacts metadata, media, and color information in MySQL/TiDB Cloud
- **SQL Analytics**: Execute predefined analytical queries on museum data
- **Interactive Visualization**: Display query results with Plotly charts in Streamlit dashboards

The architecture follows: **API → ETL → SQL → Analytics → Visualization**

## Installation

### Prerequisites

```bash
# Python 3.8+
python --version

# Install dependencies
pip install streamlit pandas requests mysql-connector-python plotly python-dotenv
```

### Environment Setup

Create a `.env` file for configuration:

```bash
# Harvard Art Museums API
HARVARD_API_KEY=your_api_key_here

# Database Configuration
DB_HOST=your_database_host
DB_PORT=4000
DB_USER=your_db_user
DB_PASSWORD=your_db_password
DB_NAME=harvard_artifacts
```

Get your Harvard Art Museums API key from: https://www.harvardartmuseums.org/collections/api

## Project Structure

```
harvard-artifacts-app/
├── app.py                  # Main Streamlit application
├── etl_pipeline.py         # ETL logic
├── database.py             # Database connection and operations
├── queries.py              # SQL analytical queries
├── config.py               # Configuration management
├── requirements.txt        # Dependencies
└── .env                    # Environment variables
```

## Core Components

### 1. API Integration

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

class HarvardAPIClient:
    def __init__(self):
        self.api_key = os.getenv('HARVARD_API_KEY')
        self.base_url = 'https://api.harvardartmuseums.org/object'
    
    def fetch_artifacts(self, page=1, size=100):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size,
            'hasimage': 1  # Only artifacts with images
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        return response.json()
    
    def fetch_batch(self, num_pages=5, size=100):
        """Fetch multiple pages of artifacts"""
        all_records = []
        
        for page in range(1, num_pages + 1):
            data = self.fetch_artifacts(page=page, size=size)
            all_records.extend(data.get('records', []))
            
            # Rate limiting
            import time
            time.sleep(0.5)
        
        return all_records
```

### 2. Database Setup

```python
import mysql.connector
import os
from dotenv import load_dotenv

load_dotenv()

class DatabaseManager:
    def __init__(self):
        self.config = {
            'host': os.getenv('DB_HOST'),
            'port': int(os.getenv('DB_PORT', 3306)),
            'user': os.getenv('DB_USER'),
            'password': os.getenv('DB_PASSWORD'),
            'database': os.getenv('DB_NAME')
        }
    
    def get_connection(self):
        """Create database connection"""
        return mysql.connector.connect(**self.config)
    
    def create_tables(self):
        """Create schema for artifacts data"""
        conn = self.get_connection()
        cursor = conn.cursor()
        
        # Artifacts Metadata Table
        cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmetadata (
            id INT PRIMARY KEY,
            title VARCHAR(500),
            culture VARCHAR(255),
            classification VARCHAR(255),
            department VARCHAR(255),
            division VARCHAR(255),
            dated VARCHAR(255),
            century VARCHAR(255),
            provenance TEXT,
            creditline TEXT,
            description TEXT,
            url VARCHAR(500),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
        """)
        
        # Artifacts Media Table
        cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactmedia (
            media_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            baseimageurl VARCHAR(500),
            iiifbaseuri VARCHAR(500),
            renditionnumber VARCHAR(50),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
        """)
        
        # Artifacts Colors Table
        cursor.execute("""
        CREATE TABLE IF NOT EXISTS artifactcolors (
            color_id INT AUTO_INCREMENT PRIMARY KEY,
            artifact_id INT,
            color VARCHAR(50),
            spectrum VARCHAR(50),
            hue VARCHAR(50),
            percent DECIMAL(5,2),
            FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(id)
        )
        """)
        
        conn.commit()
        cursor.close()
        conn.close()
```

### 3. ETL Pipeline

```python
import pandas as pd

class ETLPipeline:
    def __init__(self, db_manager):
        self.db = db_manager
    
    def extract_metadata(self, records):
        """Extract artifact metadata"""
        metadata = []
        
        for record in records:
            metadata.append({
                'id': record.get('id'),
                'title': record.get('title'),
                'culture': record.get('culture'),
                'classification': record.get('classification'),
                'department': record.get('department'),
                'division': record.get('division'),
                'dated': record.get('dated'),
                'century': record.get('century'),
                'provenance': record.get('provenance'),
                'creditline': record.get('creditline'),
                'description': record.get('description'),
                'url': record.get('url')
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self, records):
        """Extract media information"""
        media_list = []
        
        for record in records:
            artifact_id = record.get('id')
            images = record.get('images', [])
            
            for img in images:
                media_list.append({
                    'artifact_id': artifact_id,
                    'baseimageurl': img.get('baseimageurl'),
                    'iiifbaseuri': img.get('iiifbaseuri'),
                    'renditionnumber': img.get('renditionnumber')
                })
        
        return pd.DataFrame(media_list)
    
    def extract_colors(self, records):
        """Extract color information"""
        colors_list = []
        
        for record in records:
            artifact_id = record.get('id')
            colors = record.get('colors', [])
            
            for color in colors:
                colors_list.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'percent': color.get('percent')
                })
        
        return pd.DataFrame(colors_list)
    
    def load_to_database(self, metadata_df, media_df, colors_df):
        """Load transformed data into SQL database"""
        conn = self.db.get_connection()
        cursor = conn.cursor()
        
        # Insert metadata
        for _, row in metadata_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmetadata 
                (id, title, culture, classification, department, division, 
                 dated, century, provenance, creditline, description, url)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE title=VALUES(title)
            """, tuple(row))
        
        # Insert media
        for _, row in media_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactmedia 
                (artifact_id, baseimageurl, iiifbaseuri, renditionnumber)
                VALUES (%s, %s, %s, %s)
            """, tuple(row))
        
        # Insert colors
        for _, row in colors_df.iterrows():
            cursor.execute("""
                INSERT INTO artifactcolors 
                (artifact_id, color, spectrum, hue, percent)
                VALUES (%s, %s, %s, %s, %s)
            """, tuple(row))
        
        conn.commit()
        cursor.close()
        conn.close()
```

### 4. SQL Analytics Queries

```python
class AnalyticsQueries:
    """Predefined analytical queries"""
    
    @staticmethod
    def get_artifacts_by_culture():
        return """
        SELECT culture, COUNT(*) as artifact_count
        FROM artifactmetadata
        WHERE culture IS NOT NULL
        GROUP BY culture
        ORDER BY artifact_count DESC
        LIMIT 20
        """
    
    @staticmethod
    def get_artifacts_by_century():
        return """
        SELECT century, COUNT(*) as count
        FROM artifactmetadata
        WHERE century IS NOT NULL
        GROUP BY century
        ORDER BY count DESC
        """
    
    @staticmethod
    def get_color_distribution():
        return """
        SELECT color, COUNT(*) as frequency, AVG(percent) as avg_percent
        FROM artifactcolors
        WHERE color IS NOT NULL
        GROUP BY color
        ORDER BY frequency DESC
        LIMIT 15
        """
    
    @staticmethod
    def get_department_classification():
        return """
        SELECT department, classification, COUNT(*) as count
        FROM artifactmetadata
        WHERE department IS NOT NULL AND classification IS NOT NULL
        GROUP BY department, classification
        ORDER BY count DESC
        LIMIT 20
        """
    
    @staticmethod
    def get_media_coverage():
        return """
        SELECT 
            COUNT(DISTINCT m.artifact_id) as with_media,
            (SELECT COUNT(*) FROM artifactmetadata) as total_artifacts,
            ROUND(COUNT(DISTINCT m.artifact_id) * 100.0 / 
                  (SELECT COUNT(*) FROM artifactmetadata), 2) as coverage_percent
        FROM artifactmedia m
        """

def execute_query(db_manager, query):
    """Execute SQL query and return DataFrame"""
    conn = db_manager.get_connection()
    df = pd.read_sql(query, conn)
    conn.close()
    return df
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def main():
    st.title("🏛️ Harvard Art Museums Analytics Dashboard")
    st.markdown("Real-time analytics on museum artifacts collection")
    
    # Initialize components
    db = DatabaseManager()
    api_client = HarvardAPIClient()
    etl = ETLPipeline(db)
    
    # Sidebar - Data Collection
    st.sidebar.header("📥 Data Collection")
    num_pages = st.sidebar.slider("Pages to fetch", 1, 10, 5)
    
    if st.sidebar.button("Fetch & Load Data"):
        with st.spinner("Fetching data from API..."):
            records = api_client.fetch_batch(num_pages=num_pages)
            st.success(f"Fetched {len(records)} artifacts")
        
        with st.spinner("Running ETL pipeline..."):
            metadata_df = etl.extract_metadata(records)
            media_df = etl.extract_media(records)
            colors_df = etl.extract_colors(records)
            
            etl.load_to_database(metadata_df, media_df, colors_df)
            st.success("Data loaded successfully!")
    
    # Analytics Section
    st.header("📊 Analytics")
    
    analysis_options = {
        "Artifacts by Culture": AnalyticsQueries.get_artifacts_by_culture(),
        "Artifacts by Century": AnalyticsQueries.get_artifacts_by_century(),
        "Color Distribution": AnalyticsQueries.get_color_distribution(),
        "Department Classification": AnalyticsQueries.get_department_classification(),
        "Media Coverage": AnalyticsQueries.get_media_coverage()
    }
    
    selected_analysis = st.selectbox("Select Analysis", list(analysis_options.keys()))
    
    if st.button("Run Analysis"):
        query = analysis_options[selected_analysis]
        
        with st.spinner("Executing query..."):
            results_df = execute_query(db, query)
            
            st.subheader("Results")
            st.dataframe(results_df)
            
            # Visualization
            if len(results_df) > 0 and len(results_df.columns) >= 2:
                fig = px.bar(
                    results_df,
                    x=results_df.columns[0],
                    y=results_df.columns[1],
                    title=selected_analysis
                )
                st.plotly_chart(fig, use_container_width=True)

if __name__ == "__main__":
    main()
```

## Running the Application

```bash
# Start Streamlit app
streamlit run app.py

# Access at http://localhost:8501
```

## Common Workflows

### 1. Initial Setup

```python
# Initialize database
db = DatabaseManager()
db.create_tables()

# Verify connection
conn = db.get_connection()
print("Database connected successfully")
conn.close()
```

### 2. One-time Data Load

```python
# Fetch large dataset
api_client = HarvardAPIClient()
records = api_client.fetch_batch(num_pages=20, size=100)

# Run ETL
etl = ETLPipeline(db)
metadata_df = etl.extract_metadata(records)
media_df = etl.extract_media(records)
colors_df = etl.extract_colors(records)

# Load to database
etl.load_to_database(metadata_df, media_df, colors_df)
```

### 3. Custom Analytics Query

```python
# Define custom query
custom_query = """
SELECT 
    classification,
    COUNT(*) as total,
    COUNT(DISTINCT culture) as unique_cultures
FROM artifactmetadata
WHERE classification IS NOT NULL
GROUP BY classification
ORDER BY total DESC
LIMIT 10
"""

# Execute and visualize
results = execute_query(db, custom_query)
print(results)
```

## Troubleshooting

### API Rate Limiting

If you encounter 429 errors:

```python
import time

def fetch_with_retry(api_client, page, max_retries=3):
    for attempt in range(max_retries):
        try:
            return api_client.fetch_artifacts(page=page)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Database Connection Issues

```python
# Test connection with error handling
try:
    conn = mysql.connector.connect(
        host=os.getenv('DB_HOST'),
        user=os.getenv('DB_USER'),
        password=os.getenv('DB_PASSWORD')
    )
    print("✓ Database connection successful")
    conn.close()
except mysql.connector.Error as err:
    print(f"✗ Error: {err}")
```

### Handling Missing Data

```python
# Safe extraction with defaults
def safe_extract(record, key, default=None):
    return record.get(key, default)

# Apply in ETL
metadata.append({
    'id': safe_extract(record, 'id'),
    'title': safe_extract(record, 'title', 'Unknown'),
    'culture': safe_extract(record, 'culture', 'Not specified')
})
```

## Best Practices

1. **Environment Variables**: Always use `.env` for credentials
2. **Batch Processing**: Use pagination for large datasets
3. **Error Handling**: Wrap API calls in try-except blocks
4. **Data Validation**: Check for NULL values before INSERT
5. **Indexing**: Add indexes on frequently queried columns
6. **Connection Pooling**: Reuse database connections when possible

This skill equips AI agents to assist developers in building production-ready data engineering pipelines for museum and cultural heritage data analytics.
