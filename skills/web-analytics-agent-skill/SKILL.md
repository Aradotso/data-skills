---
name: web-analytics-agent-skill
description: SEO automation and traffic diagnosis using Google Search Console, GA4, and Bing Webmaster APIs
triggers:
  - "analyze my website traffic and SEO performance"
  - "fetch Google Search Console data for my site"
  - "compare my search rankings on Google and Bing"
  - "check my GA4 analytics and user sessions"
  - "diagnose SEO issues using web analytics"
  - "pull keyword rankings from search console"
  - "generate a web analytics report"
  - "set up SEO monitoring for my website"
---

# Web Analytics Agent Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to perform comprehensive SEO automation and traffic analysis by integrating Google Search Console (GSC), Google Analytics 4 (GA4), and Bing Webmaster Tools APIs. It provides autonomous keyword research, traffic diagnosis, and cross-platform search performance monitoring.

## What This Project Does

Web Analytics Agent Skill is a Python-based automation toolkit that:
- Fetches indexing status, keyword rankings, clicks, and impressions from Google Search Console
- Retrieves user sessions, bounce rates, and traffic sources from Google Analytics 4
- Pulls search statistics from Bing Webmaster Tools for cross-engine comparison
- Generates structured reports combining data from all three platforms
- Supports OAuth 2.0 for Google services and API key auth for Bing

## Installation

### 1. Clone and Set Up Environment

```bash
# Clone the repository
git clone https://github.com/SeoToolkit/web-analytics-agent-skill.git
cd web-analytics-agent-skill

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Configure Authentication

#### Bing Webmaster Tools Setup

1. Visit [Bing Webmaster Tools](https://www.bing.com/webmasters/)
2. Navigate to Settings (gear icon) → API Access → API Key
3. Generate and copy your API key

#### Google Search Console & GA4 Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project
3. Enable **Google Search Console API** and **Google Analytics Data API**
4. Configure OAuth consent screen:
   - Choose "External" type
   - Add your email as a test user
   - Add scopes: `https://www.googleapis.com/auth/webmasters.readonly` and `https://www.googleapis.com/auth/analytics.readonly`
5. Create OAuth 2.0 credentials:
   - Go to Credentials → Create Credentials → OAuth client ID
   - Select "Desktop app"
   - Download the JSON file
   - Rename it to `client_secret.json` and place in project root

### 3. Environment Configuration

Create `.env` file in project root:

```ini
# Bing Webmaster Tools
BING_API_KEY=${BING_API_KEY}
BING_SITE_URL=https://yourdomain.com

# Google Search Console
GSC_SITE_URL=sc-domain:yourdomain.com
SITE_LAUNCH_DATE=2024-01-01

# Google Analytics 4 (comma-separated for multiple properties)
GA4_PROPERTIES=123456789=MainSite,987654321=BlogSite
```

**Important Notes:**
- `GSC_SITE_URL` format: Use `sc-domain:example.com` for domain properties or `https://example.com/` for URL-prefix properties
- `GA4_PROPERTIES` format: `PropertyID=Label` (find Property ID in GA4 Admin → Property Settings)
- The Google account used for OAuth must have read access to both GSC and GA4 properties

## Key Commands

### Unified Analysis Script

Run all analytics tools sequentially:

```bash
./run_all.sh
```

This script automatically:
1. Activates virtual environment
2. Checks and installs dependencies
3. Runs Google authorization (if needed)
4. Executes GSC, GA4, and Bing analysis scripts
5. Outputs combined reports

### Individual Scripts

Run specific analysis tools:

```bash
# Activate environment first
source .venv/bin/activate

# Google OAuth authorization (run once or when token expires)
python3 scripts/auth_google.py

# Google Search Console analysis
python3 scripts/analyze_gsc.py

# Google Analytics 4 data
python3 scripts/ga4_both.py

# Bing Webmaster Tools
python3 scripts/bing_webmaster.py
```

## Python API Usage

### Google Search Console

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
import os
from datetime import datetime, timedelta

# Load credentials
creds = Credentials.from_authorized_user_file('token.json')
service = build('searchconsole', 'v1', credentials=creds)

# Define date range
end_date = datetime.now().date()
start_date = end_date - timedelta(days=7)

# Query search analytics
request = {
    'startDate': start_date.isoformat(),
    'endDate': end_date.isoformat(),
    'dimensions': ['query', 'page'],
    'rowLimit': 100,
    'startRow': 0
}

site_url = os.getenv('GSC_SITE_URL')
response = service.searchanalytics().query(
    siteUrl=site_url,
    body=request
).execute()

# Process results
for row in response.get('rows', []):
    query = row['keys'][0]
    page = row['keys'][1]
    clicks = row['clicks']
    impressions = row['impressions']
    ctr = row['ctr']
    position = row['position']
    
    print(f"Query: {query}")
    print(f"  Page: {page}")
    print(f"  Clicks: {clicks}, Impressions: {impressions}")
    print(f"  CTR: {ctr:.2%}, Position: {position:.1f}\n")
