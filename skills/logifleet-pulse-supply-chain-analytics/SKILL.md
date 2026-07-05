---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact logistics data warehouse for warehouse, fleet, and supply chain intelligence
triggers:
  - set up logifleet pulse supply chain analytics
  - configure logistics data warehouse with power bi
  - implement multi-fact star schema for fleet management
  - create warehouse gravity zone analytics
  - build cross-modal supply chain dashboard
  - deploy sql server logistics intelligence platform
  - integrate fleet telemetry with warehouse operations
  - set up predictive bottleneck detection for logistics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for logistics and supply chain analytics. It implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for cross-functional KPI analysis.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Unified warehouse and fleet analytics
- Warehouse Gravity Zones™ for spatial optimization
- Predictive bottleneck detection
- Real-time cross-fact KPI harmonization
- Role-based access control and row-level security

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics feeds, ERP systems

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

Connect to your SQL Server instance and execute the schema scripts in order:

```sql
-- 1. Create database
USE master;
GO
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Execute dimension tables (from schema/dimensions/)
-- Run in this order:
:r schema/dimensions/DimTime.sql
:r schema/dimensions/DimGeography.sql
:r schema/dimensions/DimProductGravity.sql
:r schema/dimensions/DimSupplierReliability.sql
:r schema/dimensions/DimVehicle.sql
:r schema/dimensions/DimWarehouseZone.sql

-- 3. Execute fact tables (from schema/facts/)
:r schema/facts/FactWarehouseOperations.sql
:r schema/facts/FactFleetTrips.sql
:r schema/facts/FactCrossDock.sql
:r schema/facts/FactInventorySnapshot.sql

-- 4. Create bridge tables (from schema/bridges/)
:r schema/bridges/BridgeRouteZone.sql

-- 5. Deploy views and stored procedures
:r schema/views/vw_UnifiedLogistics.sql
:r schema/procedures/sp_IncrementalLoad.sql
:r schema/procedures/sp_AlertEngine.sql
```

### Step 3: Configure Data Source Connections

Update the configuration file with your connection strings:

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "IntegratedSecurity",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "type": "REST",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telematics": {
      "type": "ODBC",
      "connectionString": "${TELEMATICS_CONNECTION_STRING}"
    },
    "weather": {
      "type": "REST",
      "endpoint": "https://api.weatherapi.com/v1",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refreshInterval": "15minutes",
  "alertThresholds": {
    "dwellTimeHours": 72,
    "idleTimePercentage": 15,
    "temperatureVariance": 2.0
  }
}
```

## Core Data Model

### Multi-Fact Star Schema

The platform uses a multi-fact architecture with shared dimensions:

```sql
-- Example: Query across facts using shared dimensions
SELECT 
    t.DateKey,
    t.HourOfDay,
    g.WarehouseName,
    g.Region,
    
    -- Warehouse metrics
    SUM(wo.PickCount) AS TotalPicks,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    
    -- Fleet metrics  
    SUM(ft.FuelConsumed) AS TotalFuel,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    
    -- Cross-fact KPI: Efficiency ratio
    CAST(SUM(wo.PickCount) AS FLOAT) / NULLIF(SUM(ft.IdleTimeMinutes), 0) AS PicksPerIdleMinute
    
