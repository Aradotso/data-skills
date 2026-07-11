---
name: power-bi-retail-analytics-salespulse360
description: Deploy and customize the SalesPulse 360 Power BI retail analytics dashboard for profit, regional, and sales performance analysis
triggers:
  - set up the SalesPulse 360 Power BI dashboard
  - configure retail analytics dashboard in Power BI
  - customize the Global Superstore sales visualization
  - implement multi-region profit analysis dashboard
  - deploy Power BI sales performance tracking
  - create interactive retail analytics with SalesPulse 360
  - troubleshoot Power BI dashboard refresh issues
  - localize the sales dashboard for multiple languages
---

# SalesPulse 360 Power BI Retail Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill enables AI agents to help developers deploy, customize, and maintain the SalesPulse 360 Power BI dashboard—a comprehensive retail analytics solution built for Global Superstore sales data with multi-regional profit analysis, predictive forecasting, and automated alerting.

## What It Does

SalesPulse 360 transforms raw retail transaction data into an interactive command center featuring:

- **Multi-granularity sales/profit analysis** (global → transaction level)
- **Predictive forecasting** with 12-month confidence corridors
- **Automated exception alerts** for profit margin and return rate thresholds
- **Multi-language support** with cultural localization (English, Spanish, French, German, Mandarin)
- **Row-level security (RLS)** for region-based data access
- **Responsive layout** adapting to desktop/tablet/mobile
- **24/7 autonomous refresh** engine for live data synchronization

The dashboard operates on the Global Superstore dataset (2011-2015) but can be adapted to any retail/e-commerce data source with similar schema.

## Installation & Setup

### Prerequisites

- Power BI Desktop (version 2.120 or higher)
- Power BI Service account (Pro or Premium for publishing)
- Modern web browser for HTML interface
- Optional: SQL Server or Azure SQL for production data sources

### Step 1: Clone the Repository

```bash
git clone https://github.com/MahbubNibir/power-bi-retail-analytics-viz.git
cd power-bi-retail-analytics-viz
```

### Step 2: Extract Dataset

The Global Superstore data is provided as a compressed CSV:

```bash
# Extract the data archive
unzip data/global_superstore.zip -d data/
# Or use tar if provided as .tar.gz
tar -xzf data/global_superstore.tar.gz -C data/
```

### Step 3: Open the Power BI File

```bash
# Open the main dashboard file
powershell -Command "Start-Process 'SalesPulse360.pbix'"
# Or simply double-click SalesPulse360.pbix in file explorer
```

On first load, Power BI Desktop will:
1. Execute data transformation pipeline (M queries)
2. Apply data cleaning rules
3. Build calculated measures and columns
4. Initialize refresh schedule settings

**Expected load time:** 2-3 minutes on standard hardware for the full dataset (9,994 rows).

## Key Components & Configuration

### Data Source Configuration

The dashboard uses Power Query (M language) for data ingestion. Default source is the local CSV, but you can redirect to live databases:

**Modify the data source path:**

1. Open Power BI Desktop → Transform Data
2. In Power Query Editor, select "Source" step in Applied Steps
3. Update file path or connection string:

```m
// For local CSV
let
    Source = Csv.Document(File.Contents("C:\path\to\data\global_superstore.csv"), [Delimiter=",", Columns=24, Encoding=65001, QuoteStyle=QuoteStyle.None])
in
    Source

// For SQL Server (production)
let
    Source = Sql.Database(Environment.GetEnvironmentVariable("SQL_SERVER"), Environment.GetEnvironmentVariable("SQL_DATABASE")),
    dbo_Orders = Source{[Schema="dbo",Item="Orders"]}[Data]
in
    dbo_Orders
```

### Threshold Calibration

The dashboard includes a `Settings` table that controls alert thresholds:

**Location:** Power BI Desktop → Model View → Settings table

| Parameter | Default | Description |
|-----------|---------|-------------|
| ProfitMarginThreshold | 15% | Alert when margin drops below |
| ReturnRateWarning | 8% | Alert when returns exceed |
| RollingAverageDays | 90 | Period for MA calculations |
| ForecastMonths | 12 | Prediction horizon |

**To modify programmatically:**

```dax
// In DAX, create or update the Settings table
Settings = 
DATATABLE(
    "Parameter", STRING,
    "Value", DOUBLE,
    {
        {"ProfitMarginThreshold", 0.15},
        {"ReturnRateWarning", 0.08},
        {"RollingAverageDays", 90},
        {"ForecastMonths", 12}
    }
)
```

### Calculated Measures (DAX)

Key measures driving the dashboard analytics:

