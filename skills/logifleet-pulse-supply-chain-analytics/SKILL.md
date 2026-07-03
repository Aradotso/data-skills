---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse operations, fleet tracking, and supply chain optimization
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure warehouse and fleet tracking star schema
  - implement cross-modal supply chain analytics
  - create logistics intelligence dashboard
  - integrate warehouse management with fleet telemetry
  - build multi-fact logistics data model
  - optimize supply chain kpi tracking
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It implements a multi-fact star schema that integrates:

- Warehouse management operations (receiving, putaway, picking, packing, shipping)
- Fleet telemetry and GPS tracking (routes, fuel, driver behavior)
- Supplier and procurement data (lead times, defect rates)
- External data feeds (weather, traffic)
- Customer order history and demand patterns

The platform uses time-phased dimensions, bridge tables for many-to-many relationships, and cross-fact KPI harmonization to provide unified logistics analytics.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to warehouse management system (WMS) and fleet telemetry APIs
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Deploy SQL Schema

Clone the repository and deploy the database schema:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

Execute the SQL schema creation script in SSMS:

```sql
-- Connect to your SQL Server instance
-- Execute the schema creation script (adjust for your environment)

USE master;
GO

CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = LogiFleetPulse_Data,
    FILENAME = 'C:\SQLData\LogiFleetPulse_Data.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = LogiFleetPulse_Log,
    FILENAME = 'C:\SQLData\LogiFleetPulse_Log.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

USE LogiFleetPulse;
GO
```

### Step 2: Create Core Dimension Tables

```sql
-- DimTime: Time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    TimeValue TIME NOT NULL,
    Hour INT NOT NULL,
    Minute INT NOT NULL,
    QuarterHour INT NOT NULL, -- 0, 15, 30, 45
    HourBucket VARCHAR(10) NOT NULL, -- '00:00', '00:15', etc.
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod INT NOT NULL,
    CONSTRAINT CK_Hour CHECK (Hour BETWEEN 0 AND 23),
    CONSTRAINT CK_Minute CHECK (Minute IN (0, 15, 30, 45))
);

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(FullDateTime);
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- 'Warehouse', 'Route Node', 'Cross-Dock'
    ParentLocationID VARCHAR(50) NULL,
    Continent VARCHAR(50) NOT NULL,
    Country VARCHAR(100) NOT NULL,
    Region VARCHAR(100) NOT NULL,
    City VARCHAR(100) NOT NULL,
    PostalCode VARCHAR(20) NULL,
    Latitude DECIMAL(9,6) NULL,
    Longitude DECIMAL(9,6) NULL,
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL
);

CREATE INDEX IX_DimGeography_LocationID ON DimGeography(LocationID);
CREATE INDEX IX_DimGeography_Type ON DimGeography(LocationType);
GO

-- DimProduct: Product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100) NOT NULL,
    SubCategory VARCHAR(100) NULL,
    UnitOfMeasure VARCHAR(20) NOT NULL,
    Weight DECIMAL(10,2) NOT NULL,
    Volume DECIMAL(10,2) NOT NULL,
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    RequiresColdStorage BIT DEFAULT 0,
    StandardCost DECIMAL(12,2) NOT NULL,
    GravityScore DECIMAL(5,2) NULL, -- Calculated: velocity * value * (1 + fragility)
    VelocityTier VARCHAR(20) NULL, -- 'Fast', 'Medium', 'Slow'
    IsActive BIT DEFAULT 1
);

CREATE INDEX IX_DimProduct_SKU ON DimProduct(SKU);
CREATE INDEX IX_DimProduct_GravityScore ON DimProduct(GravityScore DESC);
GO
```

### Step 3: Create Fact Tables

