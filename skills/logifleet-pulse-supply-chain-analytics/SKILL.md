---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure logistics data warehouse with Power BI"
  - "implement multi-fact star schema for fleet tracking"
  - "build warehouse gravity zone analytics"
  - "create cross-modal supply chain dashboard"
  - "deploy SQL Server logistics intelligence platform"
  - "integrate fleet telemetry with warehouse operations"
  - "optimize supply chain KPI harmonization"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing solution for logistics and supply chain analytics. It implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a semantic layer for real-time decision-making.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time dashboards with 15-minute refresh cycles
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive fleet triage engine with predictive maintenance
- Temporal elasticity modeling for scenario analysis

**Primary use cases:**
- 3PL operators tracking cross-dock efficiency
- Retail chains optimizing delivery routes
- Food distributors managing perishable inventory
- Supply chain managers analyzing bottlenecks

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telemetry APIs)

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Execute in SSMS or Azure Data Studio
-- Connect to your SQL Server instance first

-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema creation script
-- Assuming schema.sql is in the repository
:r schema.sql
GO

-- Verify tables are created
SELECT name, type_desc 
FROM sys.objects 
WHERE type IN ('U', 'V', 'P')
ORDER BY type_desc, name;
```

3. **Configure data source connections:**
```json
-- config.json (create from config_sample.json)
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    },
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "username": "${SQL_USERNAME}",
      "password": "${SQL_PASSWORD}"
    }
  },
  "refresh_interval_minutes": 15
}
```

4. **Set up incremental loading:**
```sql
-- Execute the incremental load stored procedure
EXEC dbo.sp_IncrementalLoad_FactWarehouseOperations;
EXEC dbo.sp_IncrementalLoad_FactFleetTrips;
EXEC dbo.sp_IncrementalLoad_FactCrossDock;

-- Schedule these using SQL Server Agent (example job)
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalRefresh',
    @enabled = 1;

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Refresh Warehouse Operations',
    @subsystem = N'TSQL',
    @command = N'EXEC LogiFleetPulse.dbo.sp_IncrementalLoad_FactWarehouseOperations',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalRefresh',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalRefresh';
```

### Power BI Configuration

1. **Open the template:**
```powershell
# Open Power BI Desktop and load template
# File -> Open -> LogiFleet_Pulse_Master.pbit
```

2. **Connect to SQL Server:**
```m
// Power Query M formula for SQL connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"),
        "LogiFleetPulse",
        [
            Query = "SELECT * FROM vw_UnifiedLogisticsView",
            CommandTimeout = #duration(0, 0, 5, 0)
        ]
    )
in
    Source
```

3. **Configure row-level security:**
```dax
// DAX formula for RLS (in Power BI Desktop)
// Create role "WarehouseManager"
[DimGeography].[WarehouseID] = LOOKUPVALUE(
    UserWarehouseMapping[WarehouseID],
    UserWarehouseMapping[UserEmail],
    USERPRINCIPALNAME()
)
```

## Key SQL Objects

### Core Fact Tables

**FactWarehouseOperations:**
```sql
-- Query warehouse efficiency metrics
SELECT 
    dw.WarehouseName,
    dt.DateKey,
    dt.HourOfDay,
    SUM(fwo.PickCount) AS TotalPicks,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
    SUM(fwo.PackingTimeSeconds) / 60.0 AS TotalPackingHours
FROM FactWarehouseOperations fwo
INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.DateKey >= CONVERT(varchar(8), DATEADD(day, -7, GETDATE()), 112)
GROUP BY dw.WarehouseName, dt.DateKey, dt.HourOfDay
ORDER BY dt.DateKey DESC, dt.HourOfDay;
```

**FactFleetTrips:**
```sql
-- Analyze fleet idle time and fuel efficiency
SELECT 
    dv.VehicleID,
    dv.VehicleType,
    dr.RouteName,
    COUNT(*) AS TripCount,
    SUM(fft.IdleTimeMinutes) AS TotalIdleMinutes,
    SUM(fft.FuelConsumedLiters) AS TotalFuel,
    AVG(fft.AverageSpeedKmh) AS AvgSpeed,
    SUM(fft.DistanceKm) AS TotalDistance
