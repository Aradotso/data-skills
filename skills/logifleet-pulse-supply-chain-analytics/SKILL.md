---
name: logifleet-pulse-supply-chain-analytics
description: Power BI and MS SQL Server logistics intelligence platform for warehouse operations, fleet management, and cross-modal supply chain analytics
triggers:
  - set up logifleet pulse warehouse analytics
  - deploy logistics data warehouse with power bi
  - configure supply chain intelligence dashboard
  - implement fleet telemetry and warehouse integration
  - create multi-fact star schema for logistics
  - build real-time logistics kpi dashboard
  - integrate warehouse and fleet data models
  - setup cross-modal supply chain analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a comprehensive MS SQL Server data warehouse and Power BI visualization platform for logistics intelligence. It unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a multi-fact star schema with time-phased dimensions. The platform enables real-time cross-modal analytics linking warehouse velocity, fleet performance, and supply chain bottlenecks through a semantic data layer.

## Core Architecture

LogiFleet Pulse uses a multi-fact star schema with:
- **FactWarehouseOperations**: Putaway, picking, packing, dwell time metrics
- **FactFleetTrips**: Route segments, fuel consumption, idle time, loading events
- **FactCrossDock**: Cross-dock transfer operations
- **DimTime**: 15-minute granularity time dimension with fiscal periods
- **DimGeography**: Hierarchical location data (continent → country → region → node)
- **DimProductGravity**: Velocity-based product classification system
- **DimSupplierReliability**: Supplier performance metrics

## Installation

### Prerequisites

```bash
# Required software
- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- Python 3.8+ (for data ingestion scripts)
```

### Database Setup

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Execute schema creation script
-- Connect to your SQL Server instance first
:setvar DatabaseName "LogiFleetPulse"
:setvar DataPath "C:\SQLData\"
:setvar LogPath "C:\SQLLogs\"

USE master;
GO

