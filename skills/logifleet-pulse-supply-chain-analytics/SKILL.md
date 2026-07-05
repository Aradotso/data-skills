---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet, and warehouse analytics with multi-fact star schema
triggers:
  - set up logistics analytics warehouse
  - deploy supply chain dashboard with power bi
  - configure fleet and warehouse data model
  - implement logifleet pulse schema
  - create logistics intelligence platform
  - build cross-modal supply chain analytics
  - integrate warehouse and fleet telemetry data
  - design multi-fact star schema for logistics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers deploy and customize LogiFleet Pulse, an advanced MS SQL Server and Power BI template for unified logistics, fleet, and warehouse analytics using a multi-fact star schema architecture.

## What LogiFleet Pulse Does

LogiFleet Pulse is a data warehousing and visualization template that:

- **Unifies disparate logistics data** from warehouse management systems (WMS), fleet telemetry, supplier portals, and external APIs
- **Implements a multi-fact star schema** with time-phased dimensions for cross-fact KPI harmonization
- **Provides Power BI dashboards** for real-time operational visibility (15-minute refresh cycles)
- **Enables predictive analytics** through composite aggregation bridges and temporal elasticity modeling
- **Supports role-based access control** with row-level security for GDPR/SOC2 compliance

Core concepts:
- **Cross-Fact KPI Harmonization**: Links warehouse operations (dwell time, pick rates) with fleet metrics (fuel consumption, idle time)
- **Warehouse Gravity Zones™**: Spatial optimization based on pick frequency, item fragility, and replenishment lead time
- **Adaptive Fleet Triage Engine**: Prioritizes maintenance by revenue impact, not just severity
- **Temporal Elasticity Modeling**: Runs time-phased simulation scenarios for capacity planning

## Installation

### Prerequisites

- **MS SQL Server 2019+** (Express, Standard, or Enterprise)
- **Power BI Desktop** (latest version recommended)
- **Data sources**: WMS, telemetry feeds, ERP systems (accessible via SQL, REST API, or file export)
- **Optional**: Azure Synapse Analytics for big data enrichment

### Deployment Steps

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema** to your SQL Server instance:
```bash
# Using sqlcmd (Windows/Linux)
sqlcmd -S YOUR_SERVER_NAME -d YOUR_DATABASE_NAME -i schema/deploy_full_schema.sql -U YOUR_USERNAME -P

# Or via SQL Server Management Studio (SSMS)
# Open schema/deploy_full_schema.sql and execute against your target database
```

3. **Configure data source connections**:
```bash
# Copy the sample configuration
cp config_sample.json config.json

# Edit with your connection strings
# DO NOT commit config.json (it's in .gitignore)
```

4. **Load initial data** (if using sample datasets):
```bash
sqlcmd -S YOUR_SERVER_NAME -d YOUR_DATABASE_NAME -i data/load_sample_data.sql -U YOUR_USERNAME -P
```

5. **Import Power BI template**:
   - Open Power BI Desktop
   - File → Open → `LogiFleet_Pulse_Master.pbit`
   - Enter SQL Server connection parameters when prompted
   - The template auto-detects fact tables and builds relationships

## SQL Schema Architecture

### Core Fact Tables

**FactWarehouseOperations** — Granular warehouse transactions:
```sql
-- Example: Query average dwell time by product category
SELECT 
    dp.ProductCategory,
    AVG(DATEDIFF(HOUR, fwo.PutawayDateTime, fwo.PickDateTime)) AS AvgDwellHours,
    COUNT(*) AS TotalOperations
FROM FactWarehouseOperations fwo
INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.FiscalYear = 2026 AND dt.FiscalQuarter = 2
GROUP BY dp.ProductCategory
ORDER BY AvgDwellHours DESC;
```

**FactFleetTrips** — Fleet telemetry and route performance:
```sql
-- Example: Identify routes with excessive idle time
SELECT 
    dg.RouteName,
    SUM(fft.IdleTimeMinutes) AS TotalIdleMinutes,
    SUM(fft.TripDurationMinutes) AS TotalTripMinutes,
    (SUM(fft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(fft.TripDurationMinutes), 0)) AS IdlePercentage
FROM FactFleetTrips fft
INNER JOIN DimGeography dg ON fft.RouteKey = dg.GeographyKey
INNER JOIN DimTime dt ON fft.DepartureDateKey = dt.TimeKey
WHERE dt.CalendarMonth = '2026-06'
GROUP BY dg.RouteName
HAVING (SUM(fft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(fft.TripDurationMinutes), 0)) > 15
ORDER BY IdlePercentage DESC;
```