```sql
-- FactWarehouseOperations: Warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    OperationID VARCHAR(50) NOT NULL UNIQUE,
    TimeKey INT NOT NULL,
    DateKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT NOT NULL,
    DwellTimeMinutes INT NULL, -- Time between operations
    OperatorID VARCHAR(50) NOT NULL,
    EquipmentID VARCHAR(50) NULL,
    StorageZone VARCHAR(50) NULL,
    BinLocation VARCHAR(50) NULL,
    BatchNumber VARCHAR(50) NULL,
    CONSTRAINT FK_WHOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WHOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WHOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

CREATE CLUSTERED INDEX IX_FactWHOps_Date_Time ON FactWarehouseOperations(DateKey, TimeKey);
CREATE INDEX IX_FactWHOps_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWHOps_Location ON FactWarehouseOperations(GeographyKey);
GO

-- FactFleetTrips: Fleet telemetry and route tracking
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID VARCHAR(50) NOT NULL UNIQUE,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50) NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    StartDateKey INT NOT NULL,
    EndDateKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TotalDistanceKM DECIMAL(10,2) NOT NULL,
    TotalDurationMinutes INT NOT NULL,
    FuelConsumedLiters DECIMAL(10,2) NOT NULL,
    IdleTimeMinutes INT NOT NULL,
    LoadWeightKG DECIMAL(10,2) NOT NULL,
    StopCount INT NOT NULL,
    AverageSpeed DECIMAL(5,2) NOT NULL,
    MaxSpeed DECIMAL(5,2) NOT NULL,
    WeatherCondition VARCHAR(50) NULL,
    TrafficDelayMinutes INT DEFAULT 0,
    CONSTRAINT FK_FleetTrips_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE CLUSTERED INDEX IX_FactFleetTrips_Date ON FactFleetTrips(StartDateKey, EndDateKey);
CREATE INDEX IX_FactFleetTrips_Vehicle ON FactFleetTrips(VehicleID);
GO

-- Bridge table for many-to-many relationship between trips and products
CREATE TABLE BridgeTripProduct (
    TripKey BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    Quantity INT NOT NULL,
    WeightPercentage DECIMAL(5,2) NOT NULL,
    PRIMARY KEY (TripKey, ProductKey),
    CONSTRAINT FK_Bridge_Trip FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_Bridge_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
GO
```

### Step 4: Configure Data Source Connections

Create a configuration file for your data sources:

```json
{
  "dataSources": {
    "wmsApi": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "fleetTelemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "apiKey": "${FLEET_API_KEY}",
      "refreshIntervalMinutes": 15
    },
    "weatherApi": {
      "endpoint": "${WEATHER_API_ENDPOINT}",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshIntervalMinutes": 60
    }
  },
  "sqlServer": {
    "connectionString": "Server=${SQL_SERVER};Database=LogiFleetPulse;Trusted_Connection=True;",
    "commandTimeout": 300
  }
}
```

### Step 5: Create ETL Stored Procedures

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO FactWarehouseOperations (
        OperationID, TimeKey, DateKey, GeographyKey, ProductKey,
        OperationType, Quantity, DwellTimeMinutes, OperatorID,
        EquipmentID, StorageZone, BinLocation, BatchNumber
    )
    SELECT 
        stg.OperationID,
        dt.TimeKey,
        CONVERT(INT, CONVERT(VARCHAR(8), stg.OperationDateTime, 112)) AS DateKey,
        dg.GeographyKey,
        dp.ProductKey,
        stg.OperationType,
        stg.Quantity,
        stg.DwellTimeMinutes,
        stg.OperatorID,
        stg.EquipmentID,
        stg.StorageZone,
        stg.BinLocation,
        stg.BatchNumber
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime dt ON 
        DATEPART(HOUR, stg.OperationDateTime) = dt.Hour
        AND DATEPART(MINUTE, stg.OperationDateTime) / 15 * 15 = dt.Minute
    INNER JOIN DimGeography dg ON stg.LocationID = dg.LocationID
    INNER JOIN DimProduct dp ON stg.SKU = dp.SKU
    WHERE stg.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations 
            WHERE OperationID = stg.OperationID
        );
    
    -- Update gravity scores based on recent activity
    UPDATE dp
    SET GravityScore = velocity.MovementScore * dp.StandardCost * (1 + CAST(dp.IsFragile AS DECIMAL(5,2)))
    FROM DimProduct dp
    INNER JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(DateKey), MAX(DateKey)), 0) AS MovementScore
        FROM FactWarehouseOperations
        WHERE DateKey >= CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -90, GETDATE()), 112))
        GROUP BY ProductKey
    ) velocity ON dp.ProductKey = velocity.ProductKey;
    
