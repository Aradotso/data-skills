---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse for fleet operations, warehouse management, and cross-modal supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy warehouse operations data model"
  - "build fleet telemetry star schema"
  - "create supply chain KPI dashboard"
  - "implement cross-fact logistics queries"
  - "optimize warehouse gravity zones"
  - "troubleshoot Power BI logistics template"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence platform** built on MS SQL Server and Power BI. It unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a semantic data warehouse using a multi-fact star schema architecture.

**Core capabilities:**
- Unified data model linking warehouse micro-operations with fleet performance
- Time-phased dimension tables (15-minute granularity)
- Cross-fact KPI harmonization (e.g., inventory dwell time vs. fleet idling cost)
- Predictive bottleneck detection using composite aggregations
- Warehouse Gravity Zones™ for spatial optimization
- Role-based security and multilingual dashboards

**Primary components:**
- SQL schema with fact/dimension tables
- ETL stored procedures for incremental loading
- Power BI template (`.pbit`) with pre-built dashboards
- Alert automation via SQL jobs

## Installation & Prerequisites

### Requirements

- **MS SQL Server 2019+** (Standard or Enterprise edition recommended)
- **Power BI Desktop** (latest version)
- **SQL Server Management Studio (SSMS)** or Azure Data Studio
- Access to source systems: WMS, TMS, telemetry APIs, ERP

### Database Deployment

```sql
-- 1. Create database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- 2. Deploy schema (execute the provided schema scripts in order)
-- Scripts should be in repository: schema/01_dimensions.sql, 02_facts.sql, 03_views.sql, 04_procedures.sql

-- 3. Verify deployment
SELECT 
    t.name AS TableName,
    s.name AS SchemaName,
    t.type_desc
FROM sys.tables t
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE s.name IN ('dbo', 'fact', 'dim')
ORDER BY s.name, t.name
```

### Power BI Template Setup

```bash
# 1. Clone repository
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse

# 2. Open Power BI template
# File: LogiFleet_Pulse_Master.pbit

# 3. When prompted, enter connection parameters:
# - Server: your-sql-server.database.windows.net
# - Database: LogiFleetPulse
# - Authentication: SQL Server or Windows
```

## Key Data Model Components

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE fact.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes DECIMAL(10,2),
    CycleTimeMinutes DECIMAL(10,2),
    QuantityHandled INT,
    OperatorKey INT,
    GravityZoneKey INT,
    LoadDateTime DATETIME2 DEFAULT GETDATE()
)

-- FactFleetTrips: Vehicle telemetry and route performance
CREATE TABLE fact.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DepartureTimeKey INT,
    ArrivalTimeKey INT,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    MaintenanceScoreDaily DECIMAL(5,2),
    LoadDateTime DATETIME2 DEFAULT GETDATE()
)

-- FactCrossDock: Transfer operations (inbound to outbound)
CREATE TABLE fact.FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    DwellTimeMinutes DECIMAL(10,2),
    QuantityTransferred INT,
    LoadDateTime DATETIME2 DEFAULT GETDATE()
)
```

### Dimension Tables

```sql
-- DimTime: Time dimension with 15-minute granularity
CREATE TABLE dim.DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2,
    Hour INT,
    Minute INT,
    QuarterHour INT, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(20),
    DayOfMonth INT,
    Month INT,
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
)

-- DimProductGravity: Products with calculated gravity scores
CREATE TABLE dim.DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50),
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency
    ValueScore DECIMAL(5,2), -- Unit value
    FragilityScore DECIMAL(5,2),
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZoneType VARCHAR(50)
)

-- DimGeography: Hierarchical location dimension
CREATE TABLE dim.DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID VARCHAR(50),
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node'
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6)
)
```

## Common Query Patterns

### Cross-Fact Analysis: Warehouse Dwell Time vs Fleet Performance

```sql
-- Analyze how warehouse dwell time affects delivery performance
WITH WarehouseDwell AS (
    SELECT 
        wo.ProductKey,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        dt.DayOfWeek
    FROM fact.FactWarehouseOperations wo
    INNER JOIN dim.DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE wo.OperationType = 'Putaway'
      AND dt.DateTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY wo.ProductKey, dt.DayOfWeek
),
FleetPerformance AS (
    SELECT 
        ft.ProductKey, -- Linked via cross-dock bridge or manifest
        AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
        dt.DayOfWeek
    FROM fact.FactFleetTrips ft
    INNER JOIN dim.DimTime dt ON ft.TimeKey = dt.TimeKey
    WHERE dt.DateTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY ft.ProductKey, dt.DayOfWeek
)
SELECT 
    p.SKU,
    p.ProductName,
    wd.DayOfWeek,
    wd.AvgDwellTime,
    fp.AvgIdleTime,
    (wd.AvgDwellTime + fp.AvgIdleTime) AS TotalDelayMinutes