-- Create database
CREATE DATABASE $(DatabaseName)
ON PRIMARY 
(
    NAME = LogiFleetPulse_Data,
    FILENAME = '$(DataPath)LogiFleetPulse.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = LogiFleetPulse_Log,
    FILENAME = '$(LogPath)LogiFleetPulse.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

USE $(DatabaseName);
GO
```

3. **Create dimension tables:**

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE dbo.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME(0) NOT NULL,
    Hour15MinBucket TINYINT NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    FiscalQuarter TINYINT NOT NULL,
    FiscalPeriod TINYINT NOT NULL,
    INDEX IX_DimTime_DateKey NONCLUSTERED (DateKey),
    INDEX IX_DimTime_FullDateTime NONCLUSTERED (FullDateTime)
);

-- DimGeography: Hierarchical location data
CREATE TABLE dbo.DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- Warehouse, Route Node, Distribution Center
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    PostalCode VARCHAR(20),
    Country VARCHAR(100) NOT NULL,
    Continent VARCHAR(50) NOT NULL,
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    EndDate DATE,
    INDEX IX_DimGeography_LocationID NONCLUSTERED (LocationID),
    INDEX IX_DimGeography_Country NONCLUSTERED (Country)
);

-- DimProductGravity: Product classification with velocity scoring
CREATE TABLE dbo.DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Composite score: velocity + value + fragility
    VelocityClass VARCHAR(20), -- Fast-Mover, Medium, Slow-Mover
    ValueClass VARCHAR(20), -- High-Value, Medium-Value, Low-Value
    FragilityScore TINYINT, -- 1-10 scale
    OptimalStorageZone VARCHAR(50),
    UnitWeight DECIMAL(10,4),
    UnitVolume DECIMAL(10,4),
    RequiresRefrigeration BIT DEFAULT 0,
    ShelfLifeDays INT,
    EffectiveDate DATE NOT NULL,
    EndDate DATE,
    INDEX IX_DimProductGravity_SKU NONCLUSTERED (SKU),
    INDEX IX_DimProductGravity_GravityScore NONCLUSTERED (GravityScore)
);

-- DimSupplierReliability: Supplier performance tracking
CREATE TABLE dbo.DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeAvgDays DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,4),
    OnTimeDeliveryRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- Platinum, Gold, Silver, Bronze
    Country VARCHAR(100),
    EffectiveDate DATE NOT NULL,
    EndDate DATE,
    INDEX IX_DimSupplierReliability_SupplierID NONCLUSTERED (SupplierID)
);
```

4. **Create fact tables:**

```sql
-- FactWarehouseOperations: Core warehouse activity metrics
CREATE TABLE dbo.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2(0) NOT NULL,
    OperationEndTime DATETIME2(0) NOT NULL,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    DwellTimeHours DECIMAL(10,2),
    QuantityProcessed INT NOT NULL,
    UnitsPerHour AS CASE WHEN DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) > 0 
                        THEN QuantityProcessed * 60.0 / DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) 
                        ELSE 0 END PERSISTED,
    EmployeeCount TINYINT,
    EquipmentUsed VARCHAR(100),
    StorageZone VARCHAR(50),
    AisleLocation VARCHAR(50),
    ErrorCount INT DEFAULT 0,
    BatchID VARCHAR(100),
    INDEX IX_FactWarehouseOps_TimeKey NONCLUSTERED (TimeKey),
    INDEX IX_FactWarehouseOps_ProductKey NONCLUSTERED (ProductKey),
    INDEX IX_FactWarehouseOps_OperationType NONCLUSTERED (OperationType),
    CONSTRAINT FK_FactWarehouseOps_DimTime FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouseOps_DimGeography FOREIGN KEY (GeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_FactWarehouseOps_DimProductGravity FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey)
);

-- FactFleetTrips: Fleet telemetry and route performance
CREATE TABLE dbo.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripStartTime DATETIME2(0) NOT NULL,
    TripEndTime DATETIME2(0) NOT NULL,
    TripDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime) PERSISTED,
    DistanceKM DECIMAL(10,2) NOT NULL,
    FuelConsumedLiters DECIMAL(10,4),
    FuelEfficiencyKmPerLiter AS CASE WHEN FuelConsumedLiters > 0 THEN DistanceKM / FuelConsumedLiters ELSE 0 END PERSISTED,
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    LoadWeight DECIMAL(10,2), -- KG
    MaxSpeed DECIMAL(5,2), -- KM/H
    AvgSpeed DECIMAL(5,2), -- KM/H
    HarshBrakingCount INT DEFAULT 0,
    HarshAccelerationCount INT DEFAULT 0,
    DelayMinutes DECIMAL(10,2) DEFAULT 0,
    DelayReason VARCHAR(200),
    WeatherCondition VARCHAR(50),
    TrafficCondition VARCHAR(50),
    RouteCompliance DECIMAL(5,4), -- 0-1 scale
    INDEX IX_FactFleetTrips_TimeKey NONCLUSTERED (TimeKey),
    INDEX IX_FactFleetTrips_VehicleID NONCLUSTERED (VehicleID),
    INDEX IX_FactFleetTrips_OriginGeographyKey NONCLUSTERED (OriginGeographyKey),
    CONSTRAINT FK_FactFleetTrips_DimTime FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_FactFleetTrips_OriginGeography FOREIGN KEY (OriginGeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleetTrips_DestinationGeography FOREIGN KEY (DestinationGeographyKey) REFERENCES dbo.DimGeography(GeographyKey)
);

-- FactCrossDock: Cross-dock transfer operations
CREATE TABLE dbo.FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ReceiveTime DATETIME2(0) NOT NULL,
    ShipTime DATETIME2(0) NOT NULL,
    CrossDockDurationMinutes AS DATEDIFF(MINUTE, ReceiveTime, ShipTime) PERSISTED,
    QuantityTransferred INT NOT NULL,
    TemperatureCompliance BIT DEFAULT 1,
    QualityCheckPassed BIT DEFAULT 1,
    INDEX IX_FactCrossDock_TimeKey NONCLUSTERED (TimeKey),
    INDEX IX_FactCrossDock_ProductKey NONCLUSTERED (ProductKey),
    CONSTRAINT FK_FactCrossDock_DimTime FOREIGN KEY (TimeKey) REFERENCES dbo.DimTime(TimeKey),
    CONSTRAINT FK_FactCrossDock_DimGeography FOREIGN KEY (GeographyKey) REFERENCES dbo.DimGeography(GeographyKey),
    CONSTRAINT FK_FactCrossDock_DimProductGravity FOREIGN KEY (ProductKey) REFERENCES dbo.DimProductGravity(ProductKey),
    CONSTRAINT FK_FactCrossDock_DimSupplierReliability FOREIGN KEY (SupplierKey) REFERENCES dbo.DimSupplierReliability(SupplierKey)
);
```

5. **Create ETL stored procedures:**

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Insert new warehouse operations from staging
        INSERT INTO dbo.FactWarehouseOperations (
            TimeKey,
            GeographyKey,
            ProductKey,
            OperationType,
            OperationStartTime,
            OperationEndTime,
            DwellTimeHours,
            QuantityProcessed,
            EmployeeCount,
            EquipmentUsed,
            StorageZone,
            AisleLocation,
            ErrorCount,
            BatchID
        )
        SELECT 
            dt.TimeKey,
            dg.GeographyKey,
            dp.ProductKey,
            stg.OperationType,
            stg.OperationStartTime,
            stg.OperationEndTime,
            stg.DwellTimeHours,
            stg.QuantityProcessed,
            stg.EmployeeCount,
            stg.EquipmentUsed,
            stg.StorageZone,
            stg.AisleLocation,
            stg.ErrorCount,
            stg.BatchID
        FROM staging.WarehouseOperations stg
        INNER JOIN dbo.DimTime dt ON 
            dt.FullDateTime = DATEADD(MINUTE, 
                (DATEPART(MINUTE, stg.OperationStartTime) / 15) * 15, 
                DATEADD(HOUR, DATEPART(HOUR, stg.OperationStartTime), 
                    CAST(CAST(stg.OperationStartTime AS DATE) AS DATETIME2)))
        INNER JOIN dbo.DimGeography dg ON stg.LocationID = dg.LocationID 
            AND stg.OperationStartTime BETWEEN dg.EffectiveDate AND ISNULL(dg.EndDate, '9999-12-31')
        INNER JOIN dbo.DimProductGravity dp ON stg.SKU = dp.SKU
            AND stg.OperationStartTime BETWEEN dp.EffectiveDate AND ISNULL(dp.EndDate, '9999-12-31')
        WHERE CAST(stg.OperationStartTime AS DATE) = @LoadDate
        AND NOT EXISTS (
            SELECT 1 FROM dbo.FactWarehouseOperations f
            WHERE f.BatchID = stg.BatchID
        );
        
        COMMIT TRANSACTION;
        
        PRINT 'Successfully loaded warehouse operations for ' + CAST(@LoadDate AS VARCHAR(10));
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
    END CATCH
END;
GO

-- Stored procedure for fleet trips load
CREATE PROCEDURE dbo.usp_LoadFleetTrips
    @LoadDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        INSERT INTO dbo.FactFleetTrips (
            TimeKey,
            OriginGeographyKey,
            DestinationGeographyKey,
            VehicleID,
            DriverID,
            TripStartTime,
            TripEndTime,
            DistanceKM,
            FuelConsumedLiters,
            IdleTimeMinutes,
            LoadingTimeMinutes,
            UnloadingTimeMinutes,
            LoadWeight,
            MaxSpeed,
            AvgSpeed,
            HarshBrakingCount,
            HarshAccelerationCount,
            DelayMinutes,
            DelayReason,
            WeatherCondition,
            TrafficCondition,
            RouteCompliance
        )
        SELECT 
            dt.TimeKey,
            dg_origin.GeographyKey,
            dg_dest.GeographyKey,
            stg.VehicleID,
            stg.DriverID,
            stg.TripStartTime,
            stg.TripEndTime,
            stg.DistanceKM,
            stg.FuelConsumedLiters,
            stg.IdleTimeMinutes,
            stg.LoadingTimeMinutes,
            stg.UnloadingTimeMinutes,
            stg.LoadWeight,
            stg.MaxSpeed,
            stg.AvgSpeed,
            stg.HarshBrakingCount,
            stg.HarshAccelerationCount,
            stg.DelayMinutes,
            stg.DelayReason,
            stg.WeatherCondition,
            stg.TrafficCondition,
            stg.RouteCompliance
        FROM staging.FleetTrips stg
        INNER JOIN dbo.DimTime dt ON 
            dt.FullDateTime = DATEADD(MINUTE, 
                (DATEPART(MINUTE, stg.TripStartTime) / 15) * 15, 
                DATEADD(HOUR, DATEPART(HOUR, stg.TripStartTime), 
                    CAST(CAST(stg.TripStartTime AS DATE) AS DATETIME2)))
        INNER JOIN dbo.DimGeography dg_origin ON stg.OriginLocationID = dg_origin.LocationID
        INNER JOIN dbo.DimGeography dg_dest ON stg.DestinationLocationID = dg_dest.LocationID
        WHERE CAST(stg.TripStartTime AS DATE) = @LoadDate
        AND NOT EXISTS (
            SELECT 1 FROM dbo.FactFleetTrips f
            WHERE f.VehicleID = stg.VehicleID
            AND f.TripStartTime = stg.TripStartTime
        );
        
        COMMIT TRANSACTION;
        
        PRINT 'Successfully loaded fleet trips for ' + CAST(@LoadDate AS VARCHAR(10));
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
    END CATCH
END;
GO
```

## Configuration

### Connection Configuration

Create `config.json` in the project root:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "integrated",
    "connection_timeout": 30,
    "query_timeout": 600
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "refresh_interval_minutes": 15
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}",
      "refresh_interval_minutes": 5
    },
    "weather_api": {
      "endpoint": "${WEATHER_API_ENDPOINT}",
      "api_key": "${WEATHER_API_KEY}",
      "refresh_interval_minutes": 60
    }
  },
  "powerbi": {
    "workspace_id": "${POWERBI_WORKSPACE_ID}",
    "dataset_refresh_schedule": "0 */1 * * *"
  },
  "alerts": {
    "smtp_server": "${SMTP_SERVER}",
    "smtp_port": 587,
    "from_email": "${ALERT_FROM_EMAIL}",
    "use_tls": true
  }
}
```

### Role-Based Security

```sql
-- Create row-level security for regional access
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_SecurityPredicate(@Country VARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS fn_SecurityPredicate_result
WHERE 
    @Country IN (SELECT Country FROM dbo.UserRegionAccess WHERE UserName = USER_NAME())
    OR IS_MEMBER('db_owner') = 1;
GO

-- Apply security policy to geography dimension
CREATE SECURITY POLICY Security.GeographySecurityPolicy
ADD FILTER PREDICATE Security.fn_SecurityPredicate(Country)
ON dbo.DimGeography
WITH (STATE = ON);
GO
```

## Key Analytics Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Calculate correlation between warehouse dwell time and fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        dt.DateKey,
        dg.Country,
        dp.Category,
        AVG(fwo.DwellTimeHours) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM dbo.FactWarehouseOperations fwo
    INNER JOIN dbo.DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN dbo.DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
    INNER JOIN dbo.DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dt.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY dt.DateKey, dg.Country, dp.Category
),
FleetIdle AS (
    SELECT 
        dt.DateKey,
        dg.Country,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(fft.IdleTimeMinutes * 0.5) AS EstimatedIdleCost, -- $0.50 per minute
        COUNT(*) AS TripCount
    FROM dbo.FactFleetTrips fft
    INNER JOIN dbo.DimTime dt ON fft.TimeKey = dt.TimeKey
    INNER JOIN dbo.DimGeography dg ON fft.OriginGeographyKey = dg.GeographyKey
    WHERE dt.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY dt.DateKey, dg.Country
)
SELECT 
    wd.DateKey,
    wd.Country,
    wd.Category,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.EstimatedIdleCost,
    CASE 
        WHEN wd.AvgDwellTime > 48 AND fi.AvgIdleTime > 30 THEN 'Critical'
        WHEN wd.AvgDwellTime > 24 AND fi.AvgIdleTime > 20 THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckStatus
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.DateKey = fi.DateKey AND wd.Country = fi.Country
ORDER BY fi.EstimatedIdleCost DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be reclassified to different gravity zones
WITH ProductVelocity AS (
    SELECT 
        dp.ProductKey,
        dp.SKU,
        dp.ProductName,
        dp.GravityScore,
        dp.OptimalStorageZone,
        fwo.StorageZone AS CurrentStorageZone,
        COUNT(*) AS PickFrequency,
        AVG(fwo.DwellTimeHours) AS AvgDwellTime,
        SUM(fwo.QuantityProcessed) AS TotalQuantity
    FROM dbo.FactWarehouseOperations fwo
    INNER JOIN dbo.DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN dbo.DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE fwo.OperationType = 'Picking'
    AND dt.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY dp.ProductKey, dp.SKU, dp.ProductName, dp.GravityScore, 
             dp.OptimalStorageZone, fwo.StorageZone
)
SELECT 
    ProductKey,
    SKU,
    ProductName,
    GravityScore,
    OptimalStorageZone,
    CurrentStorageZone,
    PickFrequency,
    AvgDwellTime,
    TotalQuantity,
    CASE 
        WHEN PickFrequency > 100 AND CurrentStorageZone LIKE '%C%' THEN 'Move to Zone A (Fast-Mover)'
        WHEN PickFrequency < 10 AND CurrentStorageZone LIKE '%A%' THEN 'Move to Zone C (Slow-Mover)'
        WHEN AvgDwellTime > 72 AND OptimalStorageZone LIKE '%Temp%' THEN 'Investigate Cold Chain'
        ELSE 'Optimal Placement'
    END AS RecommendedAction,
    CASE 
        WHEN PickFrequency > 100 THEN PickFrequency * 2.5 -- Time saved per repositioning
        WHEN PickFrequency < 10 THEN PickFrequency * 0.8
        ELSE 0
    END AS EstimatedTimeSavingsMinutes