END;
GO

-- Stored procedure for fleet trip analysis
CREATE PROCEDURE sp_AnalyzeFleetEfficiency
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate fleet efficiency metrics
    SELECT 
        v.VehicleID,
        v.VehicleType,
        COUNT(f.TripKey) AS TotalTrips,
        SUM(f.TotalDistanceKM) AS TotalDistance,
        SUM(f.FuelConsumedLiters) AS TotalFuel,
        AVG(f.FuelConsumedLiters / NULLIF(f.TotalDistanceKM, 0)) AS AvgFuelPerKM,
        SUM(f.IdleTimeMinutes) AS TotalIdleTime,
        SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(f.TotalDurationMinutes), 0) AS IdleTimePercentage,
        AVG(f.LoadWeightKG * 100.0 / v.MaxLoadCapacityKG) AS AvgCapacityUtilization,
        SUM(f.TrafficDelayMinutes) AS TotalTrafficDelay
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleID = v.VehicleID
    WHERE f.StartDateKey BETWEEN CONVERT(INT, CONVERT(VARCHAR(8), @StartDate, 112))
                            AND CONVERT(INT, CONVERT(VARCHAR(8), @EndDate, 112))
    GROUP BY v.VehicleID, v.VehicleType, v.MaxLoadCapacityKG
    ORDER BY AvgFuelPerKM DESC;
    
END;
GO
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: Your SQL Server instance name
4. Database: LogiFleetPulse
5. Data Connectivity mode: Import (for best performance) or DirectQuery (for real-time)

### Create Relationships in Power BI

The relationships should be automatically detected, but verify:

```
DimTime (TimeKey) → FactWarehouseOperations (TimeKey) [Many-to-One]
DimTime (TimeKey) → FactFleetTrips (StartTimeKey) [Many-to-One]
DimGeography (GeographyKey) → FactWarehouseOperations (GeographyKey) [Many-to-One]
DimGeography (GeographyKey) → FactFleetTrips (OriginGeographyKey) [Many-to-One]
DimProduct (ProductKey) → FactWarehouseOperations (ProductKey) [Many-to-One]
DimProduct (ProductKey) → BridgeTripProduct (ProductKey) [Many-to-One]
FactFleetTrips (TripKey) → BridgeTripProduct (TripKey) [One-to-Many]
```

### Create Calculated Measures in Power BI

```dax
// Cross-fact KPI: Warehouse dwell time vs Fleet idle time correlation
Dwell_Idle_Correlation = 
VAR WarehouseDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[OperationType] = "Putaway"
    )
VAR FleetIdle = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes])
    )
RETURN
    IF(
        NOT(ISBLANK(WarehouseDwell)) && NOT(ISBLANK(FleetIdle)),
        (WarehouseDwell / 60) * FleetIdle,
        BLANK()
    )

// Warehouse gravity zone efficiency
Gravity_Zone_Efficiency = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[Quantity]),
        FactWarehouseOperations[OperationType] = "Picking"
    ),
    CALCULATE(
        DISTINCTCOUNT(FactWarehouseOperations[OperationID]),
        FactWarehouseOperations[OperationType] = "Picking"
    )
)

// Fleet fuel efficiency with load factor
Fuel_Efficiency_Adjusted = 
DIVIDE(
    SUM(FactFleetTrips[TotalDistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters])
) * 
DIVIDE(
    AVERAGE(FactFleetTrips[LoadWeightKG]),
    1000
)

// Predictive bottleneck index (based on historical patterns)
Bottleneck_Risk_Score = 
VAR CurrentHour = HOUR(NOW())
VAR CurrentDayOfWeek = WEEKDAY(TODAY())
VAR HistoricalAvgDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DimTime[Hour] = CurrentHour,
        DimTime[DayOfWeek] = CurrentDayOfWeek
    )
VAR CurrentDwell = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FactWarehouseOperations[DateKey] = CONVERT(INT, FORMAT(TODAY(), "yyyyMMdd"))
    )
RETURN
    IF(
        CurrentDwell > HistoricalAvgDwell * 1.2,
        (CurrentDwell / HistoricalAvgDwell) * 100,
        0
    )
```

