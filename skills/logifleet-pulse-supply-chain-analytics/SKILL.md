---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up LogiFleet Pulse analytics
  - deploy supply chain data warehouse
  - configure Power BI logistics dashboard
  - implement warehouse gravity zones
  - create fleet analytics star schema
  - build cross-modal logistics intelligence
  - integrate WMS with Power BI
  - optimize warehouse and fleet KPIs
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced data warehousing and analytics template for supply chain, logistics, and fleet management. It provides a complete multi-fact star schema implementation in MS SQL Server paired with Power BI dashboards for real-time operational intelligence.

**Key Capabilities:**
- Multi-fact star schema linking warehouse operations, fleet trips, and cross-dock activities
- Time-phased dimensions with 15-minute granularity
- Warehouse Gravity Zones™ for spatial optimization
- Cross-fact KPI harmonization (inventory vs. fleet metrics)
- Predictive bottleneck detection
- Role-based access with row-level security

**Primary Language:** SQL (T-SQL) with Power BI integration

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, GPS/telematics, ERP)

### Deployment Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
:r ./sql/01_create_database.sql
:r ./sql/02_create_dimensions.sql
:r ./sql/03_create_facts.sql
:r ./sql/04_create_views.sql
:r ./sql/05_create_stored_procedures.sql
```

3. **Configure data sources:**
```json
// config.json (create from config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_connection_string": "${WMS_CONN_STRING}",
    "telematics_api_url": "${TELEMATICS_API_URL}",
    "telematics_api_key": "${TELEMATICS_API_KEY}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Import Power BI template:**
- Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
- Enter SQL Server connection details when prompted
- Publish to Power BI Service (optional)

## Core Schema Components

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:
```sql
-- Example: Create time dimension for 2026
EXEC sp_PopulateTimeDimension 
    @StartDate = '2026-01-01', 
    @EndDate = '2026-12-31',
    @GranularityMinutes = 15;

-- Query pattern
SELECT 
    TimeKey,
    FullDateTime,
    HourOfDay,
    DayOfWeek,
    FiscalPeriod,
    IsBusinessHour
FROM DimTime
WHERE Date = '2026-07-05';
```

**DimProductGravity** - Product classification with gravity scoring:
```sql
-- Calculate gravity score for SKUs
UPDATE DimProductGravity
SET GravityScore = (
    (PickFrequencyPerDay * 0.4) +
    (ItemValue / 100 * 0.3) +
    (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) +
    (100 / NULLIF(ReplenishmentLeadTimeHours, 0) * 0.3)
)
WHERE IsActive = 1;

-- High-gravity products (should be closest to shipping)
SELECT TOP 20
    SKU,
    ProductName,
    GravityScore,
    RecommendedZone
FROM DimProductGravity
ORDER BY GravityScore DESC;
```

**DimGeography** - Hierarchical location dimension:
```sql
-- Geographic hierarchy query
SELECT 
    g.GeographyKey,
    g.WarehouseCode,
    g.City,
    g.Region,
    g.Country,
    g.Continent,
    g.Latitude,
    g.Longitude
FROM DimGeography g
WHERE g.LocationType = 'Warehouse'
    AND g.IsActive = 1;
```

### Fact Tables

**FactWarehouseOperations** - Core warehouse metrics:
```sql
-- Insert warehouse operation record
INSERT INTO FactWarehouseOperations (
    TimeKey,
    GeographyKey,
    ProductKey,
    OperationType,
    Quantity,
    DwellTimeMinutes,
    PickTimeSeconds,
    PackTimeSeconds,
    ZoneFrom,
    ZoneTo
)
VALUES (
    20260705_1030, -- TimeKey for 2026-07-05 10:30
    101, -- GeographyKey for Warehouse A
    5432, -- ProductKey for SKU
    'PICK',
    50,
    180, -- 3 hours dwell
    120, -- 2 minutes pick
    90, -- 1.5 minutes pack
    'A-12',
    'SHIP-DOCK-1'
);

-- Dwell time analysis by zone
SELECT 
    wo.ZoneFrom,
    pg.ProductCategory,
    AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
    COUNT(*) AS Operations,
    SUM(wo.Quantity) AS TotalUnits
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
GROUP BY wo.ZoneFrom, pg.ProductCategory
HAVING AVG(wo.DwellTimeMinutes) > 240 -- Over 4 hours
ORDER BY AvgDwellMinutes DESC;
```