```dax
// Total Sales
Total Sales = SUM(Orders[Sales])

// Total Profit
Total Profit = SUM(Orders[Profit])

// Profit Margin
Profit Margin = 
DIVIDE(
    [Total Profit],
    [Total Sales],
    0
)

// Moving Annual Total (MAT)
MAT Sales = 
CALCULATE(
    [Total Sales],
    DATESINPERIOD(
        'Calendar'[Date],
        MAX('Calendar'[Date]),
        -12,
        MONTH
    )
)

// Period-over-Period Growth
PoP Growth = 
VAR CurrentPeriod = [Total Sales]
VAR PreviousPeriod = 
    CALCULATE(
        [Total Sales],
        DATEADD('Calendar'[Date], -1, QUARTER)
    )
RETURN
DIVIDE(
    CurrentPeriod - PreviousPeriod,
    PreviousPeriod,
    0
)

// Profit Margin Alert (Sentinel System)
Profit Alert = 
VAR CurrentMargin = [Profit Margin]
VAR Threshold = LOOKUPVALUE(Settings[Value], Settings[Parameter], "ProfitMarginThreshold")
RETURN
IF(
    CurrentMargin < Threshold,
    "⚠️ Below Threshold",
    "✓ Healthy"
)

// Forecasted Sales (12-month projection)
Forecast Sales = 
VAR HistoricalData = 
    CALCULATETABLE(
        SUMMARIZE(
            Orders,
            'Calendar'[Date],
            "Sales", [Total Sales]
        ),
        DATESINPERIOD(
            'Calendar'[Date],
            MAX('Calendar'[Date]),
            -24,
            MONTH
        )
    )
VAR ForecastPeriod = 
    LOOKUPVALUE(Settings[Value], Settings[Parameter], "ForecastMonths")
RETURN
// Note: Power BI native forecasting uses built-in analytics pane
// For custom forecasting, integrate Python/R scripts or Azure ML
```

### Row-Level Security (RLS)

To restrict data visibility by region:

1. **Model View** → Manage Roles
2. Create role for each region:

```dax
// Role: South Region Manager
[Region] = "South"

// Role: Multi-Region Director
[Region] IN {"South", "Central"}

// Role: Executive (all access)
1 = 1
```

3. **Test RLS:** Model View → View As → Select role

**Assign users after publishing:**

```powershell
# Power BI Service → Workspace → Dataset Settings → Security
# Add Azure AD users/groups to corresponding roles
```

## Publishing to Power BI Service

### Step 1: Publish from Desktop

```powershell
# In Power BI Desktop
# File → Publish → Select workspace
# Or use keyboard shortcut: Alt+Shift+P
```

### Step 2: Configure Scheduled Refresh

**Power BI Service → Workspace → Dataset → Settings → Scheduled refresh**

```json
{
  "refreshSchedule": {
    "frequency": "Daily",
    "times": ["02:00"],
    "localTimeZoneId": "UTC",
    "notifyOnFailure": true
  },
  "gatewayConnection": {
    "dataSource": "OnPremisesDataGateway",
    "credentials": "UseOAuth2"
  }
}
```

For near-real-time updates, configure streaming dataset:

```csharp
// C# example: Push data to Power BI streaming dataset
using Microsoft.PowerBI.Api.V2;
using Microsoft.PowerBI.Api.V2.Models;

var credentials = new TokenCredentials(Environment.GetEnvironmentVariable("POWERBI_ACCESS_TOKEN"));
var client = new PowerBIClient(credentials);

var datasetId = Environment.GetEnvironmentVariable("POWERBI_DATASET_ID");
var rows = new List<object>
{
    new {
        OrderDate = DateTime.Now,
        Region = "West",
        Sales = 1250.50,
        Profit = 187.58
    }
};

await client.Datasets.PostRowsAsync(datasetId, "Orders", new PostRowsRequest(rows));
```

## Common Patterns & Use Cases

### Pattern 1: Drill-Down Navigation

Enable users to navigate from global to granular views:

```dax
// Create hierarchy in Model View
Region Hierarchy = 
    'Geography'[Region] 
    → 'Geography'[Country] 
    → 'Geography'[City] 
    → 'Geography'[PostalCode]

// In visual interactions:
// Map visual → Drill mode → Click region to drill down
```

### Pattern 2: Custom Alerts with Power Automate

Trigger email alerts when thresholds are breached:

```json
// Power Automate flow configuration
{
  "trigger": {
    "type": "PowerBI.DatasetRefreshCompleted",
    "datasetId": "$POWERBI_DATASET_ID"
  },
  "actions": [
    {
      "type": "Query",
      "query": "EVALUATE FILTER(Orders, [Profit Margin] < 0.15)",
      "condition": "RowCount > 0",
      "then": {
        "type": "Email",
        "to": "$ALERT_RECIPIENT_EMAIL",
        "subject": "⚠️ Profit Margin Alert",
        "body": "Low-margin orders detected. Review dashboard immediately."
      }
    }
  ]
}
```

### Pattern 3: Multi-Language Localization