FROM FactFleetTrips fft
INNER JOIN DimVehicle dv ON fft.VehicleKey = dv.VehicleKey
INNER JOIN DimRoute dr ON fft.RouteKey = dr.RouteKey
INNER JOIN DimTime dt ON fft.DepartureTimeKey = dt.TimeKey
WHERE dt.DateKey >= CONVERT(varchar(8), DATEADD(day, -30, GETDATE()), 112)
GROUP BY dv.VehicleID, dv.VehicleType, dr.RouteName
HAVING SUM(fft.IdleTimeMinutes) / NULLIF(SUM(fft.TripDurationMinutes), 0) > 0.15
ORDER BY TotalIdleMinutes DESC;
```

**FactCrossDock:**
```sql
-- Cross-dock transfer efficiency
SELECT 
    dw.WarehouseName,
    dp.ProductCategory,
    COUNT(*) AS TransferCount,
    AVG(fcd.TransferDurationMinutes) AS AvgTransferTime,
    SUM(fcd.UnitsTransferred) AS TotalUnits
FROM FactCrossDock fcd
INNER JOIN DimWarehouse dw ON fcd.WarehouseKey = dw.WarehouseKey
INNER JOIN DimProduct dp ON fcd.ProductKey = dp.ProductKey
INNER JOIN DimTime dt ON fcd.InboundTimeKey = dt.TimeKey
WHERE dt.DateKey >= CONVERT(varchar(8), DATEADD(day, -7, GETDATE()), 112)
GROUP BY dw.WarehouseName, dp.ProductCategory
ORDER BY AvgTransferTime DESC;
```

### Dimension Tables

**DimProductGravity:**
```sql
-- Products with high gravity scores (fast-moving, high-value)
SELECT 
    ProductID,
    ProductName,
    GravityScore,
    PickFrequencyDaily,
    ProductValueUSD,
    FragilityLevel,
    RecommendedZone
FROM DimProductGravity
WHERE GravityScore > 80
ORDER BY GravityScore DESC;
```

**DimTime (time-phased dimension):**
```sql
-- Create time buckets for analysis
SELECT 
    TimeKey,
    FullDateTime,
    DateKey,
    HourOfDay,
    DayOfWeek,
    FiscalPeriod,
    IsWeekend,
    IsHoliday
FROM DimTime
WHERE DateKey >= CONVERT(varchar(8), DATEADD(day, -7, GETDATE()), 112);
```

### Views and Stored Procedures

**Unified semantic view:**
```sql
-- Pre-built view for cross-fact analysis
SELECT * FROM vw_UnifiedLogisticsView
WHERE WarehouseID = 'WH001'
  AND TripDate >= DATEADD(day, -7, GETDATE());
```

**Incremental load procedure:**
```sql
-- Manual incremental refresh
EXEC dbo.sp_IncrementalLoad_FactWarehouseOperations
    @LastLoadDate = '2026-07-01',
    @BatchSize = 10000;
```

**Alerting procedure:**
```sql
-- Set up threshold alerts
EXEC dbo.sp_ConfigureAlert
    @AlertName = 'HighIdleTime',
    @MetricName = 'FleetIdlePercentage',
    @Threshold = 15.0,
    @ComparisonOperator = '>',
    @NotificationEmail = '${ALERT_EMAIL}';
```

## Common Patterns

### Cross-Fact KPI Harmonization

Combine warehouse dwell time with fleet idle time:
```sql
-- Unified logistics efficiency score
WITH WarehouseMetrics AS (
    SELECT 
        fwo.ProductKey,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTime
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.DateKey >= CONVERT(varchar(8), DATEADD(day, -30, GETDATE()), 112)
    GROUP BY fwo.ProductKey
),
FleetMetrics AS (
    SELECT 
        fft.RouteKey,
        AVG(fft.IdleTimeMinutes * 100.0 / NULLIF(fft.TripDurationMinutes, 0)) AS AvgIdlePercentage
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.DepartureTimeKey = dt.TimeKey
    WHERE dt.DateKey >= CONVERT(varchar(8), DATEADD(day, -30, GETDATE()), 112)
    GROUP BY fft.RouteKey
)
SELECT 
    dp.ProductName,
    wm.AvgDwellTime,
    fm.AvgIdlePercentage,
    -- Composite efficiency score (lower is better)
    (wm.AvgDwellTime / 60.0) + (fm.AvgIdlePercentage / 10.0) AS CompositeScore
