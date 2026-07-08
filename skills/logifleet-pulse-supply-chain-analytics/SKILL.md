---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure multi-fact star schema for fleet and warehouse
  - implement supply chain kpi dashboard
  - create logistics intelligence platform
  - build warehouse and fleet analytics with sql server
  - integrate logifleet pulse with power bi
  - optimize supply chain data modeling
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a multi-modal logistics intelligence platform that combines:
- **MS SQL Server data warehouse** with multi-fact star schema architecture
- **Power BI dashboards** for warehouse operations, fleet management, and supply chain KPIs
- **Cross-fact KPI harmonization** linking inventory, fleet telemetry, and external data sources
- **Time-phased dimensions** for temporal analysis at 15-minute granularity
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization

The system integrates data from WMS (Warehouse Management Systems), GPS/telematics, supplier portals, weather APIs, and customer orders into a unified semantic layer.

## Installation & Setup

### Prerequisites

- **MS SQL Server 2019+** (Standard or Enterprise edition recommended)
- **Power BI Desktop** (latest version)
- **SSMS (SQL Server Management Studio)** for database administration
- Access to source systems: WMS, TMS, ERP, telematics APIs

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Create new database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Hour INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    DayOfWeek NVARCHAR(10),
    DayOfMonth INT,
    Month INT,
    Quarter INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
)
GO

CREATE NONCLUSTERED INDEX IX_DimTime_FullDateTime 
ON DimTime(FullDateTime)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeCode NVARCHAR(50) NOT NULL,
    NodeName NVARCHAR(200),
    NodeType NVARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Point'
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1
)
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    RequiresColdStorage BIT,
    UnitValue DECIMAL(18,2),
    GravityScore DECIMAL(5,2) -- Calculated based on velocity, value, fragility
)
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200),
    Country NVARCHAR(100),
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeDaysStdDev DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    IsActive BIT DEFAULT 1
)
GO

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) NOT NULL UNIQUE,
    VehicleType NVARCHAR(50), -- 'Box Truck', 'Semi', 'Refrigerated', etc.
    Capacity DECIMAL(10,2),
    FuelType NVARCHAR(30),
    YearManufactured INT,
    MaintenanceScore DECIMAL(5,2), -- Updated by telemetry
    IsActive BIT DEFAULT 1
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplier(SupplierKey),
    OperationType NVARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT, -- Time from receiving to shipping
    ZoneCode NVARCHAR(20), -- Storage zone
    PickTimeSeconds INT,
    PackTimeSeconds INT,
    OperatorID NVARCHAR(50),
    BatchNumber NVARCHAR(50)
)
GO

CREATE NONCLUSTERED INDEX IX_FactWH_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey)
INCLUDE (Quantity, DwellTimeMinutes)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    FleetKey INT FOREIGN KEY REFERENCES DimFleet(FleetKey),
    OriginKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DriverID NVARCHAR(50),
    TripDistance DECIMAL(10,2), -- km
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumed DECIMAL(10,2), -- liters
    LoadWeight DECIMAL(10,2), -- kg
    DelayMinutes INT,
    DelayReason NVARCHAR(200), -- 'Weather', 'Traffic', 'Mechanical', 'Loading'
    AvgSpeed DECIMAL(5,2),
    HarshBrakingEvents INT,
    EngineHealthScore DECIMAL(5,2) -- From telemetry
)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeVehicle 
ON FactFleetTrips(TimeKey, FleetKey)
INCLUDE (TripDistance, FuelConsumed, IdleTimeMinutes)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT,
    DockDwellMinutes INT, -- Time between inbound arrival and outbound departure
    TransferZone NVARCHAR(20)
)
GO
```

### Step 2: Create ETL Stored Procedures

```sql
-- Incremental loading procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assuming external WMS data is staged in a table
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, 
        ZoneCode, PickTimeSeconds, PackTimeSeconds,
        OperatorID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.ReceiveTime, stg.ShipTime) AS DwellTimeMinutes,
        stg.ZoneCode,
        stg.PickTimeSeconds,
        stg.PackTimeSeconds,
        stg.OperatorID,
        stg.BatchNumber
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON DATEADD(MINUTE, 
        (DATEPART(MINUTE, stg.OperationTime) / 15) * 15 - DATEPART(MINUTE, stg.OperationTime),
        stg.OperationTime) = t.FullDateTime
    INNER JOIN DimGeography g ON stg.WarehouseCode = g.NodeCode
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplier s ON stg.SupplierCode = s.SupplierCode
    WHERE stg.OperationTime > @LastLoadTime
    
    -- Log load time
    UPDATE ETLControl 
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations'
END
GO