Implement dynamic label switching:

```dax
// Create Translations table
Translations = 
DATATABLE(
    "Key", STRING,
    "English", STRING,
    "Spanish", STRING,
    "French", STRING,
    {
        {"Sales", "Sales", "Ventas", "Ventes"},
        {"Profit", "Profit", "Beneficio", "Bénéfice"},
        {"Region", "Region", "Región", "Région"}
    }
)

// Create language selector (Slicer visual)
Language = {"English", "Spanish", "French"}

// Dynamic measure for labels
Dynamic Label = 
VAR SelectedLang = SELECTEDVALUE(Language[Value], "English")
VAR Key = "Sales"
RETURN
SWITCH(
    SelectedLang,
    "English", LOOKUPVALUE(Translations[English], Translations[Key], Key),
    "Spanish", LOOKUPVALUE(Translations[Spanish], Translations[Key], Key),
    "French", LOOKUPVALUE(Translations[French], Translations[Key], Key),
    LOOKUPVALUE(Translations[English], Translations[Key], Key)
)
```

### Pattern 4: Export Data for External Analysis

Extract filtered data to CSV for Python/R analysis:

```python
# Python script in Power BI Desktop (External Tools → Python)
import pandas as pd
import os

# Power BI automatically provides 'dataset' variable
df = dataset

# Apply filters (inherits from current dashboard state)
filtered = df[df['Region'] == 'West']

# Export
output_path = os.path.join(os.getenv('TEMP'), 'west_region_export.csv')
filtered.to_csv(output_path, index=False)

print(f"Exported {len(filtered)} rows to {output_path}")
```

## HTML Interface Customization

The repository includes an HTML landing page (`index.html`) for web deployment:

```html
<!-- Embed Power BI report in custom HTML -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SalesPulse 360</title>
    <style>
        #reportContainer {
            width: 100%;
            height: 100vh;
            border: none;
        }
    </style>
</head>
<body>
    <iframe 
        id="reportContainer"
        src="https://app.powerbi.com/reportEmbed?reportId=YOUR_REPORT_ID&autoAuth=true"
        frameborder="0" 
        allowFullScreen="true">
    </iframe>

    <script src="https://cdn.jsdelivr.net/npm/powerbi-client@2.19.0/dist/powerbi.min.js"></script>
    <script>
        const embedConfig = {
            type: 'report',
            id: process.env.POWERBI_REPORT_ID,
            embedUrl: `https://app.powerbi.com/reportEmbed?reportId=${process.env.POWERBI_REPORT_ID}`,
            accessToken: process.env.POWERBI_EMBED_TOKEN,
            tokenType: models.TokenType.Embed,
            settings: {
                panes: {
                    filters: { expanded: false, visible: true },
                    pageNavigation: { visible: true }
                },
                background: models.BackgroundType.Transparent
            }
        };

        const reportContainer = document.getElementById('reportContainer');
        const powerbi = new pbi.service.Service(pbi.factories.hpmFactory, pbi.factories.wpmpFactory, pbi.factories.routerFactory);
        const report = powerbi.embed(reportContainer, embedConfig);

        report.on('loaded', () => console.log('Report loaded'));
        report.on('error', (event) => console.error('Error:', event.detail));
    </script>
</body>
</html>
```

**Generate embed tokens:**

```javascript
// Node.js backend for token generation
const msal = require('@azure/msal-node');
const axios = require('axios');

const msalConfig = {
    auth: {
        clientId: process.env.AZURE_CLIENT_ID,
        clientSecret: process.env.AZURE_CLIENT_SECRET,
        authority: `https://login.microsoftonline.com/${process.env.AZURE_TENANT_ID}`
    }
};

const cca = new msal.ConfidentialClientApplication(msalConfig);