**FactCrossDock** — Transfers without long-term storage:
```sql
-- Example: Cross-dock efficiency by warehouse
SELECT 
    dw.WarehouseName,
    COUNT(*) AS CrossDockOperations,
    AVG(fcd.TransferDurationMinutes) AS AvgTransferMinutes,
    SUM(CASE WHEN fcd.TransferDurationMinutes > 120 THEN 1 ELSE 0 END) AS OperationsOver2Hours
FROM FactCrossDock fcd
INNER JOIN DimWarehouse dw ON fcd.WarehouseKey = dw.WarehouseKey
INNER JOIN DimTime dt ON fcd.InboundTimeKey = dt.TimeKey
WHERE dt.FiscalYear = 2026
GROUP BY dw.WarehouseName
ORDER BY AvgTransferMinutes;
```

### Key Dimension Tables

**DimTime** — Time intelligence with 15-minute granularity:
```sql
-- Example: Analyze operations by hour-of-day and day-of-week
SELECT 
    dt.HourOfDay,
    dt.DayOfWeek,
    COUNT(fwo.OperationKey) AS OperationCount,
    AVG(fwo.PickTimeSeconds) AS AvgPickTime
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.FiscalYear = 2026
GROUP BY dt.HourOfDay, dt.DayOfWeek
ORDER BY dt.DayOfWeek, dt.HourOfDay;
```

**DimProductGravity** — Warehouse spatial optimization:
```sql
-- Example: Products requiring gravity zone reassignment
SELECT 
    ProductSKU,
    ProductName,
    CurrentGravityZone,
    GravityScore,
    LastMovementDate,
    CASE 
        WHEN GravityScore > 80 AND CurrentGravityZone NOT IN ('Zone-A', 'Zone-B') 
        THEN 'Reassign to High-Gravity Zone'
        WHEN GravityScore < 30 AND CurrentGravityZone IN ('Zone-A', 'Zone-B')
        THEN 'Reassign to Low-Gravity Zone'
        ELSE 'No Change Needed'
    END AS RecommendedAction
FROM DimProductGravity
WHERE IsActive = 1
ORDER BY GravityScore DESC;
```

**DimSupplierReliability** — Supplier performance metrics:
```sql
-- Example: Flag unreliable suppliers affecting operations
SELECT 
    SupplierName,
    AvgLeadTimeDays,
    LeadTimeVarianceDays,
    DefectPercentage,
    ComplianceScore,
    CASE 
        WHEN ComplianceScore < 70 THEN 'Critical'
        WHEN ComplianceScore < 85 THEN 'Watch'
        ELSE 'Good Standing'
    END AS RiskCategory
FROM DimSupplierReliability
WHERE IsActive = 1
ORDER BY ComplianceScore ASC;
```

## Data Integration Patterns

### Incremental Loading with Stored Procedures

The template includes stored procedures for ETL operations:

```sql
-- Incremental load for warehouse operations
EXEC sp_LoadWarehouseOperations 
    @LastLoadDateTime = '2026-07-01 00:00:00',
    @CurrentDateTime = '2026-07-05 23:59:59';

-- Incremental load for fleet trips
EXEC sp_LoadFleetTrips 
    @LastLoadDateTime = '2026-07-01 00:00:00',
    @CurrentDateTime = '2026-07-05 23:59:59';

-- Refresh materialized views for performance
EXEC sp_RefreshAggregates;
```

### External Data Source Integration

**Using PolyBase for streaming telemetry**:
```sql
-- Create external data source (configure once)
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = REST,
    LOCATION = 'https://your-telemetry-api.example.com',
    CREDENTIAL = TelemetryAPICredential -- Created separately
);

-- Create external table
CREATE EXTERNAL TABLE ext_FleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = TelemetryAPI,
    LOCATION = '/api/telemetry',
    FILE_FORMAT = JSONFormat
);

-- Query and insert into fact table
INSERT INTO FactFleetTrips (VehicleKey, DepartureDateKey, Latitude, Longitude, Speed, FuelLevel)
SELECT 
    dv.VehicleKey,
    dt.TimeKey,
    et.Latitude,
    et.Longitude,
    et.Speed,
    et.FuelLevel
FROM ext_FleetTelemetry et
INNER JOIN DimVehicle dv ON et.VehicleID = dv.VehicleID
INNER JOIN DimTime dt ON CAST(et.Timestamp AS DATE) = dt.CalendarDate
WHERE et.Timestamp > (SELECT MAX(LastLoadDateTime) FROM ETLControl WHERE TableName = 'FactFleetTrips');
```

