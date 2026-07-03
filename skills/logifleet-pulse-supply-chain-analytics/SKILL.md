---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain KPI harmonization
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - deploy logistics data warehouse with Power BI
  - configure multi-fact star schema for warehouse and fleet
  - implement cross-modal supply chain analytics
  - build logistics intelligence dashboard
  - create warehouse gravity zone optimization
  - integrate fleet telemetry with warehouse operations
  - harmonize supply chain KPIs across facts
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What It Does

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for supply chain and logistics analytics. It implements a **multi-fact star schema** that harmonizes warehouse operations, fleet telemetry, inventory management, and external data sources (weather, traffic, supplier feeds) into a unified semantic layer. Key innovations include:

- **Cross-fact KPI linking** via composite shared dimensions (time, geography, product hierarchy)
- **Warehouse Gravity Zones** — spatial optimization based on pick frequency, value, and fragility
- **Adaptive Fleet Triage** — maintenance prioritization weighted by revenue impact
- **Temporal elasticity modeling** — time-phased simulation scenarios
- **15-minute granularity** time-aware dimensions for real-time operational intelligence

Primary language: **SQL** (T-SQL for MS SQL Server) with **Power BI** (.pbit template) for visualization.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- ODBC/OLEDB drivers for your WMS/TMS/telemetry sources
- (Optional) Azure Synapse Analytics for external big data enrichment

### Deployment Steps

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema**:
```sql
-- Connect to your SQL Server instance via SSMS or sqlcmd
-- Execute the schema creation script
:setvar DatabaseName "LogiFleetPulse"
:setvar DataPath "C:\SQLData\"
:setvar LogPath "C:\SQLLogs\"

USE master;
GO

-- Run the full schema deployment (typically in schema/deploy.sql)
:r schema\01_create_database.sql
:r schema\02_create_dimensions.sql
:r schema\03_create_facts.sql
:r schema\04_create_views.sql
:r schema\05_create_procedures.sql
:r schema\06_create_indexes.sql
```

3. **Configure data sources** — edit `config.json`:
```json
{
  "connections": {
    "wms_api": {
      "type": "REST",
      "base_url": "${WMS_API_URL}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "fleet_telemetry": {
      "type": "SQL_EXTERNAL",
      "server": "${TELEMETRY_SQL_SERVER}",
      "database": "${TELEMETRY_DB}",
      "username": "${TELEMETRY_USER}",
      "password": "${TELEMETRY_PASS}"
    },
    "weather_api": {
      "type": "REST",
      "endpoint": "https://api.weather.com/v3",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_ops": "15min",
    "fleet_trips": "15min",
    "external_feeds": "1hour"
  }
}
```

4. **Import Power BI template**:
   - Open `dashboards/LogiFleet_Pulse_Master.pbit` in Power BI Desktop
   - Enter SQL Server connection string when prompted
   - Verify relationships auto-detect correctly
   - Publish to Power BI Service (optional)

## Key SQL Objects

### Core Dimension Tables

**DimTime** — 15-minute granular time buckets:
```sql
SELECT TOP 100
    TimeKey,
    FullDateTime,
    HourOfDay,
    DayOfWeek,
    FiscalPeriod,
    IsWeekend,
    IsHoliday
FROM dbo.DimTime
ORDER BY TimeKey DESC;
```

**DimGeography** — hierarchical location dimension:
```sql
SELECT 
    GeographyKey,
    ContinentName,
    CountryName,
    RegionName,
    WarehouseCode,
    RouteNodeID,
    Latitude,
    Longitude
FROM dbo.DimGeography
WHERE CountryName = 'United States';
```

**DimProductGravity** — product classification with gravity scoring:
```sql
SELECT 
    ProductKey,
    SKU,
    ProductName,
    GravityScore,  -- Composite: (PickFrequency * 0.5) + (ValuePerUnit * 0.3) + (Fragility * 0.2)
    CurrentZone,
    RecommendedZone,
    LastRearrangementDate
FROM dbo.DimProductGravity
WHERE GravityScore > 75  -- High-gravity items
ORDER BY GravityScore DESC;
```

**DimSupplierReliability** — supplier performance tracking:
```sql
SELECT 
    SupplierKey,
    SupplierName,
    AverageLeadTimeDays,
    LeadTimeVariance,
    DefectRatePercent,
    ComplianceScore,
    ReliabilityTier  -- Calculated: Platinum, Gold, Silver, Bronze
FROM dbo.DimSupplierReliability
WHERE DefectRatePercent < 2.0;
```

### Core Fact Tables

