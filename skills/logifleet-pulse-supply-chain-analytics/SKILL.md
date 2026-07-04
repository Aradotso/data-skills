---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics, fleet tracking, warehouse operations, and cross-modal supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - configure warehouse and fleet data model
  - create power bi logistics dashboard
  - implement multi-fact star schema for supply chain
  - build real-time fleet tracking system
  - design warehouse gravity zone optimization
  - query cross-fact logistics KPIs
  - set up predictive bottleneck detection
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics platform for supply chain operations, combining:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, inventory, and external data
- **MS SQL Server backend** for transactional data integrity and complex cross-fact queries
- **Power BI dashboards** for real-time visualization and decision support
- **Predictive analytics** for bottleneck detection, fleet maintenance, and warehouse optimization
- **Cross-modal integration** between WMS, TMS, telematics, and external APIs

The platform harmonizes disparate logistics data sources into a unified semantic layer, enabling queries like "Show shipments delayed by weather from cold-storage zones with >72hr dwell time."

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, fleet telematics, ERP systems

### Database Schema Deployment

1. **Clone the repository** (implied structure):

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema**:

```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema creation script
-- (Run scripts/01_create_schema.sql from repository)
```

3. **Configure connection string** in `config.json`:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "sql",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${FLEET_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

## Core Data Model Architecture

### Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingTimeSeconds INT,
    OperatorKey INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- FactFleetTrips: Fleet telemetry and route data
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayReasonKey INT,
    CONSTRAINT FK_Fleet_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

-- FactCrossDock: Direct transfers without long-term storage
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    ArrivalTimeKey INT NOT NULL,
    DepartureTimeKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ProductKey INT NOT NULL,
    QuantityUnits INT,
    TransferTimeMinutes INT,
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfYear INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10),
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    FifteenMinuteBucket INT, -- 0-95 (96 buckets per day)
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10)
);

-- DimProductGravity: Warehouse gravity zone assignment
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(255),
    CategoryHierarchy VARCHAR(500),
    GravityScore DECIMAL(5,2), -- Higher = faster moving, higher value
    VelocityClass VARCHAR(20), -- 'FAST', 'MEDIUM', 'SLOW'
    Fragility VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW'
    OptimalZone VARCHAR(50), -- Recommended storage zone
    LastRecalcDate DATE
);

-- DimWarehouse: Warehouse locations and zones
CREATE TABLE DimWarehouse (
    WarehouseKey INT PRIMARY KEY,
    WarehouseID VARCHAR(50) UNIQUE,
    WarehouseName VARCHAR(255),
    ZoneID VARCHAR(50),
    ZoneType VARCHAR(50), -- 'GRAVITY_HIGH', 'COLD_STORAGE', 'BULK', etc.
    GeographyKey INT,
    Capacity INT,
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- DimVehicle: Fleet vehicle master
CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY,
    VehicleID VARCHAR(50) UNIQUE,
    VehicleType VARCHAR(50), -- 'TRUCK', 'VAN', 'TRAILER'
    Capacity INT,
    FuelType VARCHAR(20),
    LastMaintenanceDate DATE,
    TriagePriority INT -- Auto-calculated based on telemetry
);
```

## Key Queries & Patterns

### Cross-Fact KPI Harmonization

```sql
-- Link warehouse dwell time with fleet delays
SELECT 
    dt.Date,
    dp.CategoryHierarchy,
    dw.ZoneType,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleTime,
    COUNT(DISTINCT fft.TripKey) AS AffectedTrips,
    SUM(fft.FuelConsumedGallons * 3.50) AS EstimatedFuelCost -- $3.50/gallon
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
LEFT JOIN FactFleetTrips fft ON fft.StartTimeKey = dt.TimeKey 
    AND fft.RouteKey IN (
        SELECT RouteKey FROM BridgeRouteWarehouse 
        WHERE WarehouseKey = fwo.WarehouseKey
    )
WHERE dt.Date >= DATEADD(DAY, -30, GETDATE())
    AND fwo.DwellTimeMinutes > 72 * 60 -- >72 hours