**FactFleetTrips** - Fleet telemetry and route data:
```sql
-- Insert fleet trip record
INSERT INTO FactFleetTrips (
    TimeKey,
    VehicleKey,
    RouteKey,
    DriverKey,
    TripSegmentNumber,
    DistanceKM,
    FuelConsumedLiters,
    IdleTimeMinutes,
    LoadWeightKG,
    EventType
)
VALUES (
    20260705_1400,
    25, -- VehicleKey
    88, -- RouteKey
    42, -- DriverKey
    3,
    45.2,
    12.5,
    18,
    3500,
    'DELIVERY_COMPLETE'
);

-- Fleet efficiency by route
SELECT 
    r.RouteName,
    v.VehicleID,
    COUNT(DISTINCT ft.TripSegmentNumber) AS Segments,
    SUM(ft.DistanceKM) AS TotalKM,
    SUM(ft.FuelConsumedLiters) AS TotalFuel,
    SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
    (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.DistanceKM) / 60.0, 0)) AS IdlePercentage
FROM FactFleetTrips ft
INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY r.RouteName, v.VehicleID
HAVING (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.DistanceKM) / 60.0, 0)) > 15
ORDER BY IdlePercentage DESC;
```

**FactCrossDock** - Cross-docking operations:
```sql
-- Cross-dock transfer tracking
INSERT INTO FactCrossDock (
    TimeKey,
    GeographyKey,
    ProductKey,
    InboundShipmentID,
    OutboundShipmentID,
    Quantity,
    TransferTimeMinutes,
    QualityCheckStatus
)
VALUES (
    20260705_0900,
    101,
    5432,
    'IB-2026-071234',
    'OB-2026-075678',
    100,
    45,
    'PASSED'
);

-- Cross-dock efficiency analysis
SELECT 
    g.WarehouseCode,
    pg.ProductCategory,
    COUNT(*) AS CrossDockTransfers,
    AVG(cd.TransferTimeMinutes) AS AvgTransferMinutes,
    SUM(CASE WHEN cd.TransferTimeMinutes <= 60 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS PercentUnder1Hour
FROM FactCrossDock cd
INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity pg ON cd.ProductKey = pg.ProductKey
INNER JOIN DimTime t ON cd.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -14, GETDATE())
GROUP BY g.WarehouseCode, pg.ProductCategory
ORDER BY AvgTransferMinutes DESC;
```

## Cross-Fact KPI Queries

### Inventory Turnover vs. Fleet Utilization
```sql
-- Correlate warehouse velocity with fleet routing efficiency
WITH WarehouseVelocity AS (
    SELECT 
        wo.GeographyKey,
        pg.ProductCategory,
        SUM(wo.Quantity) AS UnitsShipped,
        AVG(wo.DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
        AND wo.OperationType = 'SHIP'
    GROUP BY wo.GeographyKey, pg.ProductCategory
),
FleetPerformance AS (
    SELECT 
        r.OriginGeographyKey AS GeographyKey,
        COUNT(*) AS TripCount,
        AVG(ft.IdleTimeMinutes) AS AvgIdle,
        SUM(ft.FuelConsumedLiters) AS TotalFuel
    FROM FactFleetTrips ft
    INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY r.OriginGeographyKey
)
SELECT 
    g.WarehouseCode,
    wv.ProductCategory,
    wv.UnitsShipped,
    wv.AvgDwell,
    fp.TripCount,
    fp.AvgIdle,
    fp.TotalFuel,
    (wv.UnitsShipped / NULLIF(fp.TripCount, 0)) AS UnitsPerTrip,
    (fp.TotalFuel / NULLIF(wv.UnitsShipped, 0)) AS FuelPerUnit
FROM WarehouseVelocity wv
INNER JOIN FleetPerformance fp ON wv.GeographyKey = fp.GeographyKey
INNER JOIN DimGeography g ON wv.GeographyKey = g.GeographyKey
ORDER BY FuelPerUnit DESC;
```