-- Gravity score calculation procedure
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    -- Calculate gravity based on 30-day velocity, value, and handling requirements
    UPDATE p
    SET GravityScore = (
        (velocity.PicksPerDay * 0.5) +  -- 50% weight on velocity
        (LOG(p.UnitValue + 1) * 0.3) +  -- 30% weight on value (log scale)
        (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END * 0.2) -- 20% on fragility
    )
    FROM DimProduct p
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / 30 AS PicksPerDay
        FROM FactWarehouseOperations
        WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
            AND OperationType = 'Picking'
        GROUP BY ProductKey
    ) velocity ON p.ProductKey = velocity.ProductKey
END
GO
```

### Step 3: Configure Power BI Connection

1. Open `LogiFleet_Pulse_Master.pbit` from the repository
2. When prompted, enter SQL Server connection details:
   - **Server**: `your-sql-server.database.windows.net` or `localhost\SQLEXPRESS`
   - **Database**: `LogiFleetPulse`
   - **Authentication**: Windows Authentication or SQL Server (use env vars for credentials)

3. The template auto-detects relationships based on foreign keys

### Step 4: Set Up Data Refresh

```sql
-- Create scheduled job for ETL (via SQL Server Agent)
USE msdb
GO

EXEC sp_add_job 
    @job_name = 'LogiFleet_ETL_Incremental',
    @enabled = 1,
    @description = 'Incremental load of warehouse and fleet data'
GO

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_ETL_Incremental',
    @step_name = 'Load Warehouse Operations',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC usp_LoadWarehouseOperations @LastLoadTime = (SELECT LastLoadTime FROM ETLControl WHERE TableName = ''FactWarehouseOperations'')'
GO

EXEC sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15
GO

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_ETL_Incremental',
    @schedule_name = 'Every15Minutes'
GO
```

## Configuration

### Environment Variables

Set these in your deployment environment:

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="your-server.database.windows.net"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USER="your-username"
export LOGIFLEET_SQL_PASSWORD="your-password"

# External API keys (for weather, traffic enrichment)
export LOGIFLEET_WEATHER_API_KEY="your-weather-api-key"
export LOGIFLEET_TRAFFIC_API_KEY="your-traffic-api-key"

# Alert configuration
export LOGIFLEET_ALERT_EMAIL_SMTP="smtp.yourcompany.com"
export LOGIFLEET_ALERT_EMAIL_FROM="alerts@yourcompany.com"
```

### config.json (for external data source integration)

```json
{
  "dataSources": {
    "wms": {
      "type": "sql",
      "server": "${LOGIFLEET_WMS_SERVER}",
      "database": "${LOGIFLEET_WMS_DATABASE}",
      "refreshIntervalMinutes": 15
    },
    "telematics": {
      "type": "rest",
      "endpoint": "https://api.telematics-provider.com/v1/vehicles",
      "apiKey": "${LOGIFLEET_TELEMATICS_API_KEY}",
      "refreshIntervalMinutes": 5
    },
    "weather": {
      "type": "rest",
      "endpoint": "https://api.weatherservice.com/v2/conditions",
      "apiKey": "${LOGIFLEET_WEATHER_API_KEY}",
      "refreshIntervalMinutes": 30
    }
  },
  "alerts": {
    "fleetIdleThresholdPercent": 15,
    "dwellTimeWarningHours": 72,
    "maintenanceScoreThreshold": 60
  }
}
```

## Key SQL Queries & Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time