**FactWarehouseOperations** — granular warehouse activity:
```sql
SELECT 
    wo.OperationKey,
    wo.TimeKey,
    wo.ProductKey,
    wo.GeographyKey,
    wo.OperationType,  -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    wo.DwellTimeMinutes,
    wo.PickRateUnitsPerHour,
    wo.PackingTimeSeconds,
    p.GravityScore,
    t.FullDateTime
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.DwellTimeMinutes > 72 * 60  -- Items in storage > 72 hours
ORDER BY wo.DwellTimeMinutes DESC;
```

**FactFleetTrips** — vehicle telemetry and route performance:
```sql
SELECT 
    ft.TripKey,
    ft.TimeKey,
    ft.VehicleKey,
    ft.RouteKey,
    ft.OriginGeographyKey,
    ft.DestinationGeographyKey,
    ft.TripDurationMinutes,
    ft.FuelConsumedGallons,
    ft.IdleTimeMinutes,
    ft.LoadWeightKG,
    ft.EngineHealthScore,
    (ft.IdleTimeMinutes * 1.0 / ft.TripDurationMinutes) * 100 AS IdlePercentage
FROM dbo.FactFleetTrips ft
WHERE ft.TripDurationMinutes > 0
    AND (ft.IdleTimeMinutes * 1.0 / ft.TripDurationMinutes) > 0.15  -- > 15% idle time
ORDER BY IdlePercentage DESC;
```

**FactCrossDock** — direct transfer operations (no long-term storage):
```sql
SELECT 
    cd.CrossDockKey,
    cd.TimeKey,
    cd.InboundShipmentKey,
    cd.OutboundShipmentKey,
    cd.ProductKey,
    cd.TransferDurationMinutes,
    cd.UnitsTransferred,
    t.FullDateTime
FROM dbo.FactCrossDock cd
INNER JOIN dbo.DimTime t ON cd.TimeKey = t.TimeKey
WHERE cd.TransferDurationMinutes < 120  -- Fast cross-dock < 2 hours
ORDER BY cd.TransferDurationMinutes ASC;
```

## Key Stored Procedures

### Incremental Data Loading

**sp_LoadWarehouseOperations** — ETL from WMS:
```sql
EXEC dbo.sp_LoadWarehouseOperations
    @StartDateTime = '2026-07-01 00:00:00',
    @EndDateTime = '2026-07-03 23:59:59',
    @SourceTable = 'WMS_Staging.dbo.Operations',
    @Mode = 'INCREMENTAL';  -- Options: 'FULL', 'INCREMENTAL', 'UPSERT'
```

**sp_LoadFleetTelemetry** — ETL from telematics API:
```sql
EXEC dbo.sp_LoadFleetTelemetry
    @StartDateTime = '2026-07-01 00:00:00',
    @EndDateTime = '2026-07-03 23:59:59',
    @TelemetrySource = 'GPS_API',
    @Mode = 'INCREMENTAL';
```

### Cross-Fact Analytics

**sp_CalculateGravityScores** — recalculate product gravity zones:
```sql
-- Run weekly or when pick patterns shift significantly
EXEC dbo.sp_CalculateGravityScores
    @AnalysisPeriodDays = 30,
    @WeightPickFrequency = 0.5,
    @WeightValuePerUnit = 0.3,
    @WeightFragility = 0.2;
```

**sp_PredictBottlenecks** — ML-powered congestion forecasting:
```sql
EXEC dbo.sp_PredictBottlenecks
    @ForecastHours = 24,
    @ConfidenceThreshold = 0.75,
    @OutputTable = 'Analytics.BottleneckForecast';
```

### Alerting

**sp_ConfigureAlert** — set up automated threshold notifications:
```sql
EXEC dbo.sp_ConfigureAlert
    @AlertName = 'FleetIdleTimeExceeded',
    @MetricQuery = 'SELECT AVG(IdleTimeMinutes * 1.0 / TripDurationMinutes) FROM FactFleetTrips WHERE TimeKey >= @RecentTimeKey',
    @Threshold = 0.15,
    @Operator = '>',
    @NotificationEmail = '${OPERATIONS_EMAIL}',
    @NotificationSMS = '${OPERATIONS_PHONE}',
    @CheckIntervalMinutes = 15;
```

## Configuration Patterns

### Role-Based Access Control

Implement row-level security in Power BI:

```sql
-- Create security role table
CREATE TABLE dbo.UserSecurity (
    UserEmail NVARCHAR(255),
    RoleName NVARCHAR(50),  -- 'User', 'Supervisor', 'Executive'
    AllowedGeographyKeys NVARCHAR(MAX),  -- Comma-separated list
    AllowedWarehouseCodes NVARCHAR(MAX)
);

-- Create security view
CREATE VIEW dbo.vw_SecureWarehouseOperations
AS
SELECT wo.*
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimGeography g ON wo.GeographyKey = g.GeographyKey
WHERE g.GeographyKey IN (
    SELECT CAST(value AS INT)
    FROM STRING_SPLIT(
        (SELECT AllowedGeographyKeys 
         FROM dbo.UserSecurity 
         WHERE UserEmail = USER_NAME()), 
        ',')
);
```

In Power BI:
1. Go to Modeling → Manage Roles
2. Create role "RegionalManager"
3. DAX filter: `[GeographyKey] IN VALUES(UserSecurity[AllowedGeographyKeys])`
4. Publish and configure RLS in Power BI Service

### External Data Integration

**Weather API enrichment** via SQL Server external table:

```sql
-- Enable Polybase (SQL Server 2019+)
EXEC sp_configure 'polybase enabled', 1;
RECONFIGURE;

-- Create external data source
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = 'https://api.weather.com/v3',
    CREDENTIAL = WeatherAPICredential
);

-- Create external table
CREATE EXTERNAL TABLE ext_WeatherEvents (
    EventID NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    EventType NVARCHAR(50),
    SeverityLevel INT,
    StartDateTime DATETIME2,
    EndDateTime DATETIME2
)
WITH (
    DATA_SOURCE = WeatherAPI,
    LOCATION = '/weather-alerts',
    FORMAT = 'JSON'
);

-- Join with fleet trips to correlate delays
SELECT 
    ft.TripKey,
    ft.DestinationGeographyKey,
    g.Latitude,
    g.Longitude,
    we.EventType,
    we.SeverityLevel,
    CASE 
        WHEN we.EventID IS NOT NULL THEN 'Weather-Impacted'
        ELSE 'Normal'
    END AS TripStatus
FROM dbo.FactFleetTrips ft
INNER JOIN dbo.DimGeography g ON ft.DestinationGeographyKey = g.GeographyKey
LEFT JOIN ext_WeatherEvents we ON
    ABS(g.Latitude - we.Latitude) < 0.5
    AND ABS(g.Longitude - we.Longitude) < 0.5
    AND ft.TripStartDateTime BETWEEN we.StartDateTime AND we.EndDateTime;
```

## Common Analytical Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(wo.DwellTimeMinutes) AS TotalDwellMinutes
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Putaway'
        AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(day, -30, GETDATE()))
    GROUP BY p.SKU
),
FleetIdle AS (
    SELECT 
        ft.RouteKey,
        r.RouteName,
        SUM(ft.IdleTimeMinutes * v.HourlyOperatingCost / 60.0) AS TotalIdleCost
    FROM dbo.FactFleetTrips ft
    INNER JOIN dbo.DimRoute r ON ft.RouteKey = r.RouteKey
    INNER JOIN dbo.DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE ft.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(day, -30, GETDATE()))
    GROUP BY ft.RouteKey, r.RouteName
)
SELECT 
    wd.SKU,
    wd.AvgDwellMinutes,
    fi.RouteName,
    fi.TotalIdleCost,
    -- Composite score: high dwell + high idle cost = optimization opportunity
    (wd.AvgDwellMinutes / 60.0) * fi.TotalIdleCost AS OptimizationPriority
FROM WarehouseDwell wd
CROSS JOIN FleetIdle fi  -- Intentional: identify all combinations
WHERE wd.AvgDwellMinutes > 48 * 60  -- > 2 days
    AND fi.TotalIdleCost > 500
ORDER BY OptimizationPriority DESC;
```

### Warehouse Gravity Zone Analysis

```sql
WITH ProductMovement AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.CurrentZone,
        p.RecommendedZone,
        COUNT(*) AS PickCount,
        AVG(wo.PickRateUnitsPerHour) AS AvgPickRate,
        SUM(wo.DwellTimeMinutes) AS TotalDwellMinutes
    FROM dbo.FactWarehouseOperations wo
    INNER JOIN dbo.DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Picking'
        AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(day, -7, GETDATE()))
    GROUP BY p.ProductKey, p.SKU, p.CurrentZone, p.RecommendedZone
)
SELECT 
    SKU,
    CurrentZone,
    RecommendedZone,
    PickCount,
    AvgPickRate,
    TotalDwellMinutes,
    CASE 
        WHEN CurrentZone <> RecommendedZone THEN 'REARRANGE'
        ELSE 'OPTIMAL'
    END AS Action,
    -- Estimated savings from rearrangement (simplified)
    CASE 
        WHEN CurrentZone <> RecommendedZone 
        THEN (TotalDwellMinutes * 0.10)  -- Assume 10% reduction in dwell
        ELSE 0
    END AS EstimatedMinutesSaved