### Warehouse Gravity Zone Optimization
```sql
-- Identify SKUs in suboptimal zones based on gravity score
SELECT 
    pg.SKU,
    pg.ProductName,
    pg.GravityScore,
    pg.RecommendedZone,
    wo.ZoneFrom AS CurrentZone,
    AVG(wo.PickTimeSeconds) AS AvgPickTime,
    COUNT(*) AS PickOperations,
    CASE 
        WHEN pg.RecommendedZone <> wo.ZoneFrom THEN 'RELOCATE'
        ELSE 'OPTIMAL'
    END AS Action
FROM DimProductGravity pg
INNER JOIN FactWarehouseOperations wo ON pg.ProductKey = wo.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
    AND wo.OperationType = 'PICK'
GROUP BY pg.SKU, pg.ProductName, pg.GravityScore, pg.RecommendedZone, wo.ZoneFrom
HAVING COUNT(*) > 10
ORDER BY GravityScore DESC, AvgPickTime DESC;
```

## Stored Procedures

### Incremental Data Loading
```sql
-- Load warehouse operations from staging
EXEC sp_LoadWarehouseOperations 
    @LoadType = 'INCREMENTAL',
    @StartDateTime = '2026-07-05 00:00:00',
    @EndDateTime = '2026-07-05 23:59:59';

-- Load fleet trips from telemetry API
EXEC sp_LoadFleetTrips
    @LoadType = 'INCREMENTAL',
    @VehicleFilter = NULL; -- NULL loads all vehicles

-- Full refresh of product gravity scores
EXEC sp_RecalculateProductGravity
    @RecalculateAll = 1;
```

### Alert Generation
```sql
-- Configure threshold-based alerts
EXEC sp_GenerateLogisticsAlerts
    @AlertTypes = 'DWELL_TIME,IDLE_TIME,FUEL_ANOMALY',
    @NotificationMethod = 'EMAIL',
    @RecipientList = '${ALERT_EMAIL_LIST}';

-- Custom alert example: High dwell time for perishables
CREATE PROCEDURE sp_AlertPerishableDwellTime
AS
BEGIN
    DECLARE @AlertThresholdMinutes INT = 120;
    
    INSERT INTO AlertQueue (
        AlertType,
        Severity,
        Message,
        AffectedSKU,
        AffectedWarehouse,
        CreatedAt
    )
    SELECT 
        'PERISHABLE_DWELL',
        'HIGH',
        CONCAT('SKU ', pg.SKU, ' in ', g.WarehouseCode, ' has dwell time of ', 
               AVG(wo.DwellTimeMinutes), ' minutes'),
        pg.SKU,
        g.WarehouseCode,
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE pg.IsPerishable = 1
        AND t.Date = CAST(GETDATE() AS DATE)
        AND wo.OperationType IN ('RECEIVE', 'PUTAWAY')
    GROUP BY pg.SKU, g.WarehouseCode
    HAVING AVG(wo.DwellTimeMinutes) > @AlertThresholdMinutes;
END;
```

## Power BI Integration

### DAX Measures

**Key Warehouse Metrics:**
```dax
// Total Dwell Time (Hours)
TotalDwellTimeHours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Average Pick Efficiency (Units per Hour)
PickEfficiencyUnitsPerHour = 
DIVIDE(
    SUM(FactWarehouseOperations[Quantity]),
    SUM(FactWarehouseOperations[PickTimeSeconds]) / 3600,
    0
)

// Warehouse Gravity Zone Compliance %
GravityZoneCompliancePercent = 
VAR TotalPicks = COUNTROWS(FactWarehouseOperations)
VAR OptimalPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[ZoneFrom] = RELATED(DimProductGravity[RecommendedZone])
    )
RETURN
    DIVIDE(OptimalPicks, TotalPicks, 0) * 100
```