```

### Google Analytics 4

```python
from google.analytics.data_v1beta import BetaAnalyticsDataClient
from google.analytics.data_v1beta.types import (
    DateRange,
    Dimension,
    Metric,
    RunReportRequest,
)
from google.oauth2.credentials import Credentials
import os

# Load credentials
creds = Credentials.from_authorized_user_file('token.json')
client = BetaAnalyticsDataClient(credentials=creds)

# Parse property ID
ga4_props = os.getenv('GA4_PROPERTIES', '').split(',')
property_id, label = ga4_props[0].split('=')

# Build report request
request = RunReportRequest(
    property=f"properties/{property_id}",
    date_ranges=[DateRange(start_date="7daysAgo", end_date="today")],
    dimensions=[
        Dimension(name="sessionSource"),
        Dimension(name="sessionMedium")
    ],
    metrics=[
        Metric(name="sessions"),
        Metric(name="activeUsers"),
        Metric(name="bounceRate"),
        Metric(name="averageSessionDuration")
    ],
)

# Execute request
response = client.run_report(request)

# Process results
for row in response.rows:
    source = row.dimension_values[0].value
    medium = row.dimension_values[1].value
    sessions = row.metric_values[0].value
    users = row.metric_values[1].value
    bounce = row.metric_values[2].value
    duration = row.metric_values[3].value
    
    print(f"Source/Medium: {source}/{medium}")
    print(f"  Sessions: {sessions}, Users: {users}")
    print(f"  Bounce Rate: {float(bounce):.2%}")
    print(f"  Avg Duration: {float(duration):.1f}s\n")
```

### Bing Webmaster Tools

```python
import requests
import os
from datetime import datetime, timedelta

api_key = os.getenv('BING_API_KEY')
site_url = os.getenv('BING_SITE_URL')

# Calculate date range
end_date = datetime.now().date()
start_date = end_date - timedelta(days=7)

# Query stats endpoint
base_url = "https://ssl.bing.com/webmaster/api.svc/json/GetQueryStats"
params = {
    'siteUrl': site_url,
    'query': '',  # Empty for all queries
    'startDate': start_date.isoformat(),
    'endDate': end_date.isoformat()
}

headers = {
    'Authorization': f'Bearer {api_key}',
    'Content-Type': 'application/json'
}

response = requests.get(base_url, params=params, headers=headers)
data = response.json()

# Process results
for item in data.get('d', []):
    query = item.get('Query')
    clicks = item.get('Clicks')
    impressions = item.get('Impressions')
    avg_position = item.get('AvgPosition')
    
    print(f"Query: {query}")
    print(f"  Clicks: {clicks}, Impressions: {impressions}")
    print(f"  Avg Position: {avg_position:.1f}\n")
```

## Common Patterns

### Cross-Platform Keyword Comparison

```python
# Fetch keyword data from both GSC and Bing
gsc_keywords = fetch_gsc_keywords(start_date, end_date)
bing_keywords = fetch_bing_keywords(start_date, end_date)

# Create comparison dictionary
comparison = {}
for kw in gsc_keywords:
    query = kw['query']
    comparison[query] = {
        'google': {
            'clicks': kw['clicks'],
            'position': kw['position']
        },
        'bing': {'clicks': 0, 'position': None}
    }

for kw in bing_keywords:
    query = kw['query']
    if query in comparison:
        comparison[query]['bing'] = {
            'clicks': kw['clicks'],
            'position': kw['position']
        }
    else:
        comparison[query] = {
            'google': {'clicks': 0, 'position': None},
            'bing': {
                'clicks': kw['clicks'],
                'position': kw['position']
            }
        }

# Identify opportunities
for query, data in comparison.items():
    google_clicks = data['google']['clicks']
    bing_clicks = data['bing']['clicks']
    
    if bing_clicks > google_clicks * 1.5:
        print(f"⚡ Opportunity: '{query}' performs better on Bing")
```

### Traffic Source Attribution

```python
# Combine GA4 session data with GSC landing pages
ga4_sessions = fetch_ga4_traffic_sources()
gsc_pages = fetch_gsc_top_pages()

# Map landing pages to traffic sources
attribution = {}
for page_data in gsc_pages:
    page = page_data['page']
    organic_clicks = page_data['clicks']
    
    # Find corresponding GA4 sessions
    ga4_match = next(
        (s for s in ga4_sessions if page in s.get('landingPage', '')),
        None
    )
    
    if ga4_match:
        attribution[page] = {
            'organic_clicks': organic_clicks,
            'total_sessions': ga4_match['sessions'],
            'bounce_rate': ga4_match['bounceRate'],
            'conversion_rate': ga4_match.get('conversionRate', 0)
        }