## Power BI Configuration

### Connection Setup

When opening `LogiFleet_Pulse_Master.pbit`:

1. **Enter SQL Server parameters**:
   - Server: `YOUR_SQL_SERVER_NAME`
   - Database: `YOUR_DATABASE_NAME`
   - Data Connectivity mode: `DirectQuery` (for real-time) or `Import` (for performance)

2. **Configure refresh schedule** (if using Import mode):
   - File → Options and settings → Data source settings
   - Scheduled refresh: Every 15 minutes (requires Power BI Service/Gateway)

### Custom DAX Measures

**Cross-Fact KPI Example — Dwell Cost per Route**:
```dax
DwellCostPerRoute = 
VAR DwellHours = 
    CALCULATE(
        SUM(FactWarehouseOperations[DwellTimeHours]),
        USERELATIONSHIP(FactWarehouseOperations[ProductKey], DimProduct[ProductKey])
    )
VAR RouteFuelCost = 
    CALCULATE(
        SUM(FactFleetTrips[FuelCostUSD]),
        USERELATIONSHIP(FactFleetTrips[RouteKey], DimGeography[GeographyKey])
    )
RETURN
    DIVIDE(DwellHours * 2.5, RouteFuelCost, 0) -- 2.5 = hourly warehouse cost assumption
```

**Predictive Bottleneck Index**:
```dax
BottleneckIndex = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR StdDevDwell = STDEV.P(FactWarehouseOperations[DwellTimeHours])
VAR CurrentDwell = SUM(FactWarehouseOperations[DwellTimeHours])
RETURN
    IF(
        CurrentDwell > (AvgDwellTime + (2 * StdDevDwell)),
        "High Risk",
        IF(
            CurrentDwell > (AvgDwellTime + StdDevDwell),
            "Moderate Risk",
            "Normal"
        )
    )
```

### Role-Level Security (RLS)

**Configure RLS in Power BI**:
```dax
-- Create role: "Regional Manager - West"
-- Apply filter on DimGeography table:
[Region] = "West"

-- Create role: "Warehouse Supervisor - Site A"
-- Apply filter on DimWarehouse table:
[WarehouseCode] = "WHR-A"
```

Test RLS in Power BI Desktop:
- Modeling tab → Manage roles → View as role

## Alerting and Automation

### SQL Server Agent Jobs for Automated Alerts

**Create alert for excessive fleet idle time**:
```sql
-- Stored procedure to check thresholds
CREATE PROCEDURE sp_CheckFleetIdleAlerts
AS
BEGIN
    DECLARE @AlertThresholdPct DECIMAL(5,2) = 15.0;
    DECLARE @EmailRecipients VARCHAR(500) = 'fleet-ops@example.com';
    
    -- Identify routes exceeding threshold
    DECLARE @AlertTable TABLE (
        RouteName VARCHAR(100),
        IdlePercentage DECIMAL(5,2),
        TotalIdleMinutes INT
    );
    
    INSERT INTO @AlertTable
    SELECT TOP 10
        dg.RouteName,
        (SUM(fft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(fft.TripDurationMinutes), 0)) AS IdlePercentage,
        SUM(fft.IdleTimeMinutes) AS TotalIdleMinutes
    FROM FactFleetTrips fft
    INNER JOIN DimGeography dg ON fft.RouteKey = dg.GeographyKey
    INNER JOIN DimTime dt ON fft.DepartureDateKey = dt.TimeKey
    WHERE dt.CalendarDate = CAST(GETDATE() AS DATE)
    GROUP BY dg.RouteName
    HAVING (SUM(fft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(fft.TripDurationMinutes), 0)) > @AlertThresholdPct
    ORDER BY IdlePercentage DESC;
    
    -- Send email if alerts exist
    IF EXISTS (SELECT 1 FROM @AlertTable)
    BEGIN
        DECLARE @EmailBody NVARCHAR(MAX);
        SET @EmailBody = 'Fleet Idle Time Alert - Routes exceeding ' + CAST(@AlertThresholdPct AS VARCHAR) + '% idle time:<br><br>';
        
        SELECT @EmailBody = @EmailBody + 
            'Route: ' + RouteName + 
            ' | Idle %: ' + CAST(IdlePercentage AS VARCHAR) + 
            ' | Total Idle Minutes: ' + CAST(TotalIdleMinutes AS VARCHAR) + '<br>'
        FROM @AlertTable;
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @EmailRecipients,
            @subject = 'Fleet Idle Time Alert',
            @body = @EmailBody,
            @body_format = 'HTML';
    END
END;
GO

-- Schedule via SQL Server Agent (run every hour)
-- Job Step: EXEC sp_CheckFleetIdleAlerts;
```

