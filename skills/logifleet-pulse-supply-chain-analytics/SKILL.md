---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up logifleet pulse supply chain dashboard"
  - "configure logistics analytics star schema"
  - "implement warehouse gravity zones"
  - "create fleet optimization dashboard in power bi"
  - "build supply chain intelligence data model"
  - "deploy logifleet pulse sql schema"
  - "configure cross-modal logistics analytics"
  - "set up warehouse and fleet kpi tracking"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that integrates warehouse operations, fleet telemetry, and supply chain data into a unified semantic layer. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema architecture for cross-domain analytics.

**Core Capabilities:**
- Multi-fact data warehouse (warehouse operations, fleet trips, cross-dock transfers)
- Time-phased dimensional modeling (15-minute granularity)
- Warehouse Gravity Zones™ for spatial optimization
- Fleet triage engine with predictive maintenance
- Cross-fact KPI harmonization
- Real-time Power BI dashboards with row-level security

## Installation & Setup

### Prerequisites

```bash
# Required software
# - MS SQL Server 2019 or later
# - Power BI Desktop (latest version)
# - SQL Server Management Studio (SSMS)
```

### 1. Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### 2. Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Run the schema deployment script

USE master;
GO

-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Hour INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20),
    FiscalPeriod VARCHAR(10),
    QuarterName VARCHAR(10),
    IsWeekend BIT DEFAULT 0
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    WarehouseCode VARCHAR(20),
    RouteNode VARCHAR(50),
    Latitude DECIMAL(10, 7),
    Longitude DECIMAL(10, 7),
    HierarchyLevel INT
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity + value + fragility
    Velocity VARCHAR(20), -- Fast/Medium/Slow mover
    ValueTier VARCHAR(20), -- High/Medium/Low value
    FragilityIndex DECIMAL(3,2),
    OptimalZoneType VARCHAR(50)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,3),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(20)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    PickRate DECIMAL(8,2),
    PackingTimeMinutes INT,
    ZoneCode VARCHAR(20),
    BatchNumber VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    RouteCode VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferTimeMinutes INT,
    PalletCount INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
```

### 3. Populate Time Dimension

```sql
-- Generate time dimension data (15-minute intervals)
DECLARE @StartDate DATETIME = '2024-01-01';
DECLARE @EndDate DATETIME = '2026-12-31';
DECLARE @CurrentDate DATETIME = @StartDate;
DECLARE @TimeKey INT = 1;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, DateTime, Hour, DayOfWeek, DayName, FiscalPeriod, QuarterName, IsWeekend)
    VALUES (
        @TimeKey,
        @CurrentDate,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        CONCAT('FY', YEAR(@CurrentDate), '-P', FORMAT(MONTH(@CurrentDate), '00')),
        CONCAT('Q', DATEPART(QUARTER, @CurrentDate), '-', YEAR(@CurrentDate)),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END
    );
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
    SET @TimeKey = @TimeKey + 1;
END;
```

## Configuration

### Connection Configuration

Create a `config.json` file (not tracked in git):

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated",
    "connection_timeout": 30
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_schedule": {
    "interval_minutes": 15,
    "full_refresh_hour": 2
  }
}
```

### Environment Variables

```bash
# Set these in your environment
export SQL_SERVER_HOST="your-sql-server.database.windows.net"
export WMS_API_ENDPOINT="https://your-wms-system.com/api"
export WMS_API_KEY="your_wms_api_key_here"
export TELEMATICS_API_ENDPOINT="https://fleet-telemetry.com/api"
export TELEMATICS_API_KEY="your_telematics_key_here"
export WEATHER_API_KEY="your_weather_api_key_here"
```

## Key SQL Procedures & Views

### Stored Procedure: Calculate Gravity Scores

```sql
CREATE PROCEDURE sp_CalculateProductGravity
AS
BEGIN
    -- Update gravity scores based on velocity, value, and fragility
    UPDATE DimProductGravity
    SET GravityScore = (
        CASE Velocity
            WHEN 'Fast' THEN 3.0
            WHEN 'Medium' THEN 2.0
            WHEN 'Slow' THEN 1.0
            ELSE 1.5
        END
        +
        CASE ValueTier
            WHEN 'High' THEN 2.0
            WHEN 'Medium' THEN 1.0
            WHEN 'Low' THEN 0.5
            ELSE 1.0
        END
        +
        (FragilityIndex * 1.5)
    ),
    OptimalZoneType = CASE
        WHEN GravityScore >= 5.0 THEN 'High-Gravity (Near Dock)'
        WHEN GravityScore >= 3.0 THEN 'Medium-Gravity (Mid-Range)'
        ELSE 'Low-Gravity (Back Storage)'
    END;
END;
GO
```