FROM WarehouseMetrics wm
CROSS JOIN FleetMetrics fm
INNER JOIN DimProduct dp ON wm.ProductKey = dp.ProductKey
ORDER BY CompositeScore DESC;
```

### Warehouse Gravity Zone Optimization

Identify products that should be moved to different zones:
```sql
-- Products in wrong gravity zone
SELECT 
    dp.ProductID,
    dp.ProductName,
    dpg.CurrentZone,
    dpg.RecommendedZone,
    dpg.GravityScore,
    COUNT(fwo.OperationID) AS RecentPickCount
FROM DimProduct dp
INNER JOIN DimProductGravity dpg ON dp.ProductKey = dpg.ProductKey
INNER JOIN FactWarehouseOperations fwo ON dp.ProductKey = fwo.ProductKey
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.DateKey >= CONVERT(varchar(8), DATEADD(day, -7, GETDATE()), 112)
  AND dpg.CurrentZone <> dpg.RecommendedZone
  AND dpg.GravityScore > 70
GROUP BY dp.ProductID, dp.ProductName, dpg.CurrentZone, dpg.RecommendedZone, dpg.GravityScore
HAVING COUNT(fwo.OperationID) > 50
ORDER BY dpg.GravityScore DESC, RecentPickCount DESC;
```

### Predictive Bottleneck Detection

Identify routes with high risk of delays:
```sql
-- Routes with correlated delay factors
WITH RouteRisk AS (
    SELECT 
        dr.RouteID,
        dr.RouteName,
        AVG(fft.DelayMinutes) AS AvgDelay,
        COUNT(CASE WHEN fft.WeatherImpact = 1 THEN 1 END) AS WeatherDelays,
        COUNT(CASE WHEN fft.TrafficImpact = 1 THEN 1 END) AS TrafficDelays,
        SUM(fft.MaintenanceFlag) AS MaintenanceEvents
    FROM FactFleetTrips fft
    INNER JOIN DimRoute dr ON fft.RouteKey = dr.RouteKey
    INNER JOIN DimTime dt ON fft.DepartureTimeKey = dt.TimeKey
    WHERE dt.DateKey >= CONVERT(varchar(8), DATEADD(day, -90, GETDATE()), 112)
    GROUP BY dr.RouteID, dr.RouteName
)
SELECT 
    RouteID,
    RouteName,
    AvgDelay,
    WeatherDelays,
    TrafficDelays,
    MaintenanceEvents,
    -- Risk score (higher = more bottleneck risk)
    (AvgDelay * 0.4) + (WeatherDelays * 0.2) + (TrafficDelays * 0.2) + (MaintenanceEvents * 0.2) AS BottleneckRiskScore
FROM RouteRisk
WHERE AvgDelay > 15
ORDER BY BottleneckRiskScore DESC;
```

### Temporal Elasticity Modeling

Simulate capacity changes:
```sql
-- What-if scenario: increase warehouse capacity by 20%
DECLARE @CapacityIncrease FLOAT = 1.20;

WITH CurrentState AS (
    SELECT 
        dw.WarehouseID,
        dw.CurrentCapacity,
        SUM(fwo.UnitsProcessed) AS CurrentThroughput,
        AVG(fwo.DwellTimeMinutes) AS CurrentDwellTime
    FROM FactWarehouseOperations fwo
    INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.DateKey >= CONVERT(varchar(8), DATEADD(day, -30, GETDATE()), 112)
    GROUP BY dw.WarehouseID, dw.CurrentCapacity
)
SELECT 
    WarehouseID,
    CurrentCapacity,
    CurrentCapacity * @CapacityIncrease AS ProjectedCapacity,
    CurrentThroughput,
    CurrentThroughput * @CapacityIncrease AS ProjectedThroughput,
    CurrentDwellTime,
    -- Assume dwell time decreases with more capacity (inverse relationship)
    CurrentDwellTime / @CapacityIncrease AS ProjectedDwellTime