**Fleet KPIs:**
```dax
// Fleet Idle Percentage
FleetIdlePercentage = 
VAR TotalTime = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[DistanceKM] / 60  // Assume 60 km/h average
    ) * 60  // Convert to minutes
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(IdleTime, TotalTime + IdleTime, 0) * 100

// Fuel Efficiency (KM per Liter)
FuelEfficiencyKMPerLiter = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Predictive Maintenance Score
MaintenanceUrgencyScore = 
SUMX(
    FactFleetTrips,
    SWITCH(
        TRUE(),
        FactFleetTrips[IdleTimeMinutes] > 30, 10,
        FactFleetTrips[FuelConsumedLiters] / FactFleetTrips[DistanceKM] > 0.15, 8,
        5
    ) * RELATED(DimProductGravity[GravityScore]) / 100
)
```

**Cross-Fact Unified Metrics:**
```dax
// Revenue per Fuel Liter
RevenuePerFuelLiter = 
VAR TotalRevenue = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[Quantity] * RELATED(DimProductGravity[ItemValue])
    )
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedLiters])
RETURN
    DIVIDE(TotalRevenue, TotalFuel, 0)

// End-to-End Cycle Time (Receive to Deliver)
AvgCycleTimeHours = 
VAR ReceiveTime = 
    CALCULATE(
        MIN(DimTime[FullDateTime]),
        FactWarehouseOperations[OperationType] = "RECEIVE"
    )
VAR DeliverTime = 
    CALCULATE(
        MAX(DimTime[FullDateTime]),
        FactFleetTrips[EventType] = "DELIVERY_COMPLETE"
    )
RETURN
    DATEDIFF(ReceiveTime, DeliverTime, HOUR)
```

### Connecting Power BI Template

```powershell
# PowerShell script to update data source in .pbit template
$pbixPath = "LogiFleet_Pulse_Master.pbix"
$sqlServer = $env:SQL_SERVER_HOST
$database = "LogiFleetPulse"

# Update data source connection (requires PBI cmdlets)
Set-PowerBIDataSourceConnection `
    -ReportPath $pbixPath `
    -DataSourceType SQL `
    -Server $sqlServer `
    -Database $database
```

## Row-Level Security (RLS)

### Configure Role-Based Access
```sql
-- Create security dimension
CREATE TABLE DimUserSecurity (
    UserKey INT PRIMARY KEY IDENTITY(1,1),
    UserEmail NVARCHAR(255) UNIQUE NOT NULL,
    RoleName NVARCHAR(50) NOT NULL,
    AllowedWarehouses NVARCHAR(MAX), -- JSON array of warehouse codes
    AllowedRegions NVARCHAR(MAX),    -- JSON array of regions
    IsActive BIT DEFAULT 1,
    CreatedAt DATETIME2 DEFAULT GETDATE()
);

-- Insert example users
INSERT INTO DimUserSecurity (UserEmail, RoleName, AllowedWarehouses, AllowedRegions)
VALUES 
    ('manager@company.com', 'REGIONAL_MANAGER', '["WH-001","WH-002"]', '["WEST"]'),
    ('analyst@company.com', 'ANALYST', NULL, NULL), -- Full access
    ('driver@company.com', 'DRIVER', NULL, '["WEST"]');

-- Create RLS function
CREATE FUNCTION fn_SecurityFilter(@UserEmail NVARCHAR(255))
RETURNS TABLE
AS
RETURN
(
    SELECT 
        g.GeographyKey
    FROM DimGeography g
    INNER JOIN DimUserSecurity u ON u.UserEmail = @UserEmail
    WHERE u.IsActive = 1
        AND (
            u.AllowedWarehouses IS NULL -- Full access
            OR g.WarehouseCode IN (SELECT value FROM OPENJSON(u.AllowedWarehouses))
            OR g.Region IN (SELECT value FROM OPENJSON(u.AllowedRegions))
        )
);

-- Apply security to fact tables
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT fk_SecurityFilter
CHECK (GeographyKey IN (SELECT GeographyKey FROM fn_SecurityFilter(USER_NAME())));
```