FROM WarehouseDwell wd
INNER JOIN FleetPerformance fp 
    ON wd.ProductKey = fp.ProductKey 
    AND wd.DayOfWeek = fp.DayOfWeek
INNER JOIN dim.DimProductGravity p ON wd.ProductKey = p.ProductKey
ORDER BY TotalDelayMinutes DESC
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be moved to different gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZoneType AS CurrentRecommendedZone,
    AVG(wo.DwellTimeMinutes) AS ActualAvgDwell,
    CASE 
        WHEN p.GravityScore > 75 AND AVG(wo.DwellTimeMinutes) > 120 
            THEN 'Move to High-Gravity Zone'
        WHEN p.GravityScore < 40 AND AVG(wo.DwellTimeMinutes) < 30 
            THEN 'Move to Low-Gravity Zone'
        ELSE 'Optimal Placement'
    END AS RecommendedAction
FROM dim.DimProductGravity p
INNER JOIN fact.FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
WHERE wo.OperationType = 'Picking'
  AND wo.TimeKey IN (
      SELECT TimeKey FROM dim.DimTime 
      WHERE DateTime >= DATEADD(DAY, -30, GETDATE())
  )
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZoneType
HAVING AVG(wo.DwellTimeMinutes) IS NOT NULL
ORDER BY p.GravityScore DESC
```

### Predictive Bottleneck Detection

```sql
-- Calculate bottleneck risk score using temporal patterns
CREATE PROCEDURE dbo.usp_CalculateBottleneckRisk
AS
BEGIN
    WITH HourlyLoad AS (
        SELECT 
            dt.Hour,
            dt.DayOfWeek,
            COUNT(*) AS OperationCount,
            AVG(wo.CycleTimeMinutes) AS AvgCycleTime
        FROM fact.FactWarehouseOperations wo
        INNER JOIN dim.DimTime dt ON wo.TimeKey = dt.TimeKey
        WHERE dt.DateTime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY dt.Hour, dt.DayOfWeek
    )
    SELECT 
        Hour,
        DayOfWeek,
        OperationCount,
        AvgCycleTime,
        CASE 
            WHEN OperationCount > (SELECT AVG(OperationCount) * 1.5 FROM HourlyLoad) 
                 AND AvgCycleTime > (SELECT AVG(AvgCycleTime) * 1.3 FROM HourlyLoad)
                THEN 'High Risk'
            WHEN OperationCount > (SELECT AVG(OperationCount) * 1.2 FROM HourlyLoad)
                THEN 'Medium Risk'
            ELSE 'Low Risk'
        END AS BottleneckRisk
    FROM HourlyLoad
    ORDER BY 
        CASE DayOfWeek
            WHEN 'Monday' THEN 1
            WHEN 'Tuesday' THEN 2
            WHEN 'Wednesday' THEN 3
            WHEN 'Thursday' THEN 4
            WHEN 'Friday' THEN 5
            WHEN 'Saturday' THEN 6
            WHEN 'Sunday' THEN 7
        END,
        Hour