FROM CurrentState;
```

## Power BI DAX Formulas

### Key Measures

**Total Dwell Time:**
```dax
Total Dwell Time = 
SUM(FactWarehouseOperations[DwellTimeMinutes])
```

**Fleet Utilization Rate:**
```dax
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[ActiveTimeMinutes]),
    SUM(FactFleetTrips[ActiveTimeMinutes]) + SUM(FactFleetTrips[IdleTimeMinutes]),
    0
) * 100
```

**Cross-Fact Efficiency Score:**
```dax
Logistics Efficiency Score = 
VAR WarehouseScore = 
    100 - (AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 10)
VAR FleetScore = 
    [Fleet Utilization %]
RETURN
    (WarehouseScore * 0.5) + (FleetScore * 0.5)
```

**Gravity Zone Compliance:**
```dax
Gravity Zone Compliance % = 
DIVIDE(
    COUNTROWS(
        FILTER(
            DimProductGravity,
            DimProductGravity[CurrentZone] = DimProductGravity[RecommendedZone]
        )
    ),
    COUNTROWS(DimProductGravity),
    0
) * 100
```

### Time Intelligence

**Year-over-Year Comparison:**
```dax
Dwell Time YoY % = 
VAR CurrentPeriod = [Total Dwell Time]
VAR PreviousYear = 
    CALCULATE(
        [Total Dwell Time],
        SAMEPERIODLASTYEAR(DimTime[FullDateTime])
    )
RETURN
    DIVIDE(CurrentPeriod - PreviousYear, PreviousYear, 0) * 100
```

**Rolling 7-Day Average:**
```dax
Rolling 7-Day Fleet Idle = 
CALCULATE(
    AVERAGE(FactFleetTrips[IdleTimeMinutes]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -7,
        DAY
    )
)
```

## Configuration

### Role-Based Security

```sql
-- Create user-warehouse mapping
CREATE TABLE UserWarehouseMapping (
    UserEmail NVARCHAR(255),
    WarehouseID NVARCHAR(50),
    RoleName NVARCHAR(50)
);

INSERT INTO UserWarehouseMapping VALUES
('${MANAGER_EMAIL}', 'WH001', 'WarehouseManager'),
('${SUPERVISOR_EMAIL}', 'WH001', 'Supervisor');

-- Create SQL login for Power BI service
CREATE LOGIN PowerBIService WITH PASSWORD = '${POWERBI_SERVICE_PASSWORD}';
CREATE USER PowerBIService FOR LOGIN PowerBIService;

-- Grant read access
GRANT SELECT ON SCHEMA::dbo TO PowerBIService;
```

### Alert Thresholds

```sql
-- Configure automated alerts
CREATE TABLE AlertConfiguration (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName NVARCHAR(100),
    MetricName NVARCHAR(100),
    Threshold FLOAT,
    ComparisonOperator NVARCHAR(10),
    NotificationEmail NVARCHAR(255),
    IsActive BIT
);

INSERT INTO AlertConfiguration VALUES
('HighDwellTime', 'AvgDwellTimeMinutes', 120.0, '>', '${ALERT_EMAIL}', 1),
('LowFleetUtilization', 'FleetUtilizationPercentage', 75.0, '<', '${ALERT_EMAIL}', 1),
('MaintenanceOverdue', 'VehicleMaintenanceDaysOverdue', 7.0, '>', '${ALERT_EMAIL}', 1);
```

### External Data Source Integration

```sql
-- Configure external table for weather API
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${WEATHER_API_BLOB_URL}',
    CREDENTIAL = WeatherAPICredential
);

CREATE EXTERNAL TABLE ExtWeatherData (
    Timestamp DATETIME2,
    Location NVARCHAR(100),
    Condition NVARCHAR(50),
    ImpactLevel INT
)
WITH (
    DATA_SOURCE = WeatherAPI,
    LOCATION = 'weather/',
    FILE_FORMAT = JSONFormat
);
```

## Troubleshooting

### Performance Issues

**Slow dashboard refresh:**
```sql
-- Check for missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE s.avg_fragmentation_in_percent > 30
  AND OBJECT_NAME(s.object_id) LIKE 'Fact%'
