---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server & Power BI for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse analytics"
  - "configure supply chain dashboard with Power BI"
  - "create logistics data warehouse schema"
  - "implement warehouse gravity zones"
  - "build fleet optimization dashboard"
  - "query cross-fact logistics KPIs"
  - "deploy multi-fact star schema for logistics"
  - "integrate warehouse and fleet telemetry data"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and supply chain metrics into a single semantic data layer. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema that enables cross-modal KPI analysis (e.g., correlating warehouse dwell time with fleet idle costs).

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Unified semantic layer linking warehouse, fleet, and supplier data
- Real-time Power BI dashboards (15-minute refresh)
- Warehouse Gravity Zones™ for spatial optimization
- Predictive bottleneck detection
- Fleet triage engine based on telemetry scoring
- Role-based access control with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Developer, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate permissions to create databases and schemas

### Initial Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL Schema:**
```sql
-- Connect to your SQL Server instance in SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema deployment script
-- (Assumes schema.sql file in repository)
:r schema.sql
GO
```

3. **Configure Data Sources:**
```json
// config.json (update with your connection strings)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_feed": "${TELEMATICS_FEED_URL}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Import Power BI Template:**
```powershell
# Open the .pbit template
Start-Process "LogiFleet_Pulse_Master.pbit"
# Connect to your SQL Server when prompted
```

## Data Model Architecture

### Core Fact Tables

**FactWarehouseOperations** - Tracks all warehouse activities:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingTimeMinutes INT,
    GravityZone VARCHAR(20),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Recommended indexes
CREATE NONCLUSTERED INDEX IX_Warehouse_Time 
ON FactWarehouseOperations(TimeKey, WarehouseKey) 
INCLUDE (DwellTimeMinutes, PickRateUnitsPerHour);
```

**FactFleetTrips** - Fleet telemetry and route data:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    MaintenanceScore INT, -- 0-100
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_Fleet_Vehicle_Time 
ON FactFleetTrips(VehicleKey, TimeKey) 
INCLUDE (FuelLiters, IdleTimeMinutes);
```

**FactCrossDock** - Cross-docking operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ProductKey INT NOT NULL,
    TransferTimeMinutes INT,
    TemperatureCompliant BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripID),
    FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripID),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);
```

### Key Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeValue DATETIME2,
    HourOfDay INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    FiscalPeriod VARCHAR(10),
    IsBusinessHour BIT,
    QuarterHourBucket INT -- 0-3 for 15-min intervals
);

