---
name: harvard-art-museums-etl-analytics
description: End-to-end data engineering pipeline for Harvard Art Museums API with ETL, SQL analytics, and Streamlit visualization
triggers:
  - build an ETL pipeline for Harvard Art Museums data
  - create a data analytics app with museum artifacts
  - set up Harvard Art Museums API integration
  - analyze art collection data with SQL queries
  - build a Streamlit dashboard for museum data
  - implement artifact data warehouse pipeline
  - extract and transform museum API data
  - create interactive art analytics visualization
---

# Harvard Art Museums ETL Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers build end-to-end data engineering and analytics applications using the Harvard Art Museums API. The project demonstrates real-world ETL pipelines, SQL database design, analytical queries, and interactive Streamlit dashboards.

## What This Project Does

The Harvard Artifacts Collection Data Engineering Analytics App provides:

- **API Integration**: Fetch artifact data from Harvard Art Museums API with pagination and rate limiting
- **ETL Pipeline**: Extract nested JSON, transform into relational format, load into SQL databases
- **Database Design**: Structured schema with `artifactmetadata`, `artifactmedia`, and `artifactcolors` tables
- **SQL Analytics**: 20+ predefined analytical queries for insights
- **Interactive Visualization**: Streamlit dashboard with Plotly charts

**Architecture Flow**: `Harvard API → ETL → MySQL/TiDB → SQL Analytics → Streamlit Visualization`

## Installation

```bash
# Clone the repository
git clone https://github.com/Manali0711/Harvard-Artifacts-Collection-Data-Engineering-Analytics-App.git
cd Harvard-Artifacts-Collection-Data-Engineering-Analytics-App

# Install dependencies
pip install -r requirements.txt

# Required packages
pip install streamlit pandas requests mysql-connector-python plotly
```

## Configuration

### Environment Variables

Set up your configuration using environment variables:

```bash
# Harvard Art Museums API Key
export HARVARD_API_KEY="your-api-key-here"

# Database Connection
export DB_HOST="your-database-host"
export DB_USER="your-database-user"
export DB_PASSWORD="your-database-password"
export DB_NAME="harvard_artifacts"
export DB_PORT="3306"
```

### API Key Setup