### View: Cross-Fact KPI Dashboard

```sql
CREATE VIEW vw_UnifiedLogisticsKPI
AS
SELECT
    t.DateTime,
    t.DayName,
    g.WarehouseCode,
    g.Region,
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneType,
    
    -- Warehouse metrics
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
    AVG(wo.PickRate) AS AvgPickRate,
    COUNT(DISTINCT wo.OperationKey) AS OperationCount,
    
    -- Fleet metrics (related)
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTime,
    AVG(ft.FuelConsumedLiters) AS AvgFuelConsumption,
    SUM(ft.DelayMinutes) AS TotalDelayMinutes,
    
    -- Composite KPI
    (AVG(wo.DwellTimeMinutes) * 0.4 + AVG(ft.IdleTimeMinutes) * 0.6) AS CompositeWasteIndex

FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
LEFT JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey

WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())

GROUP BY
    t.DateTime,
    t.DayName,
    g.WarehouseCode,
    g.Region,
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneType;
GO
```

### Stored Procedure: Fleet Triage Alert

```sql
CREATE PROCEDURE sp_FleetTriageAlert
    @IdleThresholdPct DECIMAL(5,2) = 15.0
AS
BEGIN
    -- Identify fleet trips with excessive idle time
    SELECT
        ft.VehicleID,
        ft.DriverID,
        ft.RouteCode,
        t.DateTime,
        ft.DistanceKm,
        ft.IdleTimeMinutes,
        (ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + (ft.DistanceKm / 60.0 * 60), 0)) AS IdlePercentage,
        ft.DelayReason,
        p.SKU,
        p.GravityScore,
        CASE
            WHEN p.GravityScore >= 5.0 THEN 'CRITICAL'
            WHEN p.GravityScore >= 3.0 THEN 'HIGH'
            ELSE 'NORMAL'
        END AS PriorityLevel
    FROM FactFleetTrips ft
    JOIN DimTime t ON ft.TimeKey = t.TimeKey
    LEFT JOIN FactCrossDock cd ON ft.TripKey = cd.InboundTripKey
    LEFT JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    WHERE
        t.DateTime >= DATEADD(HOUR, -24, GETDATE())
        AND (ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + (ft.DistanceKm / 60.0 * 60), 0)) > @IdleThresholdPct
    ORDER BY PriorityLevel DESC, IdlePercentage DESC;
END;
GO
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server and database details
4. Select DirectQuery mode for real-time dashboards

### DAX Measures for Cross-Fact Analysis

```dax
// Measure: Composite Waste Index
CompositeWasteIndex = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    (AvgDwell * 0.4) + (AvgIdle * 0.6)

// Measure: Gravity-Weighted Efficiency
GravityWeightedEfficiency = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[PickRate] * 
    RELATED(DimProductGravity[GravityScore])
) / SUM(DimProductGravity[GravityScore])

// Measure: Fleet Cost per High-Gravity Item
FleetCostPerHighGravityItem = 
CALCULATE(
    SUM(FactFleetTrips[FuelConsumedLiters]) * 1.5, // Assume $1.5/liter
    FILTER(
        DimProductGravity,
        DimProductGravity[GravityScore] >= 5.0
    )
) / 
CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FILTER(
        DimProductGravity,
        DimProductGravity[GravityScore] >= 5.0
    )
)

// Measure: Time-Phased Dwell Variance
DwellVariance = 
VAR CurrentPeriod = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR PriorPeriod = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[DateTime], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentPeriod - PriorPeriod, PriorPeriod, 0)
```

### Row-Level Security (RLS)

```dax
// Create role: Regional Manager
[Region] = USERNAME()

// Create role: Warehouse Supervisor
[WarehouseCode] = LOOKUPVALUE(
    DimGeography[WarehouseCode],
    DimGeography[GeographyKey],
    [GeographyKey]
)
```

## Data Loading Patterns

### Incremental Load Procedure

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadDateTime DATETIME
AS
BEGIN
    -- Insert new warehouse operations since last load
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, DwellTimeMinutes, CycleTimeMinutes,
        PickRate, PackingTimeMinutes, ZoneCode, BatchNumber
    )
    SELECT
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        wms.OperationType,
        wms.DwellTimeMinutes,
        wms.CycleTimeMinutes,
        wms.PickRate,
        wms.PackingTimeMinutes,
        wms.ZoneCode,
        wms.BatchNumber
    FROM ExternalWMSSource wms
    JOIN DimTime t ON CAST(wms.OperationDateTime AS DATETIME) = t.DateTime
    JOIN DimGeography g ON wms.WarehouseCode = g.WarehouseCode
    JOIN DimProductGravity p ON wms.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON wms.SupplierCode = s.SupplierCode
    WHERE wms.OperationDateTime > @LastLoadDateTime;
    
    -- Update last load timestamp
    UPDATE ETL_Metadata
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### External Table Configuration (Polybase)

```sql
-- Create external data source for WMS API
CREATE EXTERNAL DATA SOURCE WMS_API
WITH (
    TYPE = HADOOP,
    LOCATION = '${WMS_API_ENDPOINT}'
);