## Common Patterns and Use Cases

### Pattern 1: Root Cause Analysis — Delayed Shipments

```sql
-- Link warehouse dwell time to fleet delays
WITH WarehouseDwell AS (
    SELECT 
        fwo.ShipmentID,
        AVG(fwo.DwellTimeHours) AS AvgDwellHours,
        dp.ProductCategory
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
    WHERE fwo.ShipmentID IS NOT NULL
    GROUP BY fwo.ShipmentID, dp.ProductCategory
),
FleetDelays AS (
    SELECT 
        fft.ShipmentID,
        SUM(fft.DelayMinutes) AS TotalDelayMinutes,
        MAX(fft.DelayReason) AS PrimaryDelayReason
    FROM FactFleetTrips fft
    WHERE fft.ShipmentID IS NOT NULL
    GROUP BY fft.ShipmentID
)
SELECT 
    wd.ShipmentID,
    wd.ProductCategory,
    wd.AvgDwellHours,
    fd.TotalDelayMinutes,
    fd.PrimaryDelayReason,
    CASE 
        WHEN wd.AvgDwellHours > 48 AND fd.TotalDelayMinutes > 60 
        THEN 'High Priority - Dual Issue'
        WHEN wd.AvgDwellHours > 48 
        THEN 'Warehouse Optimization Needed'
        WHEN fd.TotalDelayMinutes > 60 
        THEN 'Route Optimization Needed'
        ELSE 'Normal'
    END AS ActionCategory
FROM WarehouseDwell wd
INNER JOIN FleetDelays fd ON wd.ShipmentID = fd.ShipmentID
WHERE wd.AvgDwellHours > 24 OR fd.TotalDelayMinutes > 30
ORDER BY wd.AvgDwellHours DESC, fd.TotalDelayMinutes DESC;
```

### Pattern 2: Gravity Zone Optimization

```sql
-- Recommend gravity zone reassignments based on 90-day velocity
WITH ProductVelocity AS (
    SELECT 
        fwo.ProductKey,
        COUNT(*) AS PickCount,
        AVG(fwo.PickTimeSeconds) AS AvgPickTime,
        SUM(fwo.ItemValue) AS TotalValue
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.CalendarDate >= DATEADD(DAY, -90, GETDATE())
    GROUP BY fwo.ProductKey
)
SELECT 
    dp.ProductSKU,
    dp.ProductName,
    dpg.CurrentGravityZone,
    dpg.GravityScore AS CurrentGravityScore,
    pv.PickCount,
    pv.TotalValue,
    CASE 
        WHEN pv.PickCount > 500 AND pv.TotalValue > 50000 THEN 'Zone-A (High Gravity)'
        WHEN pv.PickCount > 200 OR pv.TotalValue > 20000 THEN 'Zone-B (Medium Gravity)'
        ELSE 'Zone-C (Low Gravity)'
    END AS RecommendedZone,
    CASE 
        WHEN dpg.CurrentGravityZone <> (
            CASE 
                WHEN pv.PickCount > 500 AND pv.TotalValue > 50000 THEN 'Zone-A'
                WHEN pv.PickCount > 200 OR pv.TotalValue > 20000 THEN 'Zone-B'
                ELSE 'Zone-C'
            END
        ) THEN 'Reassignment Recommended'
        ELSE 'No Change'
    END AS ActionNeeded
FROM ProductVelocity pv
INNER JOIN DimProduct dp ON pv.ProductKey = dp.ProductKey
INNER JOIN DimProductGravity dpg ON dp.ProductKey = dpg.ProductKey
WHERE dpg.IsActive = 1
ORDER BY pv.TotalValue DESC;
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- Simulate impact of warehouse capacity change on fleet utilization
DECLARE @CapacityIncreasePct DECIMAL(5,2) = 15.0; -- 15% capacity increase scenario

WITH BaselineMetrics AS (
    SELECT 
        AVG(fwo.DwellTimeHours) AS BaselineDwellHours,
        COUNT(*) AS BaselineOperations
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.FiscalYear = 2026 AND dt.FiscalQuarter = 1
),
ProjectedImpact AS (
    SELECT 
        BaselineDwellHours * (1 - (@CapacityIncreasePct / 100)) AS ProjectedDwellHours,
        BaselineOperations * (1 + (@CapacityIncreasePct / 100)) AS ProjectedOperations
    FROM BaselineMetrics
),
FleetBaseline AS (
    SELECT 
        AVG(fft.TripDurationMinutes) AS BaselineTripDuration,
        COUNT(*) AS BaselineTrips
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.DepartureDateKey = dt.TimeKey
    WHERE dt.FiscalYear = 2026 AND dt.FiscalQuarter = 1
)
SELECT 
    bm.BaselineDwellHours,
    pi.ProjectedDwellHours,
    (bm.BaselineDwellHours - pi.ProjectedDwellHours) AS DwellReductionHours,
    fb.BaselineTripDuration,
    fb.BaselineTripDuration * 0.92 AS ProjectedTripDuration, -- Estimated 8% reduction
    fb.BaselineTrips * 1.15 AS ProjectedTripsNeeded,
    (fb.BaselineTrips * 1.15) - fb.BaselineTrips AS AdditionalTripsRequired
FROM BaselineMetrics bm
CROSS JOIN ProjectedImpact pi
CROSS JOIN FleetBaseline fb;
```