FROM ProductVelocity
WHERE CurrentStorageZone <> OptimalStorageZone
ORDER BY EstimatedTimeSavingsMinutes DESC;
```

### Predictive Fleet Maintenance Triage

```sql
-- Generate maintenance priority queue based on revenue impact
WITH VehiclePerformance AS (
    SELECT 
        fft.VehicleID,
        COUNT(*) AS TripCount,
        AVG(fft.FuelEfficiencyKmPerLiter) AS AvgFuelEfficiency,
        SUM(fft.HarshBrakingCount + fft.HarshAccelerationCount) AS TotalHarshEvents,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(fft.LoadWeight) AS TotalLoadWeight,
        MAX(fft.TripEndTime) AS LastTripTime
    FROM dbo.FactFleetTrips fft
    INNER JOIN dbo.DimTime dt ON fft.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY fft.VehicleID
),
VehicleRevenue AS (
    SELECT 
        fft.VehicleID,
        SUM(fwo.QuantityProcessed * dp.GravityScore * 10) AS EstimatedRevenueImpact -- Proxy metric
    FROM dbo.FactFleetTrips fft
    INNER JOIN dbo.FactWarehouseOperations fwo ON fft.OriginGeographyKey = fwo.GeographyKey
    INNER JOIN dbo.DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN dbo.DimTime dt ON fft.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY fft.VehicleID
)
SELECT 
    vp.VehicleID,
    vp.TripCount,
    vp.AvgFuelEfficiency,
    vp.TotalHarshEvents,
    vp.AvgIdleTime,
    ISNULL(vr.EstimatedRevenueImpact, 0) AS RevenueImpact,
    DATEDIFF(DAY, vp.LastTripTime, GETDATE()) AS DaysSinceLastTrip,
    CASE 
        WHEN vp.AvgFuelEfficiency < 5.0 THEN 'Engine Diagnostics Required'
        WHEN vp.TotalHarshEvents > 50 THEN 'Brake/Suspension Check'
        WHEN vp.AvgIdleTime > 60 THEN 'Idle Management Training'
        ELSE 'No Immediate Action'
    END AS MaintenanceRecommendation,
    (
        (CASE WHEN vp.AvgFuelEfficiency < 5.0 THEN 50 ELSE 0 END) +
        (CASE WHEN vp.TotalHarshEvents > 50 THEN 30 ELSE 0 END) +
        (CASE WHEN vp.AvgIdleTime > 60 THEN 20 ELSE 0 END)
    ) * (ISNULL(vr.EstimatedRevenueImpact, 0) / 1000.0) AS PriorityScore
FROM VehiclePerformance vp
LEFT JOIN VehicleRevenue vr ON vp.VehicleID = vr.VehicleID
WHERE 
    vp.AvgFuelEfficiency < 5.0 
    OR vp.TotalHarshEvents > 50 
    OR vp.AvgIdleTime > 60
ORDER BY PriorityScore DESC;
```

### Temporal Elasticity Scenario Analysis

```sql
-- Simulate impact of warehouse capacity change on fleet utilization
DECLARE @CapacityIncreasePct DECIMAL(5,2) = 0.15; -- 15% increase
DECLARE @SimulationDays INT = 90;

WITH BaselineMetrics AS (
    SELECT 
        dt.DateKey,
        COUNT(DISTINCT fwo.GeographyKey) AS ActiveWarehouses,
        SUM(fwo.QuantityProcessed) AS TotalWarehouseVolume,
        COUNT(DISTINCT fft.VehicleID) AS ActiveVehicles,
        SUM(fft.DistanceKM) AS TotalFleetDistance,
        AVG(fwo.DwellTimeHours) AS AvgDwellTime,
        AVG(fft.IdleTimeMinutes) AS AvgFleetIdle
    FROM dbo.DimTime dt
    LEFT JOIN dbo.FactWarehouseOper