FROM DimTime t
INNER JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
INNER JOIN FactFleetTrips ft ON t.TimeKey = ft.DepartureTimeKey
INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
WHERE t.DateKey BETWEEN '20260701' AND '20260731'
GROUP BY t.DateKey, t.HourOfDay, g.WarehouseName, g.Region
ORDER BY t.DateKey, t.HourOfDay;
```

### Warehouse Gravity Zones Implementation

```sql
-- Calculate and assign gravity scores
WITH ProductVelocity AS (
    SELECT 
        ProductKey,
        COUNT(*) AS PickFrequency,
        AVG(UnitValue) AS AvgValue,
        MAX(FragilityScore) AS Fragility
    FROM FactWarehouseOperations
    WHERE DateKey >= DATEADD(day, -90, GETDATE())
    GROUP BY ProductKey
),
GravityScore AS (
    SELECT 
        ProductKey,
        -- Weighted scoring: frequency (50%), value (30%), fragility (20%)
        (PickFrequency * 0.5) + 
        (AvgValue * 0.3) + 
        (Fragility * 0.2) AS GravityScore
    FROM ProductVelocity
)
UPDATE DimProductGravity
SET 
    GravityScore = gs.GravityScore,
    RecommendedZone = CASE 
        WHEN gs.GravityScore >= 80 THEN 'A' -- High gravity (near dock)
        WHEN gs.GravityScore >= 50 THEN 'B' -- Medium gravity
        ELSE 'C' -- Low gravity (back of warehouse)
    END,
    LastCalculated = GETDATE()
FROM DimProductGravity dpg
INNER JOIN GravityScore gs ON dpg.ProductKey = gs.ProductKey;
```

### Predictive Bottleneck Index

```sql
-- Stored procedure for bottleneck detection
CREATE PROCEDURE sp_PredictiveBottleneck
    @HorizonDays INT = 7
AS
BEGIN
    WITH HistoricalPatterns AS (
        SELECT 
            WarehouseKey,
            DATEPART(HOUR, TimeKey) AS HourOfDay,
            DATEPART(WEEKDAY, TimeKey) AS DayOfWeek,
            AVG(DwellTimeHours) AS AvgDwell,
            STDEV(DwellTimeHours) AS StdDevDwell,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations
        WHERE DateKey >= DATEADD(day, -90, GETDATE())
        GROUP BY WarehouseKey, DATEPART(HOUR, TimeKey), DATEPART(WEEKDAY, TimeKey)
    ),
    CurrentTrends AS (
        SELECT 
            WarehouseKey,
            AVG(DwellTimeHours) AS RecentAvgDwell
        FROM FactWarehouseOperations
        WHERE DateKey >= DATEADD(day, -7, GETDATE())
        GROUP BY WarehouseKey
    )
    SELECT 
        hp.WarehouseKey,
        g.WarehouseName,
        hp.HourOfDay,
        hp.DayOfWeek,
        hp.AvgDwell AS HistoricalAvg,
        ct.RecentAvgDwell AS CurrentAvg,
        -- Bottleneck probability (0-100)
        CASE 
            WHEN ct.RecentAvgDwell > (hp.AvgDwell + 2 * hp.StdDevDwell) THEN 95
            WHEN ct.RecentAvgDwell > (hp.AvgDwell + hp.StdDevDwell) THEN 70
            WHEN ct.RecentAvgDwell > hp.AvgDwell THEN 40
            ELSE 10
        END AS BottleneckProbability,
        hp.OperationCount AS HistoricalVolume
    FROM HistoricalPatterns hp
    INNER JOIN CurrentTrends ct ON hp.WarehouseKey = ct.WarehouseKey
    INNER JOIN DimGeography g ON hp.WarehouseKey = g.GeographyKey
    WHERE ct.RecentAvgDwell > hp.AvgDwell
    ORDER BY BottleneckProbability DESC, hp.WarehouseKey, hp.HourOfDay;
END;
GO

-- Execute bottleneck prediction
EXEC sp_PredictiveBottleneck @HorizonDays = 7;
```

## Power BI Integration

### Connect to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. When prompted, enter your SQL Server connection details:
   - **Server**: Your SQL Server instance
   - **Database**: LogiFleetPulse
   - **Data Connectivity mode**: DirectQuery (for real-time) or Import (for performance)

### Key DAX Measures

```dax
-- Cross-fact efficiency measure
PickEfficiency = 
CALCULATE(
    DIVIDE(
        SUM(FactWarehouseOperations[PickCount]),
        SUM(FactFleetTrips[IdleTimeMinutes])
    ),
    USERELATIONSHIP(DimTime[TimeKey], FactWarehouseOperations[TimeKey])
)