END
```

## ETL & Data Loading

### Incremental Load Pattern

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE etl.usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from staging table (populated by external ETL tool)
    INSERT INTO fact.FactWarehouseOperations (
        TimeKey,
        WarehouseKey,
        ProductKey,
        OperationType,
        DwellTimeMinutes,
        CycleTimeMinutes,
        QuantityHandled,
        OperatorKey,
        GravityZoneKey
    )
    SELECT 
        dt.TimeKey,
        dw.WarehouseKey,
        dp.ProductKey,
        stg.OperationType,
        stg.DwellTimeMinutes,
        stg.CycleTimeMinutes,
        stg.QuantityHandled,
        do.OperatorKey,
        dgz.GravityZoneKey
    FROM staging.WarehouseOperations stg
    INNER JOIN dim.DimTime dt 
        ON CAST(stg.OperationDateTime AS DATE) = CAST(dt.DateTime AS DATE)
        AND DATEPART(HOUR, stg.OperationDateTime) = dt.Hour
        AND (DATEPART(MINUTE, stg.OperationDateTime) / 15) * 15 = dt.QuarterHour
    INNER JOIN dim.DimWarehouse dw ON stg.WarehouseID = dw.WarehouseID
    INNER JOIN dim.DimProductGravity dp ON stg.SKU = dp.SKU
    LEFT JOIN dim.DimOperator do ON stg.OperatorID = do.OperatorID
    LEFT JOIN dim.DimGravityZone dgz ON stg.ZoneID = dgz.ZoneID
    WHERE stg.OperationDateTime > @LastLoadDateTime
      AND stg.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE staging.WarehouseOperations
    SET IsProcessed = 1, ProcessedDateTime = GETDATE()
    WHERE OperationDateTime > @LastLoadDateTime
      AND IsProcessed = 0;
END
```

### Scheduling ETL Jobs

```sql
-- Create SQL Server Agent job for nightly ETL
USE msdb
GO

EXEC sp_add_job
    @job_name = N'LogiFleet_NightlyETL',
    @enabled = 1,
    @description = N'Incremental load of warehouse and fleet data'

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_NightlyETL',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @command = N'
        DECLARE @LastLoad DATETIME2 = (SELECT MAX(LoadDateTime) FROM fact.FactWarehouseOperations)
        EXEC etl.usp_LoadWarehouseOperations @LastLoad
    ',
    @database_name = N'LogiFleetPulse',
    @retry_attempts = 3,
    @retry_interval = 5

EXEC sp_add_schedule
    @schedule_name = N'Daily_2AM',
    @freq_type = 4, -- Daily
    @active_start_time = 020000 -- 2:00 AM

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_NightlyETL',
    @schedule_name = N'Daily_2AM'

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_NightlyETL'
```

## Power BI Configuration

### DAX Measures for Cross-Fact KPIs

```dax
// Measure: Total Dwell Time (Warehouse)
TotalDwellTime = 
SUM(FactWarehouseOperations[DwellTimeMinutes])

// Measure: Average Fleet Idle Time
AvgFleetIdleTime = 
AVERAGE(FactFleetTrips[IdleTimeMinutes])

// Measure: Composite Delay Score (Cross-Fact)
CompositeDelayScore = 
VAR WarehouseDelay = [TotalDwellTime]
VAR FleetDelay = [AvgFleetIdleTime] * COUNTROWS(FactFleetTrips)
RETURN
    (WarehouseDelay * 0.6) + (FleetDelay * 0.4)

// Measure: Gravity Zone Efficiency
GravityZoneEfficiency = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Picking"
    ),
    CALCULATE(
        SUM(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[OperationType] = "Putaway"
    ),
    0
)

// Measure: Fleet Fuel Efficiency (KM per Liter)
FleetFuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)
```

### Row-Level Security

```dax
// Create role: Regional Managers (see only their region's data)
[DimGeography[Region]] = USERNAME()

// Create role: Warehouse Supervisors (see only their warehouse)
[DimWarehouse[WarehouseID]] IN {
    LOOKUPVALUE(
        UserWarehouseMapping[WarehouseID],
        UserWarehouseMapping[UserEmail],
        USERNAME()
    )
}
```

### Power BI Template Parameters

When opening `.pbit` file, configure these parameters:

```
Server: ${SQL_SERVER_HOST}
Database: LogiFleetPulse
RefreshInterval: 15 (minutes)
SecurityRole: Manager | Supervisor | Executive
Language: en-US | es-ES | fr-FR | de-DE | zh-CN
```

## Alert Configuration

### SQL-Based Alert System