-- Create external file format (JSON)
CREATE EXTERNAL FILE FORMAT JSONFormat
WITH (FORMAT_TYPE = JSON);

-- Create external table
CREATE EXTERNAL TABLE ExternalWMSSource (
    OperationType VARCHAR(50),
    OperationDateTime DATETIME,
    WarehouseCode VARCHAR(20),
    SKU VARCHAR(50),
    SupplierCode VARCHAR(50),
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    PickRate DECIMAL(8,2),
    PackingTimeMinutes INT,
    ZoneCode VARCHAR(20),
    BatchNumber VARCHAR(50)
)
WITH (
    LOCATION = '/warehouse/operations/',
    DATA_SOURCE = WMS_API,
    FILE_FORMAT = JSONFormat
);
```

## Common Analytics Queries

### Query: Top 10 Bottleneck SKUs

```sql
SELECT TOP 10
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneType,
    wo.ZoneCode AS CurrentZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
    COUNT(*) AS OperationCount,
    CASE
        WHEN p.OptimalZoneType LIKE '%High-Gravity%' AND wo.ZoneCode NOT LIKE 'A%' THEN 'Misplaced'
        ELSE 'Correctly Placed'
    END AS PlacementStatus
FROM FactWarehouseOperations wo
JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneType, wo.ZoneCode
ORDER BY AvgDwellTime DESC;
```

### Query: Fleet Efficiency by Route

```sql
SELECT
    ft.RouteCode,
    og.Region AS OriginRegion,
    dg.Region AS DestinationRegion,
    COUNT(*) AS TripCount,
    AVG(ft.DistanceKm) AS AvgDistance,
    AVG(ft.FuelConsumedLiters) AS AvgFuel,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    SUM(ft.DelayMinutes) AS TotalDelays,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS FuelEfficiency,
    AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + (ft.DistanceKm / 60.0 * 60), 0)) AS IdlePercentage
FROM FactFleetTrips ft
JOIN DimTime t ON ft.TimeKey = t.TimeKey
JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY ft.RouteCode, og.Region, dg.Region
HAVING AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + (ft.DistanceKm / 60.0 * 60), 0)) > 10
ORDER BY IdlePercentage DESC;
```

### Query: Cross-Dock Performance

```sql
SELECT
    g.WarehouseCode,
    p.Category,
    COUNT(*) AS CrossDockCount,
    AVG(cd.TransferTimeMinutes) AS AvgTransferTime,
    SUM(cd.PalletCount) AS TotalPallets,
    AVG(inbound.IdleTimeMinutes) AS AvgInboundIdle,
    AVG(outbound.IdleTimeMinutes) AS AvgOutboundIdle,
    AVG(cd.TransferTimeMinutes + inbound.IdleTimeMinutes + outbound.IdleTimeMinutes) AS TotalTouchTime
FROM FactCrossDock cd
JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips inbound ON cd.InboundTripKey = inbound.TripKey
LEFT JOIN FactFleetTrips outbound ON cd.OutboundTripKey = outbound.TripKey
JOIN DimTime t ON cd.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -14, GETDATE())
GROUP BY g.WarehouseCode, p.Category
ORDER BY TotalTouchTime DESC;
```

## Automation & Alerting

### SQL Agent Job for Scheduled Refresh

```sql
-- Create SQL Agent job
USE msdb;
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleetPulse_IncrementalRefresh';

EXEC sp_add_jobstep
    @job_name = N'LogiFleetPulse_IncrementalRefresh',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_IncrementalLoadWarehouseOps @LastLoadDateTime = (SELECT LastLoadDateTime FROM ETL_Metadata WHERE TableName = ''FactWarehouseOperations'');',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_jobstep
    @job_name = N'LogiFleetPulse_IncrementalRefresh',
    @step_name = N'Calculate Gravity Scores',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_CalculateProductGravity;',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_jobschedule
    @job_name = N'LogiFleetPulse_IncrementalRefresh',
    @name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_add_jobserver
    @job_name = N'LogiFleetPulse_IncrementalRefresh';
GO
```

### Email Alert Configuration

```sql
-- Configure Database Mail (run once)
EXEC msdb.dbo.sysmail_add_account_sp
    @account_name = 'LogiFleetAlerts',
    @email_address = 'alerts@yourcompany.com',
    @mailserver_name = 'smtp.yourcompany.com';