-- Dynamic gravity zone compliance
GravityCompliance = 
VAR ActualPicks = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[CurrentZone] = DimProductGravity[RecommendedZone]
    )
VAR TotalPicks = COUNT(FactWarehouseOperations[OperationKey])
RETURN DIVIDE(ActualPicks, TotalPicks)

-- Fleet utilization with thresholds
FleetUtilization = 
VAR TotalTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR Utilization = DIVIDE(TotalTime - IdleTime, TotalTime)
RETURN
    IF(
        Utilization < 0.70, 
        "Low (" & FORMAT(Utilization, "0%") & ")",
        IF(
            Utilization > 0.90,
            "Optimal (" & FORMAT(Utilization, "0%") & ")",
            "Good (" & FORMAT(Utilization, "0%") & ")"
        )
    )

-- Temporal elasticity forecast
ForecastDwellTime = 
VAR HistoricalAvg = CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeHours]), DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -90, DAY))
VAR CurrentTrend = CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeHours]), DATESINPERIOD(DimTime[Date], MAX(DimTime[Date]), -7, DAY))
VAR TrendMultiplier = DIVIDE(CurrentTrend, HistoricalAvg, 1)
RETURN HistoricalAvg * TrendMultiplier * 1.1 -- 10% buffer
```

### Row-Level Security (RLS)

```dax
-- Create role: Regional Manager
[Region] = USERNAME()

-- Create role: Warehouse Supervisor
[WarehouseName] IN 
    SELECTCOLUMNS(
        FILTER(
            UserWarehouseAccess,
            UserWarehouseAccess[Username] = USERNAME()
        ),
        "Warehouse", UserWarehouseAccess[WarehouseName]
    )
```

## Automated Alerting

### Configure Email/SMS Alerts

```sql
-- Alert engine stored procedure
CREATE PROCEDURE sp_AlertEngine
AS
BEGIN
    -- Check dwell time threshold
    DECLARE @DwellAlerts TABLE (
        WarehouseName NVARCHAR(100),
        SKU NVARCHAR(50),
        DwellTimeHours DECIMAL(10,2),
        AlertMessage NVARCHAR(500)
    );
    
    INSERT INTO @DwellAlerts
    SELECT 
        g.WarehouseName,
        p.SKU,
        wo.DwellTimeHours,
        'SKU ' + p.SKU + ' has exceeded dwell threshold at ' + g.WarehouseName + 
        ' with ' + CAST(wo.DwellTimeHours AS NVARCHAR) + ' hours'
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.WarehouseKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.DwellTimeHours > 72
      AND wo.DateKey = CONVERT(VARCHAR(8), GETDATE(), 112);
    
    -- Send alerts via Database Mail
    IF EXISTS (SELECT 1 FROM @DwellAlerts)
    BEGIN
        DECLARE @EmailBody NVARCHAR(MAX);
        SET @EmailBody = 
            '<h2>Dwell Time Alerts</h2><table border="1">' +
            '<tr><th>Warehouse</th><th>SKU</th><th>Dwell Time</th></tr>';
        
        SELECT @EmailBody = @EmailBody + 
            '<tr><td>' + WarehouseName + '</td><td>' + SKU + '</td><td>' + 
            CAST(DwellTimeHours AS NVARCHAR) + ' hrs</td></tr>'
        FROM @DwellAlerts;
        
        SET @EmailBody = @EmailBody + '</table>';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet_Alerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Dwell Time Alert',
            @body = @EmailBody,
            @body_format = 'HTML';
    END;
    
    -- Check fleet idle time threshold
    DECLARE @IdleAlerts TABLE (
        VehicleID NVARCHAR(50),
        Route NVARCHAR(100),
        IdlePercentage DECIMAL(5,2),
        AlertMessage NVARCHAR(500)
    );
    
    INSERT INTO @IdleAlerts
    SELECT 
        v.VehicleID,
        ft.RouteName,
        (ft.IdleTimeMinutes * 100.0 / ft.TripDurationMinutes) AS IdlePercentage,
        'Vehicle ' + v.VehicleID + ' idle time is ' + 
        CAST((ft.IdleTimeMinutes * 100.0 / ft.TripDurationMinutes) AS NVARCHAR) + 
        '% on route ' + ft.RouteName
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE (ft.IdleTimeMinutes * 100.0 / ft.TripDurationMinutes) > 15
      AND ft.DepartureDateKey = CONVERT(VARCHAR(8), GETDATE(), 112);
    
    IF EXISTS (SELECT 1 FROM @IdleAlerts)
    BEGIN
        -- Send fleet idle alerts (similar pattern as above)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet_Alerts',
            @recipients = '${FLEET_MANAGER_EMAIL}',
            @subject = 'LogiFleet Pulse: Fleet Idle Time Alert',
            @body = 'See attached idle time violations',
            @body_format = 'HTML';
    END;