-- Populate with time dimension data
INSERT INTO DimTime (TimeKey, DateTimeValue, HourOfDay, DayOfWeek, DayName, IsBusinessHour)
SELECT 
    CAST(FORMAT(DateTimeValue, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    DateTimeValue,
    DATEPART(HOUR, DateTimeValue) AS HourOfDay,
    DATEPART(WEEKDAY, DateTimeValue) AS DayOfWeek,
    DATENAME(WEEKDAY, DateTimeValue) AS DayName,
    CASE WHEN DATEPART(HOUR, DateTimeValue) BETWEEN 8 AND 17 
         AND DATEPART(WEEKDAY, DateTimeValue) BETWEEN 2 AND 6 
         THEN 1 ELSE 0 END AS IsBusinessHour
FROM GenerateDateTimeSequence('2024-01-01', '2027-12-31', 15); -- 15-minute intervals
```

**DimProductGravity** - Products with gravity zone classification:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- 0-100
    PickFrequencyDaily DECIMAL(10,2),
    UnitValue DECIMAL(10,2),
    FragilityIndex INT, -- 1-10
    ReplenishmentLeadTimeDays INT,
    RecommendedZone VARCHAR(20) -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
);

-- Calculate gravity score
UPDATE DimProductGravity
SET GravityScore = (
    (PickFrequencyDaily / NULLIF((SELECT MAX(PickFrequencyDaily) FROM DimProductGravity), 0)) * 40 +
    (UnitValue / NULLIF((SELECT MAX(UnitValue) FROM DimProductGravity), 0)) * 30 +
    (FragilityIndex / 10.0) * 20 +
    (1.0 - (ReplenishmentLeadTimeDays / NULLIF((SELECT MAX(ReplenishmentLeadTimeDays) FROM DimProductGravity), 0))) * 10
) * 100;

UPDATE DimProductGravity
SET RecommendedZone = CASE 
    WHEN GravityScore >= 70 THEN 'High-Gravity'
    WHEN GravityScore >= 40 THEN 'Medium-Gravity'
    ELSE 'Low-Gravity'
END;
```

## Key Queries & Analytics Patterns

### Cross-Fact KPI Analysis

**Correlate warehouse dwell time with fleet idle costs:**
```sql
-- Business question: Which products have high warehouse dwell AND contribute to fleet delays?
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU, p.ProductName
),
FleetImpact AS (
    SELECT 
        p.SKU,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(f.FuelLiters * 1.5) AS IdleFuelCostUSD -- $1.50/liter
    FROM FactFleetTrips f
    JOIN FactCrossDock cd ON f.TripID = cd.OutboundTripKey
    JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTimeValue >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    fi.IdleFuelCostUSD,
    (wd.AvgDwellMinutes * 0.5 + fi.AvgIdleMinutes * 0.5) AS CombinedFrictionScore
FROM WarehouseDwell wd
LEFT JOIN FleetImpact fi ON wd.SKU = fi.SKU
ORDER BY CombinedFrictionScore DESC;
```

### Warehouse Gravity Zone Optimization

**Identify products in wrong gravity zones:**
```sql
SELECT 
    p.SKU,
    p.ProductName,
    p.RecommendedZone,
    w.GravityZone AS CurrentZone,
    AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
    AVG(w.PickRateUnitsPerHour) AS AvgPickRate,
    COUNT(*) AS TotalOperations
FROM FactWarehouseOperations w
JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE p.RecommendedZone != w.GravityZone
  AND w.TimeKey >= CAST(FORMAT(DATEADD(MONTH, -3, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.SKU, p.ProductName, p.RecommendedZone, w.GravityZone
HAVING AVG(w.DwellTimeMinutes) > 120 -- Over 2 hours dwell
ORDER BY AVG(w.DwellTimeMinutes) DESC;
```

### Predictive Bottleneck Detection

**Fleet triage based on maintenance score and cargo value:**
```sql
WITH VehicleRisk AS (
    SELECT 
        v.VehicleID,
        v.VehicleName,
        f.MaintenanceScore,
        f.LoadWeightKG,
        SUM(p.UnitValue * cd.TransferQuantity) AS CargoValueUSD
    FROM FactFleetTrips f
    JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    LEFT JOIN FactCrossDock cd ON f.TripID = cd.OutboundTripKey
    LEFT JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    WHERE f.TripEndTime IS NULL -- Active trips only
    GROUP BY v.VehicleID, v.VehicleName, f.MaintenanceScore, f.LoadWeightKG
)
SELECT 
    VehicleID,
    VehicleName,
    MaintenanceScore,
    CargoValueUSD,
    LoadWeightKG,
    -- Triage priority: (100 - maintenance) * cargo value weight
    (100 - MaintenanceScore) * (CargoValueUSD / NULLIF((SELECT MAX(CargoValueUSD) FROM VehicleRisk), 0)) * 100 AS TriagePriority
FROM VehicleRisk
WHERE MaintenanceScore < 70
ORDER BY TriagePriority DESC;
```

### Temporal Elasticity Simulation

**What-if scenario: Impact of increasing warehouse capacity:**
```sql
-- Historical baseline
DECLARE @CurrentCapacityPct DECIMAL(5,2) = 80;
DECLARE @NewCapacityPct DECIMAL(5,2) = 95;

WITH BaselineMetrics AS (
    SELECT 
        AVG(DwellTimeMinutes) AS AvgDwell,
        AVG(PickRateUnitsPerHour) AS AvgPickRate
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(FORMAT(DATEADD(MONTH, -6, GETDATE()), 'yyyyMMddHHmm') AS INT)
)
SELECT 
    @CurrentCapacityPct AS CurrentCapacity,
    @NewCapacityPct AS ProjectedCapacity,
    AvgDwell AS BaselineDwellMinutes,
    -- Simple projection: dwell increases non-linearly with capacity
    AvgDwell * POWER((@NewCapacityPct / @CurrentCapacityPct), 1.5) AS ProjectedDwellMinutes,
    AvgPickRate AS BaselinePickRate,
    AvgPickRate * (1 - ((@NewCapacityPct - @CurrentCapacityPct) / 100.0) * 0.3) AS ProjectedPickRate
FROM BaselineMetrics;
```

## Stored Procedures for Automation

### Incremental Load Procedure

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouse
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load new warehouse operations since last load
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        DwellTimeMinutes, PickRateUnitsPerHour, PackingTimeMinutes, GravityZone
    )
    SELECT 
        CAST(FORMAT(w.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        geo.GeographyKey,
        p.ProductKey,
        w.OperationType,
        DATEDIFF(MINUTE, w.StartTime, w.EndTime) AS DwellTimeMinutes,
        w.UnitsProcessed / NULLIF(DATEDIFF(HOUR, w.StartTime, w.EndTime), 0) AS PickRateUnitsPerHour,
        DATEDIFF(MINUTE, w.PackStartTime, w.PackEndTime) AS PackingTimeMinutes,
        p.RecommendedZone AS GravityZone
    FROM ExternalWMS.dbo.WarehouseOperations w
    JOIN DimGeography geo ON w.WarehouseCode = geo.WarehouseCode
    JOIN DimProductGravity p ON w.SKU = p.SKU
    WHERE w.OperationDateTime > @LastLoadTime
      AND w.OperationDateTime <= GETDATE();
    
    -- Update last load timestamp
    UPDATE ETL_Control 
    SET LastLoadTime = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Automated Alerting Procedure

```sql
CREATE PROCEDURE sp_GenerateLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert: High dwell time in high-gravity zones
    INSERT INTO AlertQueue (AlertType, Severity, Message, GeneratedTime)
    SELECT 
        'HIGH_DWELL_CRITICAL_ZONE' AS AlertType,
        'High' AS Severity,
        'SKU ' + p.SKU + ' has avg dwell of ' + 
        CAST(AVG(w.DwellTimeMinutes) AS VARCHAR(10)) + 
        ' min in High-Gravity zone' AS Message,
        GETDATE() AS GeneratedTime
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.GravityZone = 'High-Gravity'
      AND w.TimeKey >= CAST(FORMAT(DATEADD(HOUR, -24, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY p.SKU
    HAVING AVG(w.DwellTimeMinutes) > 180; -- 3 hours
    
    -- Alert: Fleet high idle time
    INSERT INTO AlertQueue (AlertType, Severity, Message, GeneratedTime)
    SELECT 
        'FLEET_IDLE_EXCESSIVE' AS AlertType,
        'Medium' AS Severity,
        'Vehicle ' + v.VehicleName + ' idle time at ' + 
        CAST((SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, f.TripStartTime, f.TripEndTime)), 0)) AS VARCHAR(10)) + 
        '% of trip duration' AS Message,
        GETDATE() AS GeneratedTime
    FROM FactFleetTrips f
    JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE f.TimeKey >= CAST(FORMAT(DATEADD(HOUR, -24, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY v.VehicleID, v.VehicleName
    HAVING (SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, f.TripStartTime, f.TripEndTime)), 0)) > 15;
    
    -- Send alerts via configured channels
    EXEC sp_SendAlertNotifications;
END;
GO

-- Schedule to run every 15 minutes
EXEC sp_add_job @job_name = 'LogiFleet_Alert_Generator';
EXEC sp_add_jobstep @job_name = 'LogiFleet_Alert_Generator', 
    @step_name = 'Generate_Alerts',
    @command = 'EXEC sp_GenerateLogisticsAlerts';
EXEC sp_add_schedule @schedule_name = 'Every15Min', @freq_type = 4, 
    @freq_interval = 1, @freq_subday_type = 4, @freq_subday_interval = 15;
```

## Power BI Configuration

### DAX Measures for Cross-Fact KPIs

**Total Cost of Delay:**
```dax
TotalCostOfDelay = 
VAR WarehouseDelayCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * 0.5 // $0.50 per minute
    )
VAR FleetDelayCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] * 2.0 // $2.00 per minute (includes fuel + driver)
    )
RETURN
    WarehouseDelayCost + FleetDelayCost
```

**Fleet Utilization Rate:**
```dax
FleetUtilization = 
DIVIDE(
    SUMX(
        FactFleetTrips,
        DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE) 
        - FactFleetTrips[IdleTimeMinutes]
    ),
    SUMX(
        FactFleetTrips,
        DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE)
    ),
    0
)
```

**Gravity Zone Compliance:**
```dax
GravityZoneCompliance = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR CompliantOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[GravityZone] = RELATED(DimProductGravity[RecommendedZone])
    )
RETURN
    DIVIDE(CompliantOps, TotalOps, 0)
```

### Row-Level Security Setup

```dax
-- Create role "Regional Manager"
-- RLS filter on DimGeography table:
[Region] = USERPRINCIPALNAME()
```

In SQL Server:
```sql
-- User-to-region mapping table
CREATE TABLE SecurityUserRegion (
    UserEmail VARCHAR(255) PRIMARY KEY,
    Region VARCHAR(100)
);

INSERT INTO SecurityUserRegion VALUES 
    ('manager.east@company.com', 'East'),
    ('manager.west@company.com', 'West');
```

## Integration Patterns

### External API Data Ingestion (Weather)

```sql
-- Create external data source for weather API
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${WEATHER_API_ENDPOINT}'
);

-- Stored procedure to fetch and parse weather data
CREATE PROCEDURE sp_IngestWeatherData
AS
BEGIN
    -- Fetch weather delays via REST API (using CLR or external tool)
    -- Parse and insert into staging table
    MERGE INTO FactWeatherDelays AS target
    USING StagingWeatherData AS source
    ON target.TimeKey = source.TimeKey 
       AND target.GeographyKey = source.GeographyKey
    WHEN MATCHED THEN 
        UPDATE SET DelayMinutes = source.DelayMinutes
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, GeographyKey, DelayMinutes)
        VALUES (source.TimeKey, source.GeographyKey, source.DelayMinutes);
END;
GO
```

### Telematics Feed Integration (Streaming)

```sql
-- Create table for real-time telemetry
CREATE TABLE StagingTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    FuelLevel DECIMAL(5,2),
    TirePressure DECIMAL(5,2),
    EngineTemp DECIMAL(5,2),
    InsertedAt DATETIME2 DEFAULT GETDATE()
);

-- Process and aggregate into FactFleetTrips hourly
CREATE PROCEDURE sp_AggregateTelemetryToTrips
AS
BEGIN
    INSERT INTO FactFleetTrips (TimeKey, VehicleKey, RouteKey, FuelLiters, MaintenanceScore)
    SELECT 
        CAST(FORMAT(s.Timestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        v.VehicleKey,
        r.RouteKey,
        AVG(s.FuelLevel) AS FuelLiters,
        CASE 
            WHEN AVG(s.TirePressure) < 30 THEN 60
            WHEN AVG(s.EngineTemp) > 95 THEN 70
            ELSE 90
        END AS MaintenanceScore
    FROM StagingTelemetry s
    JOIN DimVehicle v ON s.VehicleID = v.VehicleID
    JOIN DimRoute r ON ST_WITHIN(geography::Point(s.Latitude, s.Longitude, 4326), r.RouteGeom)
    WHERE s.InsertedAt >= DATEADD(HOUR, -1, GETDATE())
    GROUP BY CAST(FORMAT(s.Timestamp, 'yyyyMMddHHmm') AS INT), v.VehicleKey, r.RouteKey;
    
    -- Archive staged data
    DELETE FROM StagingTelemetry 
    WHERE InsertedAt < DATEADD(HOUR, -2, GETDATE());
END;
GO
```

## Troubleshooting

### Issue: Slow cross-fact queries

**Solution:** Ensure covering indexes on join keys
```sql
-- Add composite indexes
CREATE NONCLUSTERED INDEX IX_CrossDock_Products 
ON FactCrossDock(ProductKey, TimeKey) 
INCLUDE (TransferTimeMinutes, TemperatureCompliant);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Power BI refresh failures

**Check connection string and gateway:**
```powershell
# Test SQL connection from Power BI Gateway machine
Test-NetConnection -ComputerName "${SQL_SERVER_HOST}" -Port 1433

# Check gateway logs
Get-Content "C:\Program Files\On-premises data gateway\GatewayLogs.txt" -Tail 50
```

**Verify service account permissions:**
```sql
-- Grant read access to service account
CREATE USER [DOMAIN\PowerBIGateway] FOR LOGIN [DOMAIN\PowerBIGateway];
ALTER ROLE db_datareader ADD MEMBER [DOMAIN\PowerBIGateway];
```

### Issue: Gravity score calculation errors

**Validate data ranges:**
```sql
-- Check for null or outlier values
SELECT 
    COUNT(*) AS TotalProducts,
    SUM(CASE WHEN PickFrequencyDaily IS NULL THEN 1 ELSE 0 END) AS NullPickFreq,
    SUM(CASE WHEN GravityScore < 0 OR GravityScore > 100 THEN 1 ELSE 0 END) AS OutOfRangeScores
FROM DimProductGravity;

-- Recalculate for products with invalid scores
UPDATE DimProductGravity
SET GravityScore = NULL,
    RecommendedZone = NULL
WHERE GravityScore < 0 OR GravityScore > 100;

-- Re-run gravity calculation procedure
EXEC sp_CalculateProductGravity;
```

### Issue: Time dimension missing keys

**Regenerate time dimension for extended date range:**
```sql
-- Procedure to populate DimTime
CREATE PROCEDURE sp_PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, DateTimeValue, HourOfDay, DayOfWeek, DayName, QuarterHourBucket, IsBusinessHour)
        VALUES (
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime) / 15,
            CASE WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
                 AND DATEPART(WEEKDAY, @CurrentDateTime) BETWEEN 2 AND 6 
                 THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END;
END;
GO

-- Execute for 2024-2030
EXEC sp_PopulateTimeDimension '2024-01-01', '2030-12-31';
```

## Best Practices

1. **Partition large fact tables by time:** Improve query performance for recent data
```sql
ALTER DATABASE LogiFleetPulse SET ALLOW_SNAPSHOT_ISOLATION ON;
CREATE PARTITION FUNCTION PF_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (202401010000, 202404010000, 202407010000, 202410010000);
```

2. **Use columnstore indexes for analytical queries:**
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_WarehouseOperations
ON FactWarehouseOperations (TimeKey, WarehouseKey, ProductKey, DwellTimeMinutes, PickRateUnitsPerHour);
```

3. **Implement incremental refresh in Power BI:**
   - Set up RangeStart and RangeEnd parameters
   - Configure refresh policy (e.g., last 3 months incremental, rest historical)

4. **Monitor query performance with Extended Events:**
```sql
CREATE EVENT SESSION LogiFleet_QueryMonitor
ON SERVER
ADD EVENT sqlserver.sql_statement_completed
(WHERE duration > 5000000) -- 5 seconds
ADD TARGET package0.event_file(SET filename=N'LogiFleet_Queries.xel');

ALTER EVENT SESSION LogiFleet_QueryMonitor ON SERVER STATE = START;
```

5. **Use environment-specific configuration:**
```json
// config.dev.json
{
  "environment": "development",
  "refresh_interval_minutes": 30,
  "enable_debug_logging": true
}

// config.prod.json
{
  "environment": "production",
  "refresh_interval_minutes": 15,
  "enable_debug_logging": false
}
```