## Common Usage Patterns

### Pattern 1: Cross-Modal Supply Chain Query

Find all shipments delayed due to weather that originated from cold-storage zones with high dwell time:

```sql
SELECT 
    f.TripID,
    f.VehicleID,
    dg_origin.LocationName AS OriginWarehouse,
    dg_dest.LocationName AS Destination,
    wo.StorageZone,
    wo.DwellTimeMinutes,
    f.TrafficDelayMinutes,
    f.WeatherCondition,
    p.ProductName,
    p.IsPerishable
FROM FactFleetTrips f
INNER JOIN FactWarehouseOperations wo ON 
    wo.GeographyKey = f.OriginGeographyKey
    AND wo.DateKey = f.StartDateKey
INNER JOIN BridgeTripProduct btp ON f.TripKey = btp.TripKey
INNER JOIN DimProduct p ON btp.ProductKey = p.ProductKey
INNER JOIN DimGeography dg_origin ON f.OriginGeographyKey = dg_origin.GeographyKey
INNER JOIN DimGeography dg_dest ON f.DestinationGeographyKey = dg_dest.GeographyKey
WHERE 
    f.WeatherCondition IN ('Rain', 'Snow', 'Fog')
    AND f.TrafficDelayMinutes > 30
    AND wo.StorageZone LIKE '%Cold%'
    AND wo.DwellTimeMinutes > 4320 -- 72 hours
    AND p.RequiresColdStorage = 1
    AND f.StartDateKey >= CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(MONTH, -3, GETDATE()), 112))
ORDER BY f.TrafficDelayMinutes DESC, wo.DwellTimeMinutes DESC;
```

### Pattern 2: Warehouse Gravity Zone Optimization

Identify products that should be moved to higher-gravity zones:

```sql
WITH ProductVelocity AS (
    SELECT 
        ProductKey,
        COUNT(*) AS PickCount,
        AVG(DwellTimeMinutes) AS AvgDwell,
        STRING_AGG(StorageZone, ', ') WITHIN GROUP (ORDER BY StorageZone) AS CurrentZones
    FROM FactWarehouseOperations
    WHERE 
        OperationType = 'Picking'
        AND DateKey >= CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -30, GETDATE()), 112))
    GROUP BY ProductKey
)
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore AS CurrentGravityScore,
    pv.PickCount,
    pv.AvgDwell,
    pv.CurrentZones,
    CASE 
        WHEN pv.PickCount > 100 AND p.GravityScore < 50 THEN 'Move to High-Gravity Zone'
        WHEN pv.PickCount BETWEEN 50 AND 100 AND p.GravityScore < 30 THEN 'Move to Medium-Gravity Zone'
        ELSE 'Current Zone Optimal'
    END AS Recommendation
FROM DimProduct p
INNER JOIN ProductVelocity pv ON p.ProductKey = pv.ProductKey
WHERE p.IsActive = 1
ORDER BY pv.PickCount DESC;
```

### Pattern 3: Fleet Maintenance Triage

Generate proactive maintenance queue ranked by revenue impact:

```sql
WITH VehicleMetrics AS (
    SELECT 
        f.VehicleID,
        AVG(f.IdleTimeMinutes * 100.0 / f.TotalDurationMinutes) AS AvgIdlePercent,
        AVG(f.FuelConsumedLiters / NULLIF(f.TotalDistanceKM, 0)) AS AvgFuelConsumption,
        SUM(btp.Quantity * p.StandardCost) AS TotalCargoValue
    FROM FactFleetTrips f
    INNER JOIN BridgeTripProduct btp ON f.TripKey = btp.TripKey
    INNER JOIN DimProduct p ON btp.ProductKey = p.ProductKey
    WHERE f.StartDateKey >= CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -7, GETDATE()), 112))
    GROUP BY f.VehicleID
)
SELECT 
    vm.VehicleID,
    vm.AvgIdlePercent,
    vm.AvgFuelConsumption,
    vm.TotalCargoValue,
    CASE 
        WHEN vm.AvgIdlePercent > 20 THEN 'Engine Diagnostics Required'
        WHEN vm.AvgFuelConsumption > 0.15 THEN 'Fuel System Check'
        ELSE 'Normal Operations'
    END AS MaintenanceFlag,
    (vm.AvgIdlePercent * vm.TotalCargoValue / 100) AS RevenueImpactScore
FROM VehicleMetrics vm
WHERE vm.AvgIdlePercent > 15 OR vm.AvgFuelConsumption > 0.12
ORDER BY RevenueImpactScore DESC;
```

### Pattern 4: Temporal Elasticity Simulation

Simulate impact of warehouse capacity changes on fleet utilization:

```sql
DECLARE @CapacityIncreasePct DECIMAL(5,2) = 15.0; -- 15% capacity increase

WITH BaselineMetrics AS (
    SELECT 
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        COUNT(DISTINCT wo.ProductKey) AS UniqueProducts,
        AVG(f.TotalDurationMinutes) AS AvgFleetDuration
    FROM FactWarehouseOperations wo
    INNER JOIN FactFleetTrips f ON 
        wo.GeographyKey = f.OriginGeographyKey
        AND wo.DateKey = f.StartDateKey
    WHERE wo.DateKey >= CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -30, GETDATE()), 112))
),
SimulatedMetrics AS (
    SELECT 
        AVG(wo.DwellTimeMinutes) * (1 - (@CapacityIncreasePct / 100)) AS ProjectedDwell,
        COUNT(DISTINCT wo.ProductKey) * (1 + (@CapacityIncreasePct / 200)) AS ProjectedProducts,
        AVG(f.TotalDurationMinutes) * (1 - (@CapacityIncreasePct / 150)) AS ProjectedFleetDuration
    FROM FactWarehouseOperations wo
    INNER JOIN FactFleetTrips f ON 
        wo.GeographyKey = f.OriginGeographyKey
        AND wo.DateKey = f.StartDateKey
    WHERE wo.DateKey >= CONVERT(INT, CONVERT(VARCHAR(8), DATEADD(DAY, -30, GETDATE()), 112))
)
SELECT 
    'Baseline' AS Scenario,
    bm.AvgDwell,
    bm.UniqueProducts,
    bm.AvgFleetDuration
FROM BaselineMetrics bm
UNION ALL
SELECT 
    'Simulated (+' + CAST(@CapacityIncreasePct AS VARCHAR(10)) + '% capacity)' AS Scenario,
    sm.ProjectedDwell,
    sm.ProjectedProducts,
    sm.ProjectedFleetDuration
FROM SimulatedMetrics sm;
```

## Configuration & Customization

### Adjust Time Granularity

If 15-minute granularity is too fine, modify the time dimension:

```sql
-- Regenerate DimTime for 1-hour granularity
TRUNCATE TABLE DimTime;

INSERT INTO DimTime (TimeKey, FullDateTime, TimeValue, Hour, Minute, QuarterHour, HourBucket, DayOfWeek, DayName, IsWeekend, FiscalPeriod)
SELECT 
    CAST(CONVERT(VARCHAR(8), d.Date, 112) + RIGHT('00' + CAST(h.Hour AS VARCHAR(2)), 2) + '00' AS INT) AS TimeKey,
    DATEADD(HOUR, h.Hour, d.Date) AS FullDateTime,
    CAST(CAST(h.Hour AS VARCHAR(2)) + ':00:00' AS TIME) AS TimeValue,
    h.Hour,
    0 AS Minute,
    0 AS QuarterHour,
    RIGHT('00' + CAST(h.Hour AS VARCHAR(2)), 2) + ':00' AS HourBucket,
    DATEPART(WEEKDAY, d.Date) AS DayOfWeek,
    DATENAME(WEEKDAY, d.Date) AS DayName,
    CASE WHEN DATEPART(WEEKDAY, d.Date) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend,
    DATEPART(QUARTER, d.Date) AS FiscalPeriod
FROM 
    (SELECT DATEADD(DAY, number, '2020-01-01') AS Date
     FROM master..spt_values
     WHERE type = 'P' AND number < 3650) d
CROSS JOIN
    (SELECT number AS Hour FROM master..spt_values WHERE type = 'P' AND number BETWEEN 0 AND 23) h;
```