Obtain your API key from [Harvard Art Museums API](https://www.harvardartmuseums.org/collections/api):

1. Register for an account
2. Request an API key
3. Set the environment variable or configure in the Streamlit app

## Core Components

### 1. API Data Collection

```python
import requests
import pandas as pd
import os

class HarvardAPIClient:
    def __init__(self, api_key=None):
        self.api_key = api_key or os.getenv('HARVARD_API_KEY')
        self.base_url = "https://api.harvardartmuseums.org/object"
    
    def fetch_artifacts(self, page=1, size=100):
        """Fetch artifacts with pagination"""
        params = {
            'apikey': self.api_key,
            'page': page,
            'size': size
        }
        
        response = requests.get(self.base_url, params=params)
        response.raise_for_status()
        
        return response.json()
    
    def collect_batch(self, num_pages=10):
        """Collect multiple pages of artifacts"""
        all_records = []
        
        for page in range(1, num_pages + 1):
            data = self.fetch_artifacts(page=page)
            all_records.extend(data.get('records', []))
            
            # Rate limiting
            import time
            time.sleep(0.5)
        
        return all_records

# Usage
client = HarvardAPIClient()
artifacts = client.collect_batch(num_pages=5)
print(f"Collected {len(artifacts)} artifacts")
```

### 2. ETL Pipeline

```python
import pandas as pd
from datetime import datetime

class ArtifactETL:
    def __init__(self, raw_data):
        self.raw_data = raw_data
    
    def extract_metadata(self):
        """Extract artifact metadata"""
        metadata = []
        
        for artifact in self.raw_data:
            metadata.append({
                'artifact_id': artifact.get('id'),
                'title': artifact.get('title'),
                'culture': artifact.get('culture'),
                'period': artifact.get('period'),
                'century': artifact.get('century'),
                'classification': artifact.get('classification'),
                'department': artifact.get('department'),
                'dated': artifact.get('dated'),
                'medium': artifact.get('medium'),
                'dimensions': artifact.get('dimensions'),
                'creditline': artifact.get('creditline'),
                'accession_year': artifact.get('accessionyear')
            })
        
        return pd.DataFrame(metadata)
    
    def extract_media(self):
        """Extract media information"""
        media_records = []
        
        for artifact in self.raw_data:
            artifact_id = artifact.get('id')
            images = artifact.get('images', [])
            
            for img in images:
                media_records.append({
                    'artifact_id': artifact_id,
                    'media_type': 'image',
                    'media_url': img.get('baseimageurl'),
                    'iiif_url': img.get('iiifbaseuri'),
                    'width': img.get('width'),
                    'height': img.get('height')
                })
        
        return pd.DataFrame(media_records)
    
    def extract_colors(self):
        """Extract color information"""
        color_records = []
        
        for artifact in self.raw_data:
            artifact_id = artifact.get('id')
            colors = artifact.get('colors', [])
            
            for color in colors:
                color_records.append({
                    'artifact_id': artifact_id,
                    'color': color.get('color'),
                    'spectrum': color.get('spectrum'),
                    'hue': color.get('hue'),
                    'saturation': color.get('saturation'),
                    'percent': color.get('percent')
                })
        
        return pd.DataFrame(color_records)
    
    def run_etl(self):
        """Execute full ETL pipeline"""
        return {
            'metadata': self.extract_metadata(),
            'media': self.extract_media(),
            'colors': self.extract_colors()
        }

# Usage
etl = ArtifactETL(artifacts)
data = etl.run_etl()
print(f"Metadata rows: {len(data['metadata'])}")
print(f"Media rows: {len(data['media'])}")
print(f"Colors rows: {len(data['colors'])}")
```

### 3. Database Schema & Loading

```python
import mysql.connector
from mysql.connector import Error

class DatabaseManager:
    def __init__(self):
        self.connection = None
        self.connect()
    
    def connect(self):
        """Establish database connection"""
        try:
            self.connection = mysql.connector.connect(
                host=os.getenv('DB_HOST'),
                user=os.getenv('DB_USER'),
                password=os.getenv('DB_PASSWORD'),
                database=os.getenv('DB_NAME'),
                port=int(os.getenv('DB_PORT', 3306))
            )
            print("Database connection successful")
        except Error as e:
            print(f"Error connecting to database: {e}")
    
    def create_schema(self):
        """Create database tables"""
        cursor = self.connection.cursor()
        
        # Artifact Metadata Table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmetadata (
                artifact_id INT PRIMARY KEY,
                title VARCHAR(500),
                culture VARCHAR(200),
                period VARCHAR(200),
                century VARCHAR(100),
                classification VARCHAR(200),
                department VARCHAR(200),
                dated VARCHAR(200),
                medium TEXT,
                dimensions VARCHAR(500),
                creditline TEXT,
                accession_year INT
            )
        """)
        
        # Artifact Media Table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactmedia (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                media_type VARCHAR(50),
                media_url TEXT,
                iiif_url TEXT,
                width INT,
                height INT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        # Artifact Colors Table
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS artifactcolors (
                id INT AUTO_INCREMENT PRIMARY KEY,
                artifact_id INT,
                color VARCHAR(50),
                spectrum VARCHAR(50),
                hue VARCHAR(50),
                saturation FLOAT,
                percent FLOAT,
                FOREIGN KEY (artifact_id) REFERENCES artifactmetadata(artifact_id)
            )
        """)
        
        self.connection.commit()
        print("Schema created successfully")
    
    def load_data(self, df, table_name, batch_size=1000):
        """Load DataFrame to database with batch insert"""
        cursor = self.connection.cursor()
        
        # Prepare column names and placeholders
        columns = ', '.join(df.columns)
        placeholders = ', '.join(['%s'] * len(df.columns))
        
        insert_query = f"INSERT INTO {table_name} ({columns}) VALUES ({placeholders})"
        
        # Batch insert
        for i in range(0, len(df), batch_size):
            batch = df.iloc[i:i+batch_size]
            values = [tuple(x) for x in batch.values]
            
            cursor.executemany(insert_query, values)
            self.connection.commit()
        
        print(f"Loaded {len(df)} rows into {table_name}")

# Usage
db = DatabaseManager()
db.create_schema()
db.load_data(data['metadata'], 'artifactmetadata')
db.load_data(data['media'], 'artifactmedia')
db.load_data(data['colors'], 'artifactcolors')
```

### 4. SQL Analytics Queries

```python
class AnalyticsEngine:
    def __init__(self, db_connection):
        self.connection = db_connection
    
    def get_artifacts_by_culture(self):
        """Count artifacts by culture"""
        query = """
            SELECT culture, COUNT(*) as artifact_count
            FROM artifactmetadata
            WHERE culture IS NOT NULL
            GROUP BY culture
            ORDER BY artifact_count DESC
            LIMIT 20
        """
        return pd.read_sql(query, self.connection)
    
    def get_artifacts_by_century(self):
        """Artifact distribution by century"""
        query = """
            SELECT century, COUNT(*) as count
            FROM artifactmetadata
            WHERE century IS NOT NULL
            GROUP BY century
            ORDER BY count DESC
        """
        return pd.read_sql(query, self.connection)
    
    def get_media_availability(self):
        """Analyze media availability"""
        query = """
            SELECT 
                m.department,
                COUNT(DISTINCT m.artifact_id) as total_artifacts,
                COUNT(DISTINCT media.artifact_id) as artifacts_with_media,
                ROUND(COUNT(DISTINCT media.artifact_id) * 100.0 / COUNT(DISTINCT m.artifact_id), 2) as media_percentage
            FROM artifactmetadata m
            LEFT JOIN artifactmedia media ON m.artifact_id = media.artifact_id
            WHERE m.department IS NOT NULL
            GROUP BY m.department
            ORDER BY media_percentage DESC
        """
        return pd.read_sql(query, self.connection)
    
    def get_color_distribution(self):
        """Most common colors in collection"""
        query = """
            SELECT color, COUNT(*) as frequency
            FROM artifactcolors
            WHERE color IS NOT NULL
            GROUP BY color
            ORDER BY frequency DESC
            LIMIT 15
        """
        return pd.read_sql(query, self.connection)
    
    def get_classification_stats(self):
        """Classification statistics"""
        query = """
            SELECT 
                classification,
                COUNT(*) as count,
                AVG(accession_year) as avg_accession_year
            FROM artifactmetadata
            WHERE classification IS NOT NULL
            GROUP BY classification
            ORDER BY count DESC
            LIMIT 20
        """
        return pd.read_sql(query, self.connection)

# Usage
analytics = AnalyticsEngine(db.connection)
culture_data = analytics.get_artifacts_by_culture()
print(culture_data.head())
```

### 5. Streamlit Dashboard

```python
import streamlit as st
import plotly.express as px

def create_dashboard():
    """Main Streamlit dashboard"""
    st.set_page_config(page_title="Harvard Artifacts Analytics", layout="wide")
    
    st.title("🏛️ Harvard Art Museums - Analytics Dashboard")
    st.markdown("End-to-end data engineering pipeline with ETL, SQL, and visualization")
    
    # Sidebar for query selection
    st.sidebar.header("Analytics Options")
    query_type = st.sidebar.selectbox(
        "Select Analysis",
        [
            "Artifacts by Culture",
            "Artifacts by Century",
            "Media Availability",
            "Color Distribution",
            "Classification Statistics"
        ]
    )
    
    # Initialize database connection
    db = DatabaseManager()
    analytics = AnalyticsEngine(db.connection)
    
    # Execute selected query
    if query_type == "Artifacts by Culture":
        data = analytics.get_artifacts_by_culture()
        st.subheader("Top 20 Cultures by Artifact Count")
        
        col1, col2 = st.columns([2, 1])
        
        with col1:
            fig = px.bar(data, x='culture', y='artifact_count',
                        title="Artifacts by Culture",
                        labels={'artifact_count': 'Count', 'culture': 'Culture'})
            st.plotly_chart(fig, use_container_width=True)
        
        with col2:
            st.dataframe(data, height=400)
    
    elif query_type == "Color Distribution":
        data = analytics.get_color_distribution()
        st.subheader("Most Common Colors in Collection")
        
        fig = px.pie(data, names='color', values='frequency',
                    title="Color Distribution")
        st.plotly_chart(fig, use_container_width=True)
        
        st.dataframe(data)
    
    elif query_type == "Media Availability":
        data = analytics.get_media_availability()
        st.subheader("Media Availability by Department")
        
        fig = px.bar(data, x='department', y='media_percentage',
                    title="Media Availability Percentage",
                    labels={'media_percentage': 'Percentage with Media'})
        st.plotly_chart(fig, use_container_width=True)
        
        st.dataframe(data)

if __name__ == "__main__":
    create_dashboard()
```

## Running the Application

```bash
# Run the Streamlit dashboard
streamlit run app.py

# The app will be available at http://localhost:8501
```

## Common Patterns

### Rate-Limited API Fetching

```python
import time

def fetch_with_rate_limit(client, num_pages, delay=0.5):
    """Fetch data with rate limiting"""
    results = []
    
    for page in range(1, num_pages + 1):
        try:
            data = client.fetch_artifacts(page=page)
            results.extend(data.get('records', []))
            time.sleep(delay)  # Respect API rate limits
        except Exception as e:
            print(f"Error on page {page}: {e}")
            continue
    
    return results
```

### Incremental Data Loading

```python
def incremental_load(db, new_data, table_name):
    """Load only new records"""
    cursor = db.connection.cursor()
    
    # Get existing artifact IDs
    cursor.execute(f"SELECT artifact_id FROM {table_name}")
    existing_ids = set(row[0] for row in cursor.fetchall())
    
    # Filter new records
    new_records = [r for r in new_data if r['artifact_id'] not in existing_ids]
    
    if new_records:
        df = pd.DataFrame(new_records)
        db.load_data(df, table_name)
        print(f"Loaded {len(new_records)} new records")
    else:
        print("No new records to load")
```

## Troubleshooting

### API Connection Issues

```python
# Verify API key
client = HarvardAPIClient()
try:
    test_data = client.fetch_artifacts(page=1, size=1)
    print("API connection successful")
except requests.exceptions.HTTPError as e:
    if e.response.status_code == 401:
        print("Invalid API key - check HARVARD_API_KEY environment variable")
    else:
        print(f"API error: {e}")
```

### Database Connection Problems

```python
# Test database connection
try:
    db = DatabaseManager()
    cursor = db.connection.cursor()
    cursor.execute("SELECT 1")
    print("Database connection OK")
except Exception as e:
    print(f"Database error: {e}")
    print("Check DB_HOST, DB_USER, DB_PASSWORD, DB_NAME environment variables")
```

### Missing Data Handling

```python
def safe_extract(artifact, key, default=None):
    """Safely extract nested data"""
    return artifact.get(key, default)

# Use in ETL
metadata = {
    'artifact_id': safe_extract(artifact, 'id'),
    'title': safe_extract(artifact, 'title', 'Untitled'),
    'culture': safe_extract(artifact, 'culture', 'Unknown')
}
```

## Best Practices

1. **Always use environment variables** for sensitive data (API keys, DB credentials)
2. **Implement rate limiting** when fetching from APIs
3. **Use batch inserts** for database performance (1000 rows per batch recommended)
4. **Handle NULL values** in SQL queries with `IS NOT NULL` filters
5. **Create indexes** on frequently queried columns (artifact_id, culture, department)
6. **Test with small datasets** before running full ETL pipeline
7. **Log ETL operations** for debugging and monitoring

This skill equips AI agents to build production-ready data engineering pipelines with API integration, ETL processes, SQL analytics, and interactive visualizations.