GROUP BY dt.Date, dp.CategoryHierarchy, dw.ZoneType
ORDER BY AvgDwellTime DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouse zones approaching capacity with high dwell time
WITH ZoneMetrics AS (
    SELECT 
        dw.WarehouseKey,
        dw.WarehouseName,
        dw.ZoneID,
        dw.Capacity,
        COUNT(DISTINCT fwo.ProductKey) AS CurrentSKUCount,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
        STDEV(fwo.DwellTimeMinutes) AS DwellTimeStdDev
    FROM FactWarehouseOperations fwo
    INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
    WHERE fwo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY dw.WarehouseKey, dw.WarehouseName, dw.ZoneID, dw.Capacity
)
SELECT 
    *,
    (CAST(CurrentSKUCount AS FLOAT) / Capacity) AS CapacityUtilization,
    CASE 
        WHEN (CAST(CurrentSKUCount AS FLOAT) / Capacity) > 0.90 
            AND AvgDwellTime > 48 * 60 THEN 'CRITICAL'
        WHEN (CAST(CurrentSKUCount AS FLOAT) / Capacity) > 0.80 
            AND AvgDwellTime > 36 * 60 THEN 'WARNING'
        ELSE 'NORMAL'
    END AS BottleneckRisk
FROM ZoneMetrics
WHERE (CAST(CurrentSKUCount AS FLOAT) / Capacity) > 0.75
ORDER BY CapacityUtilization DESC, AvgDwellTime DESC;
```

### Fleet Adaptive Triage Scoring

```sql
-- Calculate fleet maintenance priority based on telemetry + cargo value
WITH VehicleHealth AS (
    SELECT 
        dv.VehicleKey,
        dv.VehicleID,
        dv.LastMaintenanceDate,
        DATEDIFF(DAY, dv.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaint,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(fft.DistanceMiles) AS TotalMiles,
        COUNT(fft.TripKey) AS TripCount
    FROM DimVehicle dv
    INNER JOIN FactFleetTrips fft ON dv.VehicleKey = fft.VehicleKey
    WHERE fft.StartTimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY dv.VehicleKey, dv.VehicleID, dv.LastMaintenanceDate
),
CargoValue AS (
    SELECT 
        fft.VehicleKey,
        AVG(dp.GravityScore) AS AvgCargoGravity -- Proxy for value
    FROM FactFleetTrips fft
    INNER JOIN FactWarehouseOperations fwo ON fft.RouteKey = fwo.WarehouseKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    GROUP BY fft.VehicleKey
)
SELECT 
    vh.*,
    cv.AvgCargoGravity,
    -- Triage score: maintenance urgency weighted by cargo importance
    (
        (vh.DaysSinceLastMaint / 30.0) * 40 + -- 40% weight for time since maintenance
        (vh.TotalMiles / 1000.0) * 30 + -- 30% weight for total miles
        (cv.AvgCargoGravity / 100.0) * 30 -- 30% weight for cargo value
    ) AS TriageScore
FROM VehicleHealth vh
LEFT JOIN CargoValue cv ON vh.VehicleKey = cv.VehicleKey
ORDER BY TriageScore DESC;
```

## Power BI Integration

### Connecting to SQL Server

1. Open `LogiFleet_Pulse_Master.pbit` template
2. When prompted, enter parameters:

```
Server: ${SQL_SERVER_HOST}
Database: LogiFleetPulse
```

3. Choose Import or DirectQuery mode (DirectQuery recommended for real-time)

### DAX Measures for Cross-Fact Analysis

```dax
-- Composite KPI: Warehouse efficiency impact on fleet cost
Fleet Cost per Dwell Hour = 
VAR TotalFleetCost = 
    SUM(FactFleetTrips[FuelConsumedGallons]) * 3.50 +
    SUM(FactFleetTrips[IdleTimeMinutes]) / 60 * 50 -- $50/hr labor
VAR TotalDwellHours = 
    SUM(FactWarehouseOperations[DwellTimeMinutes]) / 60
RETURN
    DIVIDE(TotalFleetCost, TotalDwellHours, 0)

-- Gravity zone compliance rate
Gravity Zone Compliance % = 
VAR ActualPlacements = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            RELATED(DimWarehouse[ZoneType]) = RELATED(DimProductGravity[OptimalZone])
        )
    )