## Troubleshooting

### Issue: Power BI Refresh Fails with Timeout

**Symptoms**: Import mode refresh exceeds 30-minute limit

**Solutions**:
1. **Switch to DirectQuery mode** for real-time dashboards without data import
2. **Partition fact tables** by date range:
```sql
-- Create partition function (SQL Server Enterprise only)
CREATE PARTITION FUNCTION pf_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401);

CREATE PARTITION SCHEME ps_TimeKey
AS PARTITION pf_TimeKey
ALL TO ([PRIMARY]);

-- Recreate fact table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey INT IDENTITY(1,1),
    TimeKey INT,
    -- ... other columns
) ON ps_TimeKey(TimeKey);
```
3. **Use incremental refresh** in Power BI (Premium feature):
   - Table tools → Incremental refresh settings
   - Archive data older than 2 years, refresh last 90 days

### Issue: Cross-Fact Queries Return Incorrect Results

**Symptoms**: Mismatched totals when joining FactWarehouseOperations and FactFleetTrips

**Cause**: Many-to-many relationships without proper bridge table

**Solution**: Use the provided bridge table or DAX CROSSFILTER:
```dax
TotalShipmentsWithFleetData = 
CALCULATE(
    DISTINCTCOUNT(FactWarehouseOperations[ShipmentID]),
    CROSSFILTER(FactWarehouseOperations[ShipmentID], FactFleetTrips[ShipmentID], Both)
)
```

### Issue: Row-Level Security Not Filtering Data

**Symptoms**: Users see data outside their assigned region/warehouse

**Checklist**:
1. **Verify role assignment** in Power BI Service (Workspace → Security)
2. **Test locally**: Modeling → View as → Select role
3. **Check filter propagation**:
```dax
-- Ensure filters flow from dimension to fact
-- In Model view, verify relationship direction
DimGeography → FactFleetTrips (single direction)
DimWarehouse → FactWarehouseOperations (single direction)
```
4. **Validate DAX filter syntax**:
```dax
-- Role: Regional Manager - West
-- Filter on DimGeography:
[Region] IN {"West", "Northwest"} -- Use IN for multiple values
```

### Issue: Slow Query Performance on Large Fact Tables

**Symptoms**: Queries taking 30+ seconds

**Optimizations**:
1. **Create covering indexes**:
```sql
-- Index for time-based queries
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeKey_ProductKey
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (DwellTimeHours, PickTimeSeconds, ItemValue);

-- Index for shipment tracking
CREATE NONCLUSTERED INDEX IX_FactFleet_ShipmentID
ON FactFleetTrips (ShipmentID)
INCLUDE (DelayMinutes, FuelCostUSD, TripDurationMinutes);
```

2. **Use columnstore indexes** for analytical queries (SQL Server 2016+):
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (TimeKey, ProductKey, DwellTimeHours, PickTimeSeconds);
```

3. **Create aggregated views**:
```sql
CREATE VIEW vw_DailyWarehouseSummary
WITH SCHEMABINDING
AS
SELECT 
    fwo.TimeKey,
    fwo.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    AVG(fwo.DwellTimeHours) AS AvgDwellHours,
    SUM(fwo.ItemValue) AS TotalValue