async function getEmbedToken(reportId) {
    const tokenRequest = {
        scopes: ['https://analysis.windows.net/powerbi/api/.default']
    };
    
    const response = await cca.acquireTokenByClientCredential(tokenRequest);
    const accessToken = response.accessToken;

    const embedUrl = `https://api.powerbi.com/v1.0/myorg/reports/${reportId}/GenerateToken`;
    const embedResponse = await axios.post(embedUrl, 
        { accessLevel: 'View' },
        { headers: { 'Authorization': `Bearer ${accessToken}` }}
    );

    return embedResponse.data.token;
}
```

## Troubleshooting

### Issue: Data Refresh Fails

**Symptoms:** "Unable to connect to data source" error in Power BI Service

**Solutions:**

1. **Check gateway connection:**
   ```powershell
   # Test on-premises data gateway
   Test-NetConnection -ComputerName gateway.domain.local -Port 8050
   ```

2. **Verify credentials:**
   - Service → Settings → Data source credentials
   - Re-enter credentials using OAuth2 or service account

3. **Firewall rules:**
   ```bash
   # Allow Power BI Service IPs (check latest ranges)
   # https://learn.microsoft.com/en-us/power-bi/enterprise/service-admin-power-bi-security
   ```

### Issue: Slow Dashboard Performance

**Symptoms:** Visuals take >5 seconds to load

**Solutions:**

1. **Optimize DAX measures:**
   ```dax
   // AVOID: Iterators over large tables
   Bad Practice = SUMX(Orders, Orders[Sales] * 1.1)
   
   // PREFER: Aggregation + calculation
   Good Practice = [Total Sales] * 1.1
   ```

2. **Reduce data model size:**
   ```m
   // In Power Query, filter early
   let
       Source = Csv.Document(...),
       FilteredRows = Table.SelectRows(Source, each [OrderDate] >= #date(2014, 1, 1)),
       RemovedColumns = Table.RemoveColumns(FilteredRows, {"Unnecessary1", "Unnecessary2"})
   in
       RemovedColumns
   ```

3. **Enable query folding:**
   ```m
   // Use native query for SQL sources
   let
       Source = Sql.Database("server", "db"),
       NativeQuery = Value.NativeQuery(Source, "
           SELECT Region, SUM(Sales) AS TotalSales
           FROM Orders
           WHERE OrderDate >= '2014-01-01'
           GROUP BY Region
       ")
   in
       NativeQuery
   ```

### Issue: RLS Not Filtering Correctly

**Symptoms:** Users see data outside their assigned region

**Solutions:**

1. **Validate role definition:**
   ```dax
   // Ensure role uses correct column
   // WRONG: [RegionName] = "South"
   // CORRECT: 'Geography'[Region] = "South"
   ```

2. **Check user assignment:**
   - Power BI Service → Dataset Security
   - Verify email matches Azure AD

3. **Test with username():**
   ```dax
   // Dynamic RLS based on user email
   [Region] = LOOKUPVALUE(
       UserRegionMapping[Region],
       UserRegionMapping[Email],
       USERNAME()
   )
   ```

### Issue: Forecasting Not Appearing

**Symptoms:** Forecast line missing in line chart

**Solutions:**

1. **Enable Analytics pane:**
   - Select line chart → Analytics pane (graph icon) → Forecast → Add

2. **Ensure date table continuity:**
   ```dax
   // Create continuous calendar
   Calendar = 
   CALENDAR(
       DATE(2011, 1, 1),
       DATE(2027, 12, 31)  // Extend beyond data range
   )
   ```

3. **Minimum data points:**
   - Forecasting requires at least 2 data points per season
   - For monthly forecast: minimum 24 months of history

## Environment Variables Reference

The project uses these environment variables for secure configuration:

| Variable | Purpose | Example |
|----------|---------|---------|
| `SQL_SERVER` | Production database server | `prod-sql.database.windows.net` |
| `SQL_DATABASE` | Database name | `RetailAnalytics` |
| `POWERBI_ACCESS_TOKEN` | Service principal token | (generated via MSAL) |
| `POWERBI_DATASET_ID` | Published dataset GUID | `abc123-def456-...` |
| `POWERBI_REPORT_ID` | Published report GUID | `xyz789-uvw012-...` |
| `AZURE_CLIENT_ID` | App registration ID | `12345678-1234-...` |
| `AZURE_CLIENT_SECRET` | App secret | (from Azure portal) |
| `AZURE_TENANT_ID` | Directory ID | `87654321-4321-...` |
| `ALERT_RECIPIENT_EMAIL` | Alert destination | `ops-team@company.com` |

**Never commit these to version control.** Use `.env` files locally and Key Vault in production.

## Advanced Customization

### Adding New Data Sources

To integrate CRM or inventory systems:

```m
// Power Query: Merge Salesforce data
let
    SalesforceSource = Salesforce.Data("https://instance.salesforce.com", [ApiVersion = "v52.0"]),
    Opportunities = SalesforceSource{[Name="Opportunity"]}[Data],
    JoinedData = Table.NestedJoin(
        Orders, {"CustomerID"},
        Opportunities, {"AccountId"},
        "OpportunityData",
        JoinKind.LeftOuter
    )
in
    JoinedData
```

### Custom Visuals Integration

Install third-party visuals for advanced analytics:

```bash
# Download .pbiviz file from AppSource
# Power BI Desktop → Insert → Get more visuals → Import from file
```

Example: Hierarchy Slicer for multi-level filtering

```json
{
  "visualType": "hierarchySlicer",
  "config": {
    "levels": ["Region", "Country", "City"],
    "expandAll": false,
    "singleSelect": true
  }
}
```

This skill covers the complete lifecycle of deploying and maintaining the SalesPulse 360 dashboard. AI agents can now guide developers through installation, customization, security configuration, performance optimization, and troubleshooting.