### Power BI RLS Configuration
```dax
// Create RLS role in Power BI (Model view > Manage Roles)
// Role: RegionalAccess
[GeographyKey] IN 
    CALCULATETABLE(
        VALUES(DimUserSecurity[AllowedGeographyKey]),
        FILTER(
            DimUserSecurity,
            DimUserSecurity[UserEmail] = USERPRINCIPALNAME()
        )
    )
```

## Troubleshooting

### Performance Optimization

**Slow cross-fact queries:**
```sql
-- Create covering indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Geography_Product
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey)
INCLUDE (Quantity, DwellTimeMinutes, PickTimeSeconds);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Vehicle_Route
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey)
INCLUDE (DistanceKM, FuelConsumedLiters, IdleTimeMinutes);

-- Partition fact tables by month
ALTER PARTITION SCHEME ps_Monthly
NEXT USED [PRIMARY];

ALTER TABLE FactWarehouseOperations
SWITCH PARTITION 7 TO FactWarehouseOperations_Archive PARTITION 7;
```

**Power BI refresh timeout:**
```sql
-- Create aggregated views for Power BI
CREATE VIEW vw_WarehouseDaily AS
SELECT 
    CAST(t.Date AS DATE) AS Date,
    wo.GeographyKey,
    wo.ProductKey,
    SUM(wo.Quantity) AS TotalQuantity,
    AVG(wo.DwellTimeMinutes) AS AvgDwell,
    AVG(wo.PickTimeSeconds) AS AvgPickTime,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
GROUP BY CAST(t.Date AS DATE), wo.GeographyKey, wo.ProductKey;

-- Use incremental refresh in Power BI on RangeStart/RangeEnd parameters
```

### Data Quality Issues

**Missing time keys:**
```sql
-- Populate missing time records
DECLARE @MissingStart DATETIME2, @MissingEnd DATETIME2;

SELECT @MissingStart = MIN(FullDateTime), @MissingEnd = MAX(FullDateTime)
FROM (
    SELECT DISTINCT TimeKey FROM FactWarehouseOperations
    UNION
    SELECT DISTINCT TimeKey FROM FactFleetTrips
) fk
LEFT JOIN DimTime t ON fk.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL;

EXEC sp_PopulateTimeDimension @MissingStart, @MissingEnd, 15;
```

**Duplicate product records:**
```sql
-- Identify and merge duplicates
WITH DuplicateSKUs AS (
    SELECT SKU, COUNT(*) AS DupCount
    FROM DimProductGravity
    GROUP BY SKU
    HAVING COUNT(*) > 1
)
SELECT 
    pg.ProductKey,
    pg.SKU,
    pg.ProductName,
    ROW_NUMBER() OVER (PARTITION BY pg.SKU ORDER BY pg.CreatedAt) AS RowNum
FROM DimProductGravity pg
INNER JOIN DuplicateSKUs d ON pg.SKU = d.SKU;

-- Merge logic: Keep earliest record, update fact table references
```

### Alert System Not Triggering

**Check SQL Agent job:**
```sql
-- Verify alert job status
SELECT 
    j.name AS JobName,
    h.run_date,
    h.run_time,
    h.run_status,
    h.message
FROM msdb.dbo.sysjobs j
INNER JOIN msdb.dbo.sysjobhistory h ON j.job_id = h.job_id
WHERE j.name = 'LogiFleet_Alert_Generation'
ORDER BY h.run_date DESC, h.run_time DESC;

-- Manual execution
EXEC sp_GenerateLogisticsAlerts 
    @AlertTypes = 'ALL',
    @NotificationMethod = 'LOG'; -- Test mode
```

**Email notifications not sending:**
```sql
-- Configure Database Mail
EXEC msdb.dbo.sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC msdb.dbo.sp_configure 'Database Mail XPs', 1;
RECONFIGURE;

-- Test email
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'LogiFleet_Mail_Profile',
    @recipients = '${ALERT_EMAIL_TEST}',
    @subject = 'LogiFleet Test Alert',
    @body = 'Email system operational';
```