ORDER BY s.avg_fragmentation_in_percent DESC;

-- Rebuild fragmented indexes
ALTER INDEX ALL ON FactWarehouseOperations REBUILD;
ALTER INDEX ALL ON FactFleetTrips REBUILD;
```

**Add missing indexes:**
```sql
-- Index on TimeKey for date range queries
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeKey
ON FactWarehouseOperations(TimeKey)
INCLUDE (DwellTimeMinutes, UnitsProcessed);

-- Composite index for cross-fact queries
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Route_Time
ON FactFleetTrips(RouteKey, DepartureTimeKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);
```

### Data Quality Issues

**Check for orphaned records:**
```sql
-- Find fact records without matching dimensions
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanCount
FROM FactWarehouseOperations fwo
LEFT JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
WHERE dw.WarehouseKey IS NULL

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM FactFleetTrips fft
LEFT JOIN DimVehicle dv ON fft.VehicleKey = dv.VehicleKey
WHERE dv.VehicleKey IS NULL;
```

**Validate time dimension continuity:**
```sql
-- Find gaps in time dimension
WITH TimeSeries AS (
    SELECT 
        TimeKey,
        LEAD(TimeKey) OVER (ORDER BY TimeKey) AS NextTimeKey,
        FullDateTime
    FROM DimTime
)
SELECT *
FROM TimeSeries
WHERE DATEDIFF(MINUTE, FullDateTime, 
    (SELECT FullDateTime FROM DimTime WHERE TimeKey = NextTimeKey)) > 15;
```

### Power BI Connection Issues

**Test SQL connectivity:**
```sql
-- Verify Power BI service account permissions
EXECUTE AS USER = 'PowerBIService';
SELECT 
    SCHEMA_NAME(schema_id) AS SchemaName,
    name AS ObjectName,
    type_desc
FROM sys.objects
WHERE type IN ('U', 'V')
ORDER BY SchemaName, ObjectName;
REVERT;
```

**Refresh credentials in Power BI:**
```powershell
# PowerShell script to update data source credentials
# Run in Power BI Service
Set-PowerBIDataSourceCredentials `
    -DataSourceId "your-datasource-id" `
    -Credential (Get-Credential) `
    -CredentialType Basic
```

### Incremental Load Failures

**Check load status:**
```sql
-- View load history
SELECT 
    LoadDate,
    TableName,
    RecordsLoaded,
    LoadStatus,
    ErrorMessage
FROM LoadHistory
WHERE LoadDate >= DATEADD(day, -7, GETDATE())
ORDER BY LoadDate DESC;

-- Retry failed loads
EXEC dbo.sp_RetryFailedLoads @TableName = 'FactWarehouseOperations';
```

## Best Practices

1. **Partitioning strategy:** Partition fact tables by date for better query performance
```sql
-- Create partition function for monthly partitions
CREATE PARTITION FUNCTION PF_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES 
(20260101, 20260201, 20260301, 20260401, 20260501, 20260601);

-- Apply to fact table
CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition ALL TO ([PRIMARY]);
```

2. **Incremental refresh in Power BI:** Configure for last 30 days only
3. **Aggregate tables:** Create pre-aggregated views for common queries
```sql
CREATE VIEW vw_DailySummary AS
SELECT 
    dt.DateKey,
    dw.WarehouseID,
    SUM(fwo.UnitsProcessed) AS TotalUnits,
    AVG(fwo.DwellTimeMinutes) AS AvgDwell
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
GROUP BY dt.DateKey, dw.WarehouseID;
```

4. **Monitor query performance:**
```sql
-- Find expensive queries
SELECT TOP 10
    qs.execution_count,
    qs.total_elapsed_time / 1000000.0 AS total_seconds,
    qt.text AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
WHERE qt.text LIKE '%FactWarehouse%' OR qt.text LIKE '%FactFleet%'
ORDER BY qs.total_elapsed_time DESC;
```

5. **Regular maintenance:** Schedule weekly index maintenance and statistics updates