```sql
-- Create alert threshold table
CREATE TABLE config.AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY,
    AlertName VARCHAR(100),
    MetricType VARCHAR(50), -- 'DwellTime', 'IdleTime', 'FuelEfficiency'
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '='
    NotificationEmail VARCHAR(500),
    IsActive BIT DEFAULT 1
)

-- Insert sample thresholds
INSERT INTO config.AlertThresholds (AlertName, MetricType, ThresholdValue, ComparisonOperator, NotificationEmail)
VALUES 
    ('High Dwell Time Alert', 'DwellTime', 180, '>', 'warehouse-ops@company.com'),
    ('Excessive Fleet Idle', 'IdleTime', 30, '>', 'fleet-manager@company.com'),
    ('Low Fuel Efficiency', 'FuelEfficiency', 5.5, '<', 'logistics-director@company.com')

-- Alert evaluation stored procedure
CREATE PROCEDURE alerts.usp_EvaluateAlerts
AS
BEGIN
    DECLARE @AlertEmail VARCHAR(500)
    DECLARE @Subject VARCHAR(200)
    DECLARE @Body NVARCHAR(MAX)
    
    -- Check warehouse dwell time alerts
    IF EXISTS (
        SELECT 1 
        FROM fact.FactWarehouseOperations wo
        INNER JOIN dim.DimTime dt ON wo.TimeKey = dt.TimeKey
        INNER JOIN config.AlertThresholds at ON at.MetricType = 'DwellTime'
        WHERE dt.DateTime >= DATEADD(HOUR, -1, GETDATE())
          AND wo.DwellTimeMinutes > at.ThresholdValue
          AND at.IsActive = 1
    )
    BEGIN
        SET @Subject = 'LogiFleet Alert: High Warehouse Dwell Time Detected'
        SET @Body = N'Multiple items exceeded dwell time threshold in the last hour. Check dashboard for details.'
        
        -- Use Database Mail (requires configuration)
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = (SELECT NotificationEmail FROM config.AlertThresholds WHERE MetricType = 'DwellTime'),
            @subject = @Subject,
            @body = @Body
    END
END
```

## Configuration Files

### Sample Connection Config (JSON)

```json
{
  "database": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "encrypt": true,
    "trustServerCertificate": false
  },
  "etl": {
    "scheduledRefreshInterval": 15,
    "maxRetryAttempts": 3,
    "timeoutSeconds": 300
  },
  "dataSources": {
    "wms": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telemetry": {
      "endpoint": "${TELEMETRY_API_ENDPOINT}",
      "apiKey": "${TELEMETRY_API_KEY}"
    },
    "weather": {
      "endpoint": "https://api.openweathermap.org/data/2.5",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "alerts": {
    "smtpServer": "${SMTP_SERVER}",
    "smtpPort": 587,
    "emailFrom": "alerts@logifleet.company.com"
  }
}
```

## Troubleshooting

### Power BI Template Connection Issues

**Problem:** "Cannot connect to SQL Server" error when opening `.pbit`

**Solution:**
```sql
-- 1. Verify SQL Server allows remote connections
EXEC sp_configure 'remote access', 1
RECONFIGURE

-- 2. Check firewall allows port 1433
-- 3. Verify SQL Server authentication mode
SELECT SERVERPROPERTY('IsIntegratedSecurityOnly') 
-- Should return 0 for mixed mode
```

### Slow Dashboard Performance

**Problem:** Dashboards take >30 seconds to load

**Solutions:**
```sql
-- 1. Add indexes to fact tables
CREATE NONCLUSTERED INDEX IX_FactWO_TimeProduct 
ON fact.FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, CycleTimeMinutes)

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeVehicle
ON fact.FactFleetTrips(TimeKey, VehicleKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters)

-- 2. Enable table partitioning by date
CREATE PARTITION FUNCTION PF_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301) -- TimeKey format

CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition
ALL TO ([PRIMARY])

-- 3. Use Power BI aggregations (create aggregated tables)
CREATE TABLE fact.FactWarehouseOperations_Daily
AS
SELECT 
    CAST(dt.DateTime AS DATE) AS Date,
    wo.ProductKey,
    wo.WarehouseKey,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    SUM(wo.QuantityHandled) AS TotalQuantity
FROM fact.FactWarehouseOperations wo
INNER JOIN dim.DimTime dt ON wo.TimeKey = dt.TimeKey
GROUP BY CAST(dt.DateTime AS DATE), wo.ProductKey, wo.WarehouseKey
```

### Missing Data After ETL

**Problem:** Staging tables have data but fact tables are empty