FROM dbo.FactWarehouseOperations fwo
GROUP BY fwo.TimeKey, fwo.ProductKey;

-- Create index on view for materialization
CREATE UNIQUE CLUSTERED INDEX UCI_DailyWarehouseSummary
ON vw_DailyWarehouseSummary (TimeKey, ProductKey);
```

## Advanced Configuration

### Integrating External Weather Data

```sql
-- Create external table for weather API (example using Azure)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://YOUR_STORAGE_ACCOUNT.blob.core.windows.net/weather',
    CREDENTIAL = AzureStorageCredential
);

CREATE EXTERNAL TABLE ext_WeatherData (
    LocationID VARCHAR(50),
    ObservationDate DATE,
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    WindSpeed DECIMAL(5,2),
    Conditions VARCHAR(100)
)
WITH (
    DATA_SOURCE = WeatherAPI,
    FILE_FORMAT = CSVFormat,
    LOCATION = '/daily/'
);

-- Enrich fleet trips with weather context
SELECT 
    fft.TripID,
    fft.RouteName,
    fft.DelayMinutes,
    ew.Conditions AS WeatherConditions,
    ew.Precipitation,
    CASE 
        WHEN ew.Precipitation > 10 AND fft.DelayMinutes > 30 
        THEN 'Weather-Related Delay'
        ELSE 'Other Cause'
    END AS DelayClassification
FROM FactFleetTrips fft
INNER JOIN DimGeography dg ON fft.RouteKey = dg.GeographyKey
INNER JOIN ext_WeatherData ew ON dg.LocationID = ew.LocationID 
    AND CAST(fft.DepartureDateTime AS DATE) = ew.ObservationDate
WHERE fft.DelayMinutes > 15;
```

### Implementing Change Data Capture (CDC)

```sql
-- Enable CDC on source tables (requires SQL Server Standard+)
EXEC sys.sp_cdc_enable_db;

EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'FactWarehouseOperations',
    @role_name = NULL,
    @supports_net_changes = 1;

-- Query CDC changes
DECLARE @LastSync DATETIME = '2026-07-05 12:00:00';
DECLARE @CurrentSync DATETIME = GETDATE();

SELECT *
FROM cdc.fn_cdc_get_net_changes_dbo_FactWarehouseOperations(
    sys.fn_cdc_map_time_to_lsn('smallest greater than', @LastSync),
    sys.fn_cdc_map_time_to_lsn('largest less than or equal', @CurrentSync),
    'all'
);
```

## Environment Variables Reference

When deploying in production, use environment variables for sensitive configuration:

- `SQL_SERVER_NAME`: SQL Server instance name
- `SQL_DATABASE_NAME`: Target database name
- `SQL_USERNAME`: Database authentication username
- `SQL_PASSWORD`: Database authentication password (use Azure Key Vault in production)
- `PBI_WORKSPACE_ID`: Power BI workspace GUID (for REST API automation)
- `PBI_REFRESH_TOKEN`: OAuth token for scheduled refresh automation
- `TELEMETRY_API_KEY`: Fleet telemetry API authentication key
- `WEATHER_API_KEY`: External weather service API key
- `SMTP_SERVER`: Email server for alerting
- `ALERT_EMAIL_RECIPIENTS`: Comma-separated email list for critical alerts

Example connection string using environment variables (in ETL script):
```python
import os
import pyodbc

conn_str = (
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER_NAME')};"
    f"DATABASE={os.getenv('SQL_DATABASE_NAME')};"
    f"UID={os.getenv('SQL_USERNAME')};"
    f"PWD={os.getenv('SQL_PASSWORD')}"
)

conn = pyodbc.connect(conn_str)
```

## Key Takeaways

- **LogiFleet Pulse is a template**, not a SaaS product — you own and customize the schema
- **Multi-fact architecture** requires careful relationship design; use the provided bridge tables
- **DirectQuery vs. Import**: Use DirectQuery for real-time, Import for performance (with partitioning)
- **Row-Level Security**: Essential for multi-tenant deployments; test thoroughly before production
- **Incremental loading**: Always use `sp_Load*` stored procedures to avoid full table scans
- **Indexing strategy**: Columnstore for analytics, B-tree for transactional lookups
- **External data sources**: PolyBase simplifies integration but requires proper authentication

This skill equips AI agents to guide developers through schema deployment, data integration, Power BI configuration, and performance optimization for LogiFleet Pulse implementations.