END;
GO

-- Schedule via SQL Server Agent
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_HourlyAlerts',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_HourlyAlerts',
    @step_name = 'Run Alert Engine',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_AlertEngine;',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Hourly',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 8,
    @freq_subday_interval = 1;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_HourlyAlerts',
    @schedule_name = 'Hourly';
```

## Common Patterns

### Incremental Data Loading

```sql
-- Incremental load procedure
CREATE PROCEDURE sp_IncrementalLoad_WarehouseOps
    @LastLoadTimestamp DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from staging table populated by SSIS/ETL
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationTypeKey,
        PickCount, DwellTimeHours, CurrentZone, LoadTimestamp
    )
    SELECT 
        CAST(CONVERT(VARCHAR(8), s.OperationTimestamp, 112) + 
             RIGHT('0' + CAST(DATEPART(HOUR, s.OperationTimestamp) AS VARCHAR), 2) + 
             RIGHT('0' + CAST((DATEPART(MINUTE, s.OperationTimestamp) / 15) * 15 AS VARCHAR), 2) AS INT) AS TimeKey,
        dg.GeographyKey AS WarehouseKey,
        dp.ProductKey,
        ot.OperationTypeKey,
        s.PickCount,
        DATEDIFF(HOUR, s.ReceiveTimestamp, s.OperationTimestamp) AS DwellTimeHours,
        s.CurrentZone,
        GETDATE() AS LoadTimestamp
    FROM Staging_WarehouseOps s
    INNER JOIN DimGeography dg ON s.WarehouseCode = dg.WarehouseCode
    INNER JOIN DimProductGravity dp ON s.SKU = dp.SKU
    INNER JOIN DimOperationType ot ON s.OperationType = ot.OperationTypeName
    WHERE s.OperationTimestamp > @LastLoadTimestamp
      AND s.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE Staging_WarehouseOps
    SET IsProcessed = 1, ProcessedTimestamp = GETDATE()
    WHERE OperationTimestamp > @LastLoadTimestamp
      AND IsProcessed = 0;
      
    -- Update last load watermark
    UPDATE ETL_Config
    SET LastLoadTimestamp = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Cross-Fact Analysis