```sql
-- Find correlation between warehouse dwell time and fleet idle time
-- for the same product shipments
SELECT 
    p.Category,
    AVG(wh.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    COUNT(DISTINCT wh.OperationKey) AS TotalShipments,
    CORR(wh.DwellTimeMinutes, ft.IdleTimeMinutes) AS Correlation
FROM FactWarehouseOperations wh
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON 
    ft.OriginKey = wh.WarehouseKey 
    AND ft.TimeKey BETWEEN wh.TimeKey AND wh.TimeKey + 100 -- Within ~24 hours
WHERE wh.OperationType = 'Shipping'
    AND wh.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.Category
HAVING COUNT(DISTINCT wh.OperationKey) > 100
ORDER BY Correlation DESC
```

### Warehouse Gravity Zone Analysis

```sql
-- Identify products in wrong gravity zones (high-velocity in low-access zones)
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wh.ZoneCode,
    zone_stats.AvgPickTimeSeconds,
    COUNT(*) AS PickCount,
    AVG(wh.PickTimeSeconds) AS ActualPickTime,
    CASE 
        WHEN p.GravityScore > 70 AND zone_stats.AvgPickTimeSeconds > 30 
        THEN 'RELOCATE TO HIGH-GRAVITY ZONE'
        ELSE 'OK'
    END AS Recommendation
FROM FactWarehouseOperations wh
INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
LEFT JOIN (
    SELECT ZoneCode, AVG(PickTimeSeconds) AS AvgPickTimeSeconds
    FROM FactWarehouseOperations
    WHERE OperationType = 'Picking'
    GROUP BY ZoneCode
) zone_stats ON wh.ZoneCode = zone_stats.ZoneCode
WHERE wh.OperationType = 'Picking'
    AND wh.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.SKU, p.ProductName, p.GravityScore, wh.ZoneCode, zone_stats.AvgPickTimeSeconds
HAVING COUNT(*) > 10
ORDER BY p.GravityScore DESC
```

### Predictive Fleet Maintenance Triage

```sql
-- Prioritize vehicle maintenance by revenue impact
WITH FleetHealth AS (
    SELECT 
        f.VehicleID,
        f.VehicleType,
        AVG(ft.EngineHealthScore) AS AvgHealthScore,
        SUM(ft.LoadWeight * p.UnitValue / 1000) AS TotalCargoValue, -- Approx value
        COUNT(*) AS TripCount,
        SUM(ft.DelayMinutes) AS TotalDelayMin
    FROM FactFleetTrips ft
    INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
    LEFT JOIN FactWarehouseOperations wh ON wh.TimeKey = ft.TimeKey
    LEFT JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    WHERE ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY f.VehicleID, f.VehicleType
)
SELECT 
    VehicleID,
    VehicleType,
    AvgHealthScore,
    TotalCargoValue,
    TripCount,
    TotalDelayMin,
    (100 - AvgHealthScore) * (TotalCargoValue / 1000000) AS MaintenancePriorityScore
FROM FleetHealth
WHERE AvgHealthScore < 80
ORDER BY MaintenancePriorityScore DESC
```

### Time-Phased Scenario Analysis

```sql
-- Simulate impact of 95% warehouse capacity vs 80%
DECLARE @CurrentCapacity DECIMAL(5,2) = 0.80
DECLARE @SimulatedCapacity DECIMAL(5,2) = 0.95

SELECT 
    'Current State' AS Scenario,
    AVG(DwellTimeMinutes) AS AvgDwellTime,
    AVG(PickTimeSeconds) AS AvgPickTime,
    COUNT(*) AS TotalOperations
FROM FactWarehouseOperations
WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)

UNION ALL

SELECT 
    'Simulated 95% Capacity' AS Scenario,
    AVG(DwellTimeMinutes * (1 + (@SimulatedCapacity - @CurrentCapacity) * 2)) AS AvgDwellTime, -- Empirical multiplier
    AVG(PickTimeSeconds * (1 + (@SimulatedCapacity - @CurrentCapacity) * 1.5)) AS AvgPickTime,
    COUNT(*) AS TotalOperations
FROM FactWarehouseOperations
WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
```