## Common Patterns

### Daily ETL Workflow
```sql
-- Scheduled job: Daily full load at 2 AM
BEGIN TRY
    BEGIN TRANSACTION;
    
    -- Load warehouse operations
    EXEC sp_LoadWarehouseOperations 
        @LoadType = 'INCREMENTAL',
        @StartDateTime = DATEADD(DAY, -1, CAST(GETDATE() AS DATE)),
        @EndDateTime = CAST(GETDATE() AS DATE);
    
    -- Load fleet trips
    EXEC sp_LoadFleetTrips
        @LoadType = 'INCREMENTAL',
        @VehicleFilter = NULL;
    
    -- Recalculate gravity scores weekly
    IF DATEPART(WEEKDAY, GETDATE()) = 1 -- Sunday
    BEGIN
        EXEC sp_RecalculateProductGravity @RecalculateAll = 1;
    END
    
    -- Generate alerts
    EXEC sp_GenerateLogisticsAlerts
        @AlertTypes = 'ALL',
        @NotificationMethod = 'EMAIL';
    
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    ROLLBACK TRANSACTION;
    -- Log error
    INSERT INTO ETLErrorLog (ErrorMessage, ErrorDate)
    VALUES (ERROR_MESSAGE(), GETDATE());
END CATCH
```

### API Integration (External Data)
```sql
-- Example: Load weather data affecting routes
CREATE PROCEDURE sp_LoadWeatherImpact
    @WeatherAPIKey NVARCHAR(255) = NULL
AS
BEGIN
    SET @WeatherAPIKey = ISNULL(@WeatherAPIKey, '${WEATHER_API_KEY}');
    
    -- Call external API (requires CLR or external script)
    -- Store results in staging table
    INSERT INTO StagingWeatherEvents (Date, Region, EventType, Severity)
    SELECT * FROM OPENROWSET(
        BULK 'https://api.weather.com/v1/events',
        FORMAT = 'JSON',
        HEADER = CONCAT('Authorization: Bearer ', @WeatherAPIKey)
    );
    
    -- Correlate with routes
    UPDATE FactFleetTrips
    SET WeatherImpactFlag = 1
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    INNER JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    INNER JOIN StagingWeatherEvents w 
        ON t.Date = w.Date 
        AND r.Region = w.Region
    WHERE w.Severity IN ('HIGH', 'SEVERE');
END
```

### Power BI Report Template Usage
```python
# Python script to automate Power BI deployment
import requests
import os

pbi_workspace_id = os.getenv('PBI_WORKSPACE_ID')
pbi_access_token = os.getenv('PBI_ACCESS_TOKEN')
report_path = 'LogiFleet_Pulse_Master.pbix'

# Upload report to workspace
url = f'https://api.powerbi.com/v1.0/myorg/groups/{pbi_workspace_id}/imports'
headers = {
    'Authorization': f'Bearer {pbi_access_token}'
}
files = {
    'file': open(report_path, 'rb')
}
response = requests.post(url, headers=headers, files=files)

print(f"Report uploaded: {response.json()}")
```

## Best Practices

1. **Time Dimension**: Always use TimeKey for joins, not raw timestamps
2. **Incremental Loads**: Use watermark columns to track last processed record
3. **Gravity Scores**: Recalculate weekly, not real-time (computationally expensive)
4. **Indexes**: Create filtered indexes for active records only
5. **Security**: Use environment variables for all credentials
6. **Power BI**: Use DirectQuery for real-time dashboards, Import for historical analysis
7. **Alerts**: Set thresholds based on percentile analysis, not arbitrary values

## Reference

- **SQL Scripts**: `./sql/` directory contains all DDL and stored procedures
- **Power BI Template**: `LogiFleet_Pulse_Master.pbit`
- **Sample Data**: `./sample_data/` for testing
- **Documentation**: Full schema ERD in `./docs/schema_diagram.png`