### Configure Alert Thresholds

Create alert configuration table:

```sql
CREATE TABLE AlertConfiguration (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName VARCHAR(100) NOT NULL,
    MetricType VARCHAR(50) NOT NULL, -- 'DwellTime', 'IdleTime', 'FuelConsumption', etc.
    Threshold DECIMAL(10,2) NOT NULL,
    ComparisonOperator VARCHAR(10) NOT NULL, -- '>', '<', '>=', '<=', '='
    NotificationEmail VARCHAR(255) NOT NULL,
    IsActive BIT DEFAULT 1
);

-- Example alert configurations
INSERT INTO AlertConfiguration (AlertName, MetricType, Threshold, ComparisonOperator, NotificationEmail)
VALUES 
    ('High Dwell Time Alert', 'DwellTime', 4320, '>', '${ALERT_EMAIL}'),
    ('Excessive Fleet Idle', 'IdleTimePercentage', 20, '>', '${ALERT_EMAIL}'),
    ('Low Capacity Utilization', 'CapacityUtilization', 60, '<', '${ALERT_EMAIL}');

-- Scheduled job to check alerts (run every 15 minutes)
CREATE PROCEDURE sp_CheckAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check warehouse dwell time alerts
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations wo
        INNER JOIN AlertConfiguration ac ON ac.MetricType = 'DwellTime' AND ac.IsActive = 1
        WHERE wo.DwellTimeMinutes > ac.Threshold
            AND wo.DateKey = CONVERT(INT, CONVERT(VARCHAR(8), GETDATE(), 112))
    )
    BEGIN
        SET @AlertMessage = 'ALERT: High dwell time detected in warehouse operations.';
        -- Send email notification (use sp_send_dbmail or external integration)
    END;
    
    -- Additional alert checks...
END;
GO
```

## Troubleshooting

### Issue: Power BI refresh takes too long

**Solution**: Implement incremental refresh in Power BI:

1. In Power BI Desktop, create parameters: `RangeStart` and `RangeEnd`
2. Filter fact tables using these parameters:
   ```
   = Table.SelectRows(FactWarehouseOperations, each [DateKey] >= RangeStart and [DateKey] < RangeEnd)
   ```
3. Configure incremental refresh policy in Power BI Service

### Issue: Cross-fact queries are slow

**Solution**: Create indexed views for common join patterns:

```sql
CREATE VIEW vw_WarehouseFleetJoin
WITH SCHEMABINDING
AS
SELECT 
    wo.ProductKey,
    wo.GeographyKey,
    wo.DateKey,
    COUNT_BIG(*) AS OperationCount,
    AVG(wo.DwellTimeMinutes) AS AvgDwell,
    SUM(wo.Quantity) AS TotalQuantity
FROM dbo.FactWarehouseOperations wo
GROUP BY wo.ProductKey, wo.GeographyKey, wo.DateKey;
GO

CREATE UNIQUE CLUSTERED INDEX IX_WHFleet ON vw_WarehouseFleetJoin (ProductKey, GeographyKey, DateKey);
GO
```

### Issue: Gravity scores not updating

**Solution**: Schedule the gravity score calculation stored procedure:

```sql
-- Create SQL Server Agent job to run sp_LoadWarehouseOperations