VAR TotalPlacements = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(ActualPlacements, TotalPlacements, 0) * 100

-- Predictive delay probability (using historical patterns)
Delay Risk Score = 
CALCULATE(
    AVERAGEX(
        FactFleetTrips,
        IF(FactFleetTrips[IdleTimeMinutes] > 60, 1, 0)
    ),
    USERELATIONSHIP(DimTime[TimeKey], FactFleetTrips[StartTimeKey]),
    DimTime[DayOfWeek] = SELECTEDVALUE(DimTime[DayOfWeek]),
    DimTime[Hour] = SELECTEDVALUE(DimTime[Hour])
) * 100
```

## Data Loading & ETL Patterns

### Incremental Load Stored Procedure

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    -- Load new operations from staging table (populated by SSIS/external ETL)
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        DwellTimeMinutes, PickRateUnitsPerHour, PackingTimeSeconds, OperatorKey
    )
    SELECT 
        CAST(FORMAT(stg.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        dp.ProductKey,
        dw.WarehouseKey,
        stg.OperationType,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        stg.UnitsProcessed / NULLIF(DATEDIFF(HOUR, stg.StartTime, stg.EndTime), 0) AS PickRate,
        DATEDIFF(SECOND, stg.PackStart, stg.PackEnd) AS PackingTime,
        do.OperatorKey
    FROM StagingWarehouseOps stg
    INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU
    INNER JOIN DimWarehouse dw ON stg.WarehouseID = dw.WarehouseID AND stg.Zone = dw.ZoneID
    LEFT JOIN DimOperator do ON stg.OperatorID = do.OperatorID
    WHERE stg.OperationDateTime > @LastLoadDateTime
        AND stg.IsProcessed = 0;
    
    -- Update staging table
    UPDATE StagingWarehouseOps
    SET IsProcessed = 1, ProcessedDateTime = GETDATE()
    WHERE OperationDateTime > @LastLoadDateTime
        AND IsProcessed = 0;
    
    COMMIT TRANSACTION;
    
    -- Recalculate gravity scores for affected products
    EXEC sp_RecalculateGravityScores;
END;
GO

-- Recalculate product gravity scores based on recent velocity
CREATE PROCEDURE sp_RecalculateGravityScores
AS
BEGIN
    UPDATE DimProductGravity
    SET 
        GravityScore = (
            -- Velocity component (60% weight)
            (ISNULL(recent.OperationsPerDay, 0) / 10.0) * 60 +
            -- Value component (30% weight) - placeholder, integrate with pricing table
            30 +
            -- Fragility penalty (10% weight)
            CASE Fragility 
                WHEN 'HIGH' THEN -10
                WHEN 'MEDIUM' THEN -5
                ELSE 0
            END
        ),
        VelocityClass = CASE 
            WHEN ISNULL(recent.OperationsPerDay, 0) >= 20 THEN 'FAST'
            WHEN ISNULL(recent.OperationsPerDay, 0) >= 5 THEN 'MEDIUM'
            ELSE 'SLOW'
        END,
        LastRecalcDate = GETDATE()
    FROM DimProductGravity dpg
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(dt.Date), MAX(dt.Date)), 0) AS OperationsPerDay
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.Date >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    ) recent ON dpg.ProductKey = recent.ProductKey;
END;
GO
```

### External API Integration (Weather Data)

```sql
-- Example: Create external table for weather data (SQL Server 2019+ PolyBase)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = '${WEATHER_API_ENDPOINT}',
    CREDENTIAL = WeatherAPICredential
);

-- Materialized view refreshed hourly
CREATE VIEW vw_WeatherImpact
AS
SELECT 
    fft.TripKey,
    fft.StartTimeKey,
    dr.RouteID,
    weather.Condition,
    weather.TemperatureFahrenheit,
    weather.PrecipitationInches,
    CASE 
        WHEN weather.Condition IN ('SNOW', 'ICE', 'HEAVY_RAIN') THEN 1
        ELSE 0
    END AS WeatherDelayLikely
FROM FactFleetTrips fft
INNER JOIN DimRoute dr ON fft.RouteKey = dr.RouteKey
CROSS APPLY (
    SELECT TOP 1 * 
    FROM ExternalWeatherData weather
    WHERE weather.LocationLat = dr.StartLat
        AND weather.LocationLon = dr.StartLon
        AND weather.ObservationDateTime <= fft.StartDateTime
    ORDER BY weather.ObservationDateTime DESC
) weather;
```