**Diagnosis:**
```sql
-- Check staging table counts
SELECT 
    'staging.WarehouseOperations' AS TableName,
    COUNT(*) AS RowCount,
    MAX(OperationDateTime) AS LatestRecord,
    SUM(CASE WHEN IsProcessed = 1 THEN 1 ELSE 0 END) AS ProcessedCount
FROM staging.WarehouseOperations

-- Check dimension key mismatches
SELECT 
    stg.SKU,
    CASE WHEN dp.ProductKey IS NULL THEN 'Missing in DimProduct' ELSE 'OK' END AS Status
FROM staging.WarehouseOperations stg
LEFT JOIN dim.DimProductGravity dp ON stg.SKU = dp.SKU
WHERE stg.IsProcessed = 0
GROUP BY stg.SKU, dp.ProductKey

-- Solution: Add missing dimension records before fact load
INSERT INTO dim.DimProductGravity (SKU, ProductName, Category, VelocityScore, ValueScore, FragilityScore)
SELECT DISTINCT
    SKU,
    'Unknown Product' AS ProductName,
    'Uncategorized' AS Category,
    50 AS VelocityScore,
    50 AS ValueScore,
    50 AS FragilityScore
FROM staging.WarehouseOperations
WHERE SKU NOT IN (SELECT SKU FROM dim.DimProductGravity)
```

### DAX Measure Returns Blank

**Problem:** Custom DAX measures show blank values

**Common causes:**
```dax
// Issue: Incorrect filter context
// Wrong:
WrongMeasure = SUM(FactWarehouseOperations[DwellTimeMinutes])

// Correct: Explicit relationship navigation
CorrectMeasure = 
CALCULATE(
    SUM(FactWarehouseOperations[DwellTimeMinutes]),
    USERELATIONSHIP(FactWarehouseOperations[TimeKey], DimTime[TimeKey])
)

// Issue: Circular dependency
// Check with DAX Studio or Performance Analyzer
```

### Row-Level Security Not Working

**Problem:** Users see all data despite RLS roles

**Verification:**
```sql
-- Check if roles are defined in Power BI
-- File > Options > Security > Manage Roles

-- Test RLS in Power BI Desktop:
-- Modeling tab > View As > Select role

-- Verify user email matches USERNAME() function
-- Add diagnostic measure:
CurrentUser = USERNAME()
```

## Best Practices

### Optimize for Production

```sql
-- 1. Regular statistics updates
CREATE PROCEDURE maintenance.usp_UpdateStatistics
AS
BEGIN
    UPDATE STATISTICS fact.FactWarehouseOperations WITH FULLSCAN
    UPDATE STATISTICS fact.FactFleetTrips WITH FULLSCAN
    UPDATE STATISTICS fact.FactCrossDock WITH FULLSCAN
END

-- Schedule weekly via SQL Agent

-- 2. Archive old data (keep 2 years active)
CREATE PROCEDURE maintenance.usp_ArchiveOldData
    @ArchiveBeforeDate DATE
AS
BEGIN
    -- Move to archive tables
    INSERT INTO archive.FactWarehouseOperations_Archive
    SELECT * FROM fact.FactWarehouseOperations wo
    INNER JOIN dim.DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.DateTime < @ArchiveBeforeDate
    
    -- Delete from active tables
    DELETE wo
    FROM fact.FactWarehouseOperations wo
    INNER JOIN dim.DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.DateTime < @ArchiveBeforeDate
END
```

### Data Quality Checks

```sql
-- Create automated data quality validation
CREATE PROCEDURE dq.usp_ValidateDataQuality
AS
BEGIN
    -- Check for orphaned fact records
    SELECT 'Orphaned Warehouse Operations' AS Issue, COUNT(*) AS Count
    FROM fact.FactWarehouseOperations wo
    WHERE NOT EXISTS (SELECT 1 FROM dim.DimTime WHERE TimeKey = wo.TimeKey)
    
    UNION ALL
    
    -- Check for negative dwell times
    SELECT 'Negative Dwell Times', COUNT(*)
    FROM fact.FactWarehouseOperations
    WHERE DwellTimeMinutes < 0
    
    UNION ALL
    
    -- Check for fuel efficiency outliers
    SELECT 'Fuel Efficiency Outliers', COUNT(*)
    FROM fact.FactFleetTrips
    WHERE (DistanceKM / NULLIF(FuelConsumedLiters, 0)) > 20 -- Unrealistic efficiency
       OR (DistanceKM / NULLIF(FuelConsumedLiters, 0)) < 2
END
```

This skill provides comprehensive guidance for deploying and using LogiFleet Pulse, covering database setup, ETL patterns, Power BI configuration, and operational maintenance.