```

### Automated Weekly Reporting

```python
from datetime import datetime, timedelta
import json

def generate_weekly_report():
    end_date = datetime.now().date()
    start_date = end_date - timedelta(days=7)
    
    report = {
        'period': {
            'start': start_date.isoformat(),
            'end': end_date.isoformat()
        },
        'google': {
            'search_console': fetch_gsc_summary(start_date, end_date),
            'analytics': fetch_ga4_summary(start_date, end_date)
        },
        'bing': fetch_bing_summary(start_date, end_date)
    }
    
    # Save report
    report_path = f"reports/weekly_{end_date.isoformat()}.json"
    with open(report_path, 'w') as f:
        json.dump(report, f, indent=2)
    
    return report_path
```

## Troubleshooting

### 403 Permission Errors (Google APIs)

**Problem:** `HttpError 403: User does not have sufficient permissions`

**Solution:**
1. Verify the Google account has "Owner" or "Full User" role in GSC/GA4
2. Re-run authentication: `python3 scripts/auth_google.py`
3. During OAuth consent, ensure you check **all** permission boxes
4. Check OAuth scopes in `client_secret.json` match required permissions
5. If using a service account, ensure it's added as a user in GSC/GA4 properties

### Token Expiration

**Problem:** `RefreshError: invalid_grant`

**Solution:**
```bash
# Delete expired token
rm token.json

# Re-authenticate
python3 scripts/auth_google.py
```

### Bing API Rate Limits

**Problem:** `429 Too Many Requests`

**Solution:**
- Bing Webmaster API has a limit of ~10,000 calls/day
- Implement exponential backoff:

```python
import time
from requests.exceptions import HTTPError

def bing_api_call_with_retry(url, headers, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(url, headers=headers)
            response.raise_for_status()
            return response.json()
        except HTTPError as e:
            if e.response.status_code == 429:
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### Missing GA4 Property ID

**Problem:** Cannot find GA4 Property ID

**Solution:**
1. Log in to [Google Analytics](https://analytics.google.com/)
2. Click Admin (gear icon, bottom left)
3. Select your property
4. Go to Property Settings
5. Copy the numeric Property ID (e.g., `123456789`)

### GSC Domain vs URL-Prefix Properties

**Problem:** Site URL format confusion

**Solution:**
- **Domain property:** Use `sc-domain:example.com` (requires DNS verification)
- **URL-prefix property:** Use `https://example.com/` (note trailing slash)
- Check your exact property URL in [Google Search Console](https://search.google.com/search-console)

### Virtual Environment Issues

**Problem:** Command not found or import errors

**Solution:**
```bash
# Ensure virtual environment is activated
source .venv/bin/activate

# Verify Python version (requires 3.7+)
python3 --version

# Reinstall dependencies
pip install --upgrade -r requirements.txt
```

## Configuration Reference

### Required Files

- **`.env`**: Environment variables (API keys, site URLs)
- **`client_secret.json`**: Google OAuth 2.0 credentials (downloaded from Cloud Console)
- **`token.json`**: Auto-generated OAuth access token (created on first run)

### Environment Variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `BING_API_KEY` | For Bing | Bing Webmaster API key | `abc123...` |
| `BING_SITE_URL` | For Bing | Verified domain in Bing | `https://example.com` |
| `GSC_SITE_URL` | For GSC | Search Console property URL | `sc-domain:example.com` |
| `GA4_PROPERTIES` | For GA4 | Property ID and label pairs | `123456789=Site1,987654321=Site2` |
| `SITE_LAUNCH_DATE` | Optional | Website launch date for metrics | `2024-01-01` |

### OAuth Scopes

The `client_secret.json` must include these scopes:
- `https://www.googleapis.com/auth/webmasters.readonly` (GSC read access)
- `https://www.googleapis.com/auth/analytics.readonly` (GA4 read access)

## Output Formats

All scripts output structured text reports. Example output structure:

```
=== Google Search Console Report ===
Period: 2024-01-01 to 2024-01-07
Total Clicks: 1,234
Total Impressions: 45,678
Average CTR: 2.7%
Average Position: 12.3

Top Keywords:
1. "example keyword" - 234 clicks, Pos 5.2
2. "another query" - 189 clicks, Pos 8.7
...

=== Google Analytics 4 Report ===
Property: example.com (123456789)
Sessions: 3,456
Active Users: 2,890
Bounce Rate: 45.2%
Avg Session Duration: 125.3s

Traffic Sources:
1. google/organic - 1,234 sessions
2. direct/none - 890 sessions
...

=== Bing Webmaster Report ===
Period: 2024-01-01 to 2024-01-07
Total Clicks: 456
Total Impressions: 12,345
Average Position: 15.7

Top Queries:
1. "bing keyword" - 78 clicks, Pos 11.2
2. "another term" - 56 clicks, Pos 18.5
...
```