## Power BI DAX Measures

### Composite KPI: Fleet Efficiency Index

```dax
Fleet Efficiency Index = 
VAR AvgIdlePercent = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[TripDurationMinutes]),
        0
    )
VAR FuelEfficiency = 
    DIVIDE(
        SUM(FactFleetTrips[TripDistance]),
        SUM(FactFleetTrips[FuelConsumed]),
        0
    )
VAR HealthScore = AVERAGE(FactFleetTrips[EngineHealthScore])

RETURN 
    (1 - AvgIdlePercent) * 0.4 +  -- 40% weight on low idle time
    (FuelEfficiency / 15) * 0.3 +  -- 30% weight on fuel efficiency (normalized by 15 km/L baseline)
    (HealthScore / 100) * 0.3      -- 30% weight on engine health
```

### Warehouse Throughput Velocity

```dax
Throughput Velocity = 
DIVIDE(
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[OperationType] = "Shipping"
    ),
    CALCULATE(
        DISTINCTCOUNT(DimTime[DayOfMonth]),
        ALLSELECTED(DimTime)
    ),
    0
)
```

### Predictive Bottleneck Heatmap (using historical patterns)

```dax
Bottleneck Risk Score = 
VAR CurrentHourPicks = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[OperationType] = "Picking",
        DimTime[Hour] = HOUR(NOW())
    )
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[PickTimeSeconds]),
        DimTime[Hour] = HOUR(NOW()),
        DATESBETWEEN(DimTime[FullDateTime], TODAY()-30, TODAY())
    )
VAR CurrentAvg = AVERAGE(FactWarehouseOperations[PickTimeSeconds])

RETURN 
IF(
    CurrentAvg > HistoricalAvg * 1.2 && CurrentHourPicks > 50,
    1,  -- High risk
    0   -- Normal
)
```

## Common Troubleshooting

### Issue: Power BI dashboard shows no data

**Solution**: Check SQL Server connection and verify data exists in fact tables

```sql
-- Verify data in fact tables
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS RowCount FROM FactWarehouseOperations
UNION ALL
SELECT 'FactFleetTrips', COUNT(*) FROM FactFleetTrips
UNION ALL
SELECT 'DimTime', COUNT(*) FROM DimTime
UNION ALL
SELECT 'DimProduct', COUNT(*) FROM DimProduct

-- Check latest loaded time
SELECT TimeKey, FullDateTime 
FROM DimTime 
WHERE TimeKey = (SELECT MAX(TimeKey) FROM FactWarehouseOperations)
```

### Issue: Slow query performance on cross-fact KPIs

**Solution**: Add covering indexes and consider indexed views

```sql
-- Create indexed view for frequent cross-fact query
CREATE VIEW vw_WarehouseFleetJoin
WITH SCHEMABINDING
AS
SELECT 
    wh.TimeKey,
    wh.ProductKey,
    wh.WarehouseKey,
    wh.DwellTimeMinutes,
    ft.FleetKey,
    ft.IdleTimeMinutes,
    ft.TripDistance
FROM dbo.FactWarehouseOperations wh
INNER JOIN dbo.FactFleetTrips ft ON ft.OriginKey = wh.WarehouseKey
WHERE wh.OperationType = 'Shipping'
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_WarehouseFleetJoin
ON vw_WarehouseFleetJoin(TimeKey, ProductKey, FleetKey)
GO
```

### Issue: Gravity scores not updating

**Solution**: Ensure scheduled job is running and execute manually

```sql
-- Check last execution
SELECT 
    job.name,
    activity.run_requested_date,
    activity.last_executed_step_date,
    activity.stop_execution_date
FROM msdb.dbo.sysjobs job
LEFT JOIN msdb.dbo.sysjobactivity activity ON job.job_id = activity.job_id
WHERE job.name = 'LogiFleet_ETL_Incremental'

-- Execute manually
EXEC usp_UpdateProductGravityScores
```

### Issue: Power BI template doesn't recognize relationships

**Solution**: Manually recreate relationships in Power BI model view