## Automated Alerting Setup

```sql
-- Alert configuration table
CREATE TABLE AlertConfig (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    AlertName VARCHAR(255),
    AlertType VARCHAR(50), -- 'THRESHOLD', 'ANOMALY', 'PREDICTIVE'
    MetricName VARCHAR(100),
    ThresholdValue DECIMAL(18,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '>=', '<=', '='
    CheckFrequencyMinutes INT,
    RecipientEmails VARCHAR(MAX), -- Comma-separated
    IsActive BIT DEFAULT 1
);

-- Stored procedure to check alerts and send notifications
CREATE PROCEDURE sp_CheckAlerts
AS
BEGIN
    DECLARE @AlertID INT, @MetricName VARCHAR(100), @Threshold DECIMAL(18,2);
    DECLARE @ComparisonOp VARCHAR(10), @Recipients VARCHAR(MAX);
    DECLARE @CurrentValue DECIMAL(18,2), @AlertMessage NVARCHAR(MAX);
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, MetricName, ThresholdValue, ComparisonOperator, RecipientEmails
    FROM AlertConfig
    WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricName, @Threshold, @ComparisonOp, @Recipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Example: Fleet idling time threshold
        IF @MetricName = 'FleetIdleTimePercent'
        BEGIN
            SELECT @CurrentValue = 
                (SUM(IdleTimeMinutes) * 100.0) / NULLIF(SUM(IdleTimeMinutes + DATEDIFF(MINUTE, 0, CAST(DistanceMiles / 45.0 AS TIME))), 0)
            FROM FactFleetTrips
            WHERE StartTimeKey >= CAST(FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm') AS INT);
            
            IF (@ComparisonOp = '>' AND @CurrentValue > @Threshold) OR
               (@ComparisonOp = '<' AND @CurrentValue < @Threshold)
            BEGIN
                SET @AlertMessage = 'ALERT: Fleet idle time is ' + CAST(@CurrentValue AS VARCHAR(10)) + 
                    '% (threshold: ' + CAST(@Threshold AS VARCHAR(10)) + '%)';
                
                -- Send email via Database Mail
                EXEC msdb.dbo.sp_send_dbmail
                    @profile_name = 'LogiFleetAlerts',
                    @recipients = @Recipients,
                    @subject = 'LogiFleet Pulse Alert: Fleet Idle Time Exceeded',
                    @body = @AlertMessage;
            END
        END
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricName, @Threshold, @ComparisonOp, @Recipients;
    END
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
GO

-- Schedule via SQL Server Agent job (runs every 15 minutes)
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Clustered columnstore for large fact tables (best for analytics)
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOps 
ON FactWarehouseOperations;

-- Nonclustered rowstore indexes for selective queries
CREATE NONCLUSTERED INDEX IX_FactFleet_TimeVehicle
ON FactFleetTrips (StartTimeKey, VehicleKey)
INCLUDE (FuelConsumedGallons, IdleTimeMinutes);

-- Partitioning by date for large fact tables
CREATE PARTITION FUNCTION PF_ByMonth (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601,
    20260701, 20260801, 20260901, 20261001, 20261101, 20261201
);

CREATE PARTITION SCHEME PS_ByMonth
AS PARTITION PF_ByMonth
ALL TO ([PRIMARY]);

-- Apply to fact table (requires recreation)
```

### Row-Level Security