FROM ProductMovement
WHERE PickCount > 50  -- High-frequency items
ORDER BY EstimatedMinutesSaved DESC;
```

### Temporal Elasticity Simulation

```sql
-- Scenario: What if warehouse capacity increases from 80% to 95%?
DECLARE @CurrentCapacityPct DECIMAL(5,2) = 80.0;
DECLARE @ProposedCapacityPct DECIMAL(5,2) = 95.0;
DECLARE @WarehouseVolumeCubicMeters DECIMAL(12,2) = 50000.0;

WITH BaselineMetrics AS (
    SELECT 
        AVG(wo.DwellTimeMinutes) AS BaselineDwellMinutes,
        AVG(wo.PickRateUnitsPerHour) AS BaselinePickRate,
        COUNT(*) AS BaselineOperations
    FROM dbo.FactWarehouseOperations wo
    WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(day, -30, GETDATE()))
),
SimulatedMetrics AS (
    SELECT 
        -- Assume 5% increase in dwell time per 10% capacity increase (simplified model)
        BaselineDwellMinutes * (1 + ((@ProposedCapacityPct - @CurrentCapacityPct) / 10.0 * 0.05)) AS ProjectedDwellMinutes,
        -- Assume 3% decrease in pick rate due to congestion
        BaselinePickRate * (1 - ((@ProposedCapacityPct - @CurrentCapacityPct) / 10.0 * 0.03)) AS ProjectedPickRate,
        -- Projected operations volume
        BaselineOperations * (@ProposedCapacityPct / @CurrentCapacityPct) AS ProjectedOperations
    FROM BaselineMetrics
)
SELECT 
    bm.BaselineDwellMinutes,
    sm.ProjectedDwellMinutes,
    ((sm.ProjectedDwellMinutes - bm.BaselineDwellMinutes) / bm.BaselineDwellMinutes) * 100 AS DwellChangePercent,
    bm.BaselinePickRate,
    sm.ProjectedPickRate,
    ((sm.ProjectedPickRate - bm.BaselinePickRate) / bm.BaselinePickRate) * 100 AS PickRateChangePercent,
    bm.BaselineOperations,
    sm.ProjectedOperations
FROM BaselineMetrics bm
CROSS JOIN SimulatedMetrics sm;
```

## Power BI DAX Patterns

### Cross-Fact Measure: Fleet Cost per Warehouse Operation

```dax
TotalFleetCostPerOperation = 
VAR FleetCost = 
    CALCULATE(
        SUM(FactFleetTrips[FuelConsumedGallons]) * 3.50 + 
        SUM(FactFleetTrips[IdleTimeMinutes]) * 0.85
    )
VAR WarehouseOps = 
    COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(FleetCost, WarehouseOps, 0)
```

### Time Intelligence: Rolling 7-Day Average Dwell Time

```dax
RollingAvgDwellTime7Day = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -7,
        DAY
    )
)
```

### Bottleneck Prediction Score

```dax
BottleneckScore = 
VAR DwellScore = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        72 * 60,  -- Normalize to 72-hour threshold
        0
    )
VAR IdleScore = 
    DIVIDE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        AVERAGE(FactFleetTrips[TripDurationMinutes]),
        0
    )
VAR CapacityScore = 
    DIVIDE(
        SUM(FactWarehouseOperations[UnitsInStorage]),
        50000,  -- Max warehouse capacity in units
        0
    )
RETURN
    (DwellScore * 0.4) + (IdleScore * 0.3) + (CapacityScore * 0.3)
```

## Troubleshooting

### Query Performance Issues

**Problem**: Cross-fact queries taking > 30 seconds

**Solution**: Ensure columnstore indexes on fact tables:
```sql
-- Create clustered columnstore index (best for large fact tables)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations
ON dbo.FactWarehouseOperations
WITH (DROP_EXISTING = OFF);

-- Create nonclustered columnstore for frequent filter columns
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips_Analytics
ON dbo.FactFleetTrips (TimeKey, VehicleKey, RouteKey, FuelConsumedGallons, IdleTimeMinutes);