```sql
-- Unified logistics view
CREATE VIEW vw_UnifiedLogistics
AS
SELECT 
    t.Date,
    t.HourOfDay,
    g.WarehouseName,
    g.Region,
    
    -- Warehouse metrics
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOperations,
    AVG(wo.DwellTimeHours) AS AvgDwellTime,
    SUM(wo.PickCount) AS TotalPicks,
    
    -- Fleet metrics
    COUNT(DISTINCT ft.TripKey) AS FleetTrips,
    SUM(ft.FuelConsumed) AS TotalFuel,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    
    -- Cross-dock metrics
    COUNT(DISTINCT cd.CrossDockKey) AS CrossDockTransfers,
    AVG(cd.TransferTimeMinutes) AS AvgTransferTime,
    
    -- Composite KPIs
    CAST(SUM(wo.PickCount) AS FLOAT) / NULLIF(COUNT(DISTINCT ft.TripKey), 0) AS PicksPerTrip,
    CAST(SUM(ft.IdleTimeMinutes) AS FLOAT) / NULLIF(SUM(ft.TripDurationMinutes), 0) AS IdlePercentage
    
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.DepartureTimeKey
LEFT JOIN FactCrossDock cd ON t.TimeKey = cd.TransferTimeKey
LEFT JOIN DimGeography g ON COALESCE(wo.WarehouseKey, ft.OriginWarehouseKey, cd.OriginWarehouseKey) = g.GeographyKey
GROUP BY t.Date, t.HourOfDay, g.WarehouseName, g.Region;
GO
```

## Troubleshooting

### Common Issues

**Issue: Power BI refresh fails with "Unable to connect"**
```sql
-- Check SQL Server allows remote connections
EXEC sp_configure 'remote access', 1;
RECONFIGURE;

-- Verify firewall rules allow port 1433
-- Check SQL Server Browser service is running
```

**Issue: Slow dashboard performance**
```sql
-- Create recommended indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time 
ON FactWarehouseOperations(TimeKey) 
INCLUDE (WarehouseKey, ProductKey, PickCount);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Departure
ON FactFleetTrips(DepartureTimeKey)
INCLUDE (VehicleKey, RouteName, TripDurationMinutes);

-- Enable query store for performance analysis
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
```

**Issue: Gravity scores not updating**
```sql
-- Check last calculation timestamp
SELECT ProductKey, SKU, GravityScore, LastCalculated
FROM DimProductGravity
WHERE LastCalculated < DATEADD(day, -1, GETDATE());

-- Manually trigger recalculation
EXEC sp_RecalculateGravityScores;
```

**Issue: Alerts not sending**
```sql
-- Verify Database Mail is configured
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'LogiFleet_Alerts',
    @recipients = 'test@company.com',
    @subject = 'Test Alert',
    @body = 'This is a test';

-- Check mail queue
SELECT * FROM msdb.dbo.sysmail_mailitems
WHERE send_request_date >= DATEADD(hour, -1, GETDATE());

-- Check for errors
SELECT * FROM msdb.dbo.sysmail_event_log
WHERE event_type = 'error'
  AND log_date >= DATEADD(hour, -1, GETDATE());
```

## Environment Variables Reference

Required environment variables for deployment:

```bash
# SQL Server connection
SQL_SERVER_HOST=your-server.database.windows.net
SQL_USERNAME=logifleet_admin
SQL_PASSWORD=your_secure_password

# Data source APIs
WMS_API_ENDPOINT=https://wms.yourcompany.com/api/v1
WMS_API_KEY=your_wms_api_key
TELEMATICS_CONNECTION_STRING=DSN=Telematics;UID=user;PWD=pass
WEATHER_API_KEY=your_weather_api_key

# Alerting
ALERT_EMAIL_RECIPIENTS=ops-team@company.com;logistics@company.com
FLEET_MANAGER_EMAIL=fleet-manager@company.com
```

## Best Practices

1. **Partitioning**: Partition fact tables by date range (monthly) for tables exceeding 100M rows
2. **Indexing**: Create covering indexes on frequently joined columns
3. **Incremental Loads**: Always use watermark-based incremental loading for large datasets
4. **DirectQuery vs Import**: Use DirectQuery for real-time dashboards, Import for historical analysis
5. **Security**: Implement RLS in Power BI for multi-tenant or departmental access
6. **Monitoring**: Enable Query Store and review top resource-consuming queries weekly