```sql
-- Create security policies for regional access
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_WarehouseSecurityPredicate(@GeographyKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessPermitted
    FROM dbo.UserGeographyAccess uga
    WHERE uga.UserName = USER_NAME()
        AND uga.GeographyKey = @GeographyKey
);
GO

CREATE SECURITY POLICY WarehouseAccessPolicy
ADD FILTER PREDICATE Security.fn_WarehouseSecurityPredicate(GeographyKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

## Troubleshooting Common Issues

### Power BI Refresh Failures

```sql
-- Check for orphaned records in fact tables
SELECT 'Orphaned Products' AS Issue, COUNT(*) AS Count
FROM FactWarehouseOperations fwo
WHERE NOT EXISTS (SELECT 1 FROM DimProductGravity dp WHERE dp.ProductKey = fwo.ProductKey)
UNION ALL
SELECT 'Orphaned Warehouses', COUNT(*)
FROM FactWarehouseOperations fwo
WHERE NOT EXISTS (SELECT 1 FROM DimWarehouse dw WHERE dw.WarehouseKey = fwo.WarehouseKey);

-- Fix orphaned records by assigning to 'Unknown' dimension member
UPDATE FactWarehouseOperations
SET ProductKey = -1 -- Unknown product
WHERE NOT EXISTS (SELECT 1 FROM DimProductGravity WHERE ProductKey = FactWarehouseOperations.ProductKey);
```

### Query Performance Optimization

```sql
-- Analyze slow queries via Query Store
SELECT 
    qt.query_sql_text,
    rs.avg_duration / 1000.0 AS avg_duration_ms,
    rs.avg_logical_io_reads,
    rs.count_executions
FROM sys.query_store_query q
INNER JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan p ON q.query_id = p.query_id
INNER JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
WHERE qt.query_sql_text LIKE '%FactWarehouseOperations%'
ORDER BY rs.avg_duration DESC;

-- Update statistics for better query plans
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Data Quality Validation

```sql
-- Detect anomalies in dwell time (statistical outliers)
WITH DwellStats AS (
    SELECT 
        AVG(DwellTimeMinutes) AS MeanDwell,
        STDEV(DwellTimeMinutes) AS StdDevDwell
    FROM FactWarehouseOperations
    WHERE TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
)
SELECT 
    fwo.OperationKey,
    fwo.DwellTimeMinutes,
    ds.MeanDwell,
    ds.StdDevDwell,
    (fwo.DwellTimeMinutes - ds.MeanDwell) / NULLIF(ds.StdDevDwell, 0) AS ZScore
FROM FactWarehouseOperations fwo
CROSS JOIN DwellStats ds
WHERE ABS((fwo.DwellTimeMinutes - ds.MeanDwell) / NULLIF(ds.StdDevDwell, 0)) > 3
ORDER BY ZScore DESC;
```

## Advanced Use Cases

### Temporal Elasticity Simulation

```sql
-- Simulate impact of increasing warehouse capacity from 80% to 95%
WITH CurrentState AS (
    SELECT 
        dw.WarehouseKey,
        dw.Capacity,
        COUNT(DISTINCT fwo.ProductKey) AS CurrentLoad,
        AVG(fwo.DwellTimeMinutes) AS CurrentAvgDwell
    FROM DimWarehouse dw
    INNER JOIN FactWarehouseOperations fwo ON dw.WarehouseKey = fwo.WarehouseKey
    WHERE fwo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY dw.WarehouseKey, dw.Capacity
),
SimulatedState AS (
    SELECT 
        WarehouseKey,
        Capacity,
        CurrentLoad * 1.1875 AS SimulatedLoad, -- Scale from 80% to 95%
        -- Assume dwell time increases exponentially near capacity
        CurrentAvgDwell * POWER(1.1875, 2) AS SimulatedAvgDwell
    FROM CurrentState
)
SELECT 
    cs.WarehouseKey,
    cs.CurrentLoad,
    ss.SimulatedLoad,
    cs.CurrentAvgDwell,
    ss.SimulatedAvgDwell,
    (ss.SimulatedAvgDwell - cs.CurrentAvgDwell) / cs.CurrentAvgDwell * 100 AS DwellTimeIncreasePercent
FROM CurrentState cs
INNER JOIN SimulatedState ss ON cs.WarehouseKey = ss.WarehouseKey
ORDER BY DwellTimeIncreasePercent DESC;
```

This skill provides comprehensive guidance for implementing and using LogiFleet Pulse's multi-modal supply chain analytics platform, from schema deployment to advanced cross-fact queries and Power BI integration.