-- Update statistics
UPDATE STATISTICS dbo.FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS dbo.FactFleetTrips WITH FULLSCAN;
```

**Problem**: Power BI refresh timeout (> 60 minutes)

**Solution**: Implement incremental refresh in Power BI:
1. Add `RangeStart` and `RangeEnd` parameters (DateTime type)
2. Filter fact tables: `WHERE FullDateTime >= RangeStart AND FullDateTime < RangeEnd`
3. Configure incremental refresh policy (e.g., 7 days, 2 years archive)

### Data Quality Issues

**Problem**: Missing dimension lookups causing NULL joins

**Solution**: Implement dimension default rows:
```sql
-- Insert "Unknown" row in each dimension
INSERT INTO dbo.DimProductGravity (ProductKey, SKU, ProductName, GravityScore, CurrentZone)
VALUES (-1, 'UNKNOWN', 'Unknown Product', 0, 'UNASSIGNED');

-- Add foreign key with default
ALTER TABLE dbo.FactWarehouseOperations
ADD CONSTRAINT DF_ProductKey DEFAULT -1 FOR ProductKey;

-- Clean orphaned facts
UPDATE dbo.FactWarehouseOperations
SET ProductKey = -1
WHERE ProductKey NOT IN (SELECT ProductKey FROM dbo.DimProductGravity);
```

**Problem**: Duplicate telemetry records from API

**Solution**: Use MERGE with surrogate key:
```sql
CREATE PROCEDURE dbo.sp_UpsertFleetTelemetry
    @TelemetrySourceTable NVARCHAR(255)
AS
BEGIN
    MERGE INTO dbo.FactFleetTrips AS Target
    USING (
        SELECT 
            VehicleID,
            TripStartDateTime,
            TripEndDateTime,
            -- Use composite business key
            HASHBYTES('SHA2_256', 
                CONCAT(VehicleID, '|', FORMAT(TripStartDateTime, 'yyyy-MM-dd HH:mm:ss'))
            ) AS BusinessKey,
            FuelConsumedGallons,
            IdleTimeMinutes,
            LoadWeightKG
        FROM @TelemetrySourceTable
    ) AS Source
    ON Target.BusinessKey = Source.BusinessKey
    WHEN MATCHED THEN
        UPDATE SET 
            Target.FuelConsumedGallons = Source.FuelConsumedGallons,
            Target.IdleTimeMinutes = Source.IdleTimeMinutes,
            Target.LoadWeightKG = Source.LoadWeightKG,
            Target.LastModifiedDateTime = GETDATE()
    WHEN NOT MATCHED THEN
        INSERT (BusinessKey, VehicleKey, TimeKey, FuelConsumedGallons, IdleTimeMinutes, LoadWeightKG)
        VALUES (Source.BusinessKey, dbo.fn_GetVehicleKey(Source.VehicleID), 
                dbo.fn_GetTimeKey(Source.TripStartDateTime), 
                Source.FuelConsumedGallons, Source.IdleTimeMinutes, Source.LoadWeightKG);
END;
```

### Power BI Relationship Issues

**Problem**: Ambiguous relationships between Date dimensions

**Solution**: Use role-playing dimensions:
```sql
-- Create separate date dimension views
CREATE VIEW dbo.vw_DimTime_Trip AS
SELECT * FROM dbo.DimTime;

CREATE VIEW dbo.vw_DimTime_Warehouse AS
SELECT * FROM dbo.DimTime;
```

In Power BI:
1. Import both views
2. Rename: `DimTime (Trip)` and `DimTime (Warehouse)`
3. Create explicit relationships:
   - FactFleetTrips[TimeKey] → DimTime (Trip)[TimeKey]
   - FactWarehouseOperations[TimeKey] → DimTime (Warehouse)[TimeKey]
4. Use USERELATIONSHIP() in DAX when needed

## Best Practices

1. **Partition large fact tables by month**:
```sql
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (202601, 202602, 202603, ...);

CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition ALL TO ([PRIMARY]);

CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- ... other columns
) ON ps_MonthlyPartition(TimeKey);
```

2. **Schedule ETL during off-peak hours**:
```sql
-- Create SQL Server Agent job
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_DailyETL';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_DailyETL',
    @step_name = 'Load Warehouse Operations',
    @command = 'EXEC dbo.sp_LoadWarehouseOperations @Mode = ''INCREMENTAL'';';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Daily_2AM',
    @freq_type = 4,  -- Daily
    @active_start_time = 020000;  -- 2:00 AM
```

3. **Version control DAX measures** in Power BI:
   - Export measures to Tabular Editor
   - Store in Git repository as `.json` files
   - Use CI/CD pipelines for deployment

4. **Monitor query performance** with Query Store:
```sql
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
ALTER DATABASE LogiFleetPulse 
SET QUERY_STORE (OPERATION_MODE = READ_WRITE, MAX_STORAGE_SIZE_MB = 1024);
```