-- Create alert stored procedure
CREATE PROCEDURE sp_SendFleetTriageAlert
AS
BEGIN
    DECLARE @AlertBody NVARCHAR(MAX);
    
    -- Build alert message
    SELECT @AlertBody = STRING_AGG(
        CONCAT(
            'Vehicle: ', VehicleID, 
            ' | Route: ', RouteCode, 
            ' | Idle %: ', CAST(IdlePercentage AS VARCHAR), 
            ' | Priority: ', PriorityLevel
        ),
        CHAR(13) + CHAR(10)
    )
    FROM (
        EXEC sp_FleetTriageAlert @IdleThresholdPct = 15.0
    ) AS AlertData;
    
    -- Send email if alerts exist
    IF @AlertBody IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'logistics-team@yourcompany.com',
            @subject = 'Fleet Triage Alert - High Idle Time Detected',
            @body = @AlertBody;
    END;
END;
GO
```

## Troubleshooting

### Issue: Slow Query Performance

**Solution:** Check indexing and partitioning

```sql
-- Analyze missing indexes
SELECT
    migs.avg_total_user_cost * (migs.avg_user_impact / 100.0) * (migs.user_seeks + migs.user_scans) AS improvement_measure,
    'CREATE INDEX IX_' + OBJECT_NAME(mid.object_id) + '_' + 
    REPLACE(REPLACE(REPLACE(mid.equality_columns, '[', ''), ']', ''), ', ', '_') +
    ' ON ' + mid.statement + ' (' + ISNULL(mid.equality_columns, '') + 
    CASE WHEN mid.inequality_columns IS NOT NULL THEN ',' + mid.inequality_columns ELSE '' END + ')' AS create_index_statement
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
WHERE database_id = DB_ID('LogiFleetPulse')
ORDER BY improvement_measure DESC;

-- Enable table partitioning for large fact tables
CREATE PARTITION FUNCTION PF_MonthlyPartition (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2024-01-01', '2024-02-01', '2024-03-01', '2024-04-01',
    '2024-05-01', '2024-06-01', '2024-07-01', '2024-08-01',
    '2024-09-01', '2024-10-01', '2024-11-01', '2024-12-01'
);

CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition
ALL TO ([PRIMARY]);
```

### Issue: Data Freshness Gaps

**Solution:** Monitor ETL metadata and add logging

```sql
-- Create ETL logging table
CREATE TABLE ETL_Metadata (
    TableName VARCHAR(100) PRIMARY KEY,
    LastLoadDateTime DATETIME,
    RecordsLoaded INT,
    LoadStatus VARCHAR(50)
);

-- Check data freshness
SELECT
    TableName,
    LastLoadDateTime,
    DATEDIFF(MINUTE, LastLoadDateTime, GETDATE()) AS MinutesSinceLoad,
    LoadStatus
FROM ETL_Metadata
WHERE DATEDIFF(MINUTE, LastLoadDateTime, GETDATE()) > 20; -- Alert if > 20 min
```

### Issue: Power BI DirectQuery Timeout

**Solution:** Use aggregation tables

```sql
-- Create aggregation table for common queries
CREATE TABLE AggWarehouseDaily (
    DateKey INT,
    WarehouseCode VARCHAR(20),
    ProductCategory VARCHAR(100),
    TotalOperations INT,
    AvgDwellTime DECIMAL(10,2),
    AvgCycleTime DECIMAL(10,2),
    PRIMARY KEY (DateKey, WarehouseCode, ProductCategory)
);

-- Populate aggregation (run daily)
INSERT INTO AggWarehouseDaily
SELECT
    CAST(CONVERT(VARCHAR, t.DateTime, 112) AS INT) AS DateKey,
    g.WarehouseCode,
    p.Category,
    COUNT(*) AS TotalOperations,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime
FROM FactWarehouseOperations wo
JOIN DimTime t ON wo.TimeKey = t.TimeKey
JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE t.DateTime >= DATEADD(DAY, -1, GETDATE())
GROUP BY CAST(CONVERT(VARCHAR, t.DateTime, 112) AS INT), g.WarehouseCode, p.Category;
```

## Best Practices

1. **Incremental Loading**: Always use timestamp-based incremental loads to minimize data transfer
2. **Indexing Strategy**: Create indexes on all foreign keys and frequently filtered columns
3. **Row-Level Security**: Implement RLS early to avoid rework when scaling to multiple teams
4. **Gravity Score Recalculation**: Run `sp_CalculateProductGravity` weekly to adapt to changing patterns
5. **