1. Go to **Model** view in Power BI Desktop
2. Drag from `FactWarehouseOperations[TimeKey]` to `DimTime[TimeKey]`
3. Set cardinality to **Many-to-One**
4. Set cross-filter direction to **Single**
5. Repeat for all fact-to-dimension relationships

## Advanced Patterns

### Row-Level Security (RLS) for Multi-Tenant Warehouses

```dax
-- Create role in Power BI: "WarehouseManager"
-- Add filter on DimGeography table:
[GeographyKey] IN 
    (
        SELECT GeographyKey 
        FROM UserWarehouseAccess 
        WHERE UserEmail = USERPRINCIPALNAME()
    )
```

```sql
-- Create SQL table for user access mapping
CREATE TABLE UserWarehouseAccess (
    UserEmail NVARCHAR(200),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey)
)

-- Insert sample mappings
INSERT INTO UserWarehouseAccess VALUES ('manager1@company.com', 1)
INSERT INTO UserWarehouseAccess VALUES ('manager1@company.com', 2)
INSERT INTO UserWarehouseAccess VALUES ('manager2@company.com', 3)
```

### Automated Alerting via Stored Procedure

```sql
CREATE PROCEDURE usp_SendAlerts
AS
BEGIN
    -- Find fleet vehicles with critical idle time
    DECLARE @AlertBody NVARCHAR(MAX)
    
    SELECT @AlertBody = STRING_AGG(
        CONCAT(VehicleID, ' - Idle: ', IdlePercent, '% (Threshold: 15%)'),
        CHAR(13) + CHAR(10)
    )
    FROM (
        SELECT 
            f.VehicleID,
            CAST(SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.TripDurationMinutes), 0) AS DECIMAL(5,1)) AS IdlePercent
        FROM FactFleetTrips ft
        INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
        WHERE ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -1, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY f.VehicleID
        HAVING SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.TripDurationMinutes), 0) > 15
    ) alerts
    
    IF @AlertBody IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = 'fleet-manager@company.com',
            @subject = 'LogiFleet ALERT: High Fleet Idle Time',
            @body = @AlertBody
    END
END
GO
```

### External Data Enrichment (Weather API)

```sql
-- Create external table for weather data (using Polybase or linked server)
CREATE EXTERNAL TABLE ExternalWeather (
    LocationID NVARCHAR(50),
    ObservationTime DATETIME2,
    Condition NVARCHAR(100),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2)
)
WITH (
    LOCATION = '/weather/observations/',
    DATA_SOURCE = WeatherAPISource,
    FILE_FORMAT = JSONFormat
)
GO

-- Join weather with fleet trips to analyze delay correlation
SELECT 
    ft.TripKey,
    ft.DelayMinutes,
    w.Condition,
    w.Precipitation,
    CASE 
        WHEN w.Condition IN ('Rain', 'Snow', 'Fog') THEN 'Weather-Related'
        ELSE 'Other'
    END AS DelayCategory
FROM FactFleetTrips ft
INNER JOIN DimGeography g ON ft.DestinationKey = g.GeographyKey
LEFT JOIN ExternalWeather w ON 
    g.City = w.LocationID 
    AND ABS(DATEDIFF(MINUTE, ft.TimeKey, w.ObservationTime)) < 60
WHERE ft.DelayMinutes > 15
```

## Best Practices

1. **Partition fact tables by time**: Use SQL Server table partitioning on `TimeKey` for tables with >10M rows
2. **Use columnstore indexes**: Add clustered columnstore indexes on fact tables for analytical workloads
3. **Implement incremental refresh in Power BI**: Configure parameters for `RangeStart` and `RangeEnd`
4. **Cache frequently used aggregations**: Create indexed views for common KPI calculations
5. **Monitor ETL job duration**: Set alerts if load time exceeds 15-minute window
6. **Validate data quality**: Add constraints and triggers to enforce referential integrity
7. **Document calculated measures**: Use description field in Power BI for all DAX measures
8. **Version control Power BI templates**: Store `.pbit` files in git with semantic versioning
