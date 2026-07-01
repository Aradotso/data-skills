---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact star schema data warehouse for logistics intelligence, fleet optimization, and warehouse analytics
triggers:
  - "set up logistics data warehouse"
  - "create supply chain analytics dashboard"
  - "deploy logifleet pulse system"
  - "configure warehouse and fleet tracking"
  - "implement multi-fact star schema for logistics"
  - "build power bi logistics reports"
  - "optimize warehouse gravity zones"
  - "create fleet telemetry analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is an advanced data warehousing and analytics platform for supply chain logistics. It integrates warehouse operations, fleet telemetry, inventory management, and external data sources into a unified multi-fact star schema on MS SQL Server, with Power BI dashboards for real-time visualization and predictive analytics.

## What This Project Does

- **Multi-Modal Data Integration**: Combines warehouse management, fleet GPS/telemetry, supplier data, and external APIs (weather, traffic) into a single semantic layer
- **Multi-Fact Star Schema**: Custom dimensional model with time-phased dimensions and bridge tables for many-to-many relationships
- **Real-Time Dashboards**: Power BI templates for warehouse operations, fleet optimization, and cross-functional KPIs
- **Predictive Analytics**: Bottleneck detection, maintenance prioritization, and temporal elasticity modeling
- **Cross-Fact KPI Harmonization**: Links inventory turnover with fleet fuel consumption, dwell time with route efficiency

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telemetry APIs)

### Step 1: Deploy SQL Schema

```sql
-- Execute the main schema deployment script
-- This creates all fact tables, dimensions, views, and stored procedures

USE master;
GO

-- Create the database
CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = 'LogiFleetPulse_Data',
    FILENAME = 'D:\SQLData\LogiFleetPulse.mdf',
    SIZE = 10GB,
    MAXSIZE = 500GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = 'LogiFleetPulse_Log',
    FILENAME = 'D:\SQLData\LogiFleetPulse_log.ldf',
    SIZE = 5GB,
    MAXSIZE = 100GB,
    FILEGROWTH = 512MB
);
GO

USE LogiFleetPulse;
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName NVARCHAR(20) NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL,
    FiscalPeriod INT NOT NULL,
    FiscalQuarter TINYINT NOT NULL,
    FiscalYear INT NOT NULL
);
GO

CREATE UNIQUE NONCLUSTERED INDEX IX_DimTime_FullDateTime 
ON DimTime(FullDateTime);
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- 'Warehouse', 'Distribution Center', 'Route Node'
    Address NVARCHAR(500),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL
);
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) NOT NULL,
    ProductName NVARCHAR(500) NOT NULL,
    Category NVARCHAR(200),
    SubCategory NVARCHAR(200),
    UnitWeight DECIMAL(10,2), -- kg
    UnitVolume DECIMAL(10,3), -- cubic meters
    Fragility TINYINT, -- 1-10 scale
    TemperatureRequirement NVARCHAR(50), -- 'Ambient', 'Refrigerated', 'Frozen'
    PickFrequencyScore DECIMAL(5,2), -- Calculated velocity
    ValueDensity DECIMAL(10,2), -- $ per unit
    GravityScore AS (PickFrequencyScore * ValueDensity * Fragility / 10), -- Computed column
    OptimalStorageZone NVARCHAR(50),
    IsActive BIT DEFAULT 1
);
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200) NOT NULL,
    SupplierType NVARCHAR(100),
    AverageLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4), -- Percentage
    OnTimeDeliveryRate DECIMAL(5,4), -- Percentage
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier NVARCHAR(20), -- 'Platinum', 'Gold', 'Silver', 'Bronze'
    IsActive BIT DEFAULT 1
);
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT NULL FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID NVARCHAR(100),
    BatchNumber NVARCHAR(100),
    QuantityUnits INT NOT NULL,
    CycleTimeMinutes DECIMAL(8,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone NVARCHAR(50),
    PickPathDistance DECIMAL(8,2), -- meters
    LabourMinutes DECIMAL(8,2),
    EquipmentID NVARCHAR(50),
    ErrorFlag BIT DEFAULT 0,
    ErrorCode NVARCHAR(50),
    CreatedTimestamp DATETIME2 DEFAULT GETDATE()
);
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time 
ON FactWarehouseOperations(TimeKey) INCLUDE (OperationType, QuantityUnits);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product 
ON FactWarehouseOperations(ProductKey, OperationType);
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyEnd INT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    LoadingTimeMinutes DECIMAL(8,2),
    UnloadingTimeMinutes DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(10,2),
    AverageSpeed DECIMAL(5,2),
    MaxSpeed DECIMAL(5,2),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,3),
    TollCost DECIMAL(10,2),
    MaintenanceFlag BIT DEFAULT 0,
    WeatherCondition NVARCHAR(50),
    TrafficDelayMinutes DECIMAL(8,2),
    OnTimeDelivery BIT,
    CreatedTimestamp DATETIME2 DEFAULT GETDATE()
);
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeStart 
ON FactFleetTrips(TimeKeyStart) INCLUDE (VehicleID, DistanceKm);

CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle 
ON FactFleetTrips(VehicleID, TimeKeyStart);
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyInbound INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKeyOutbound INT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    QuantityUnits INT NOT NULL,
    DockDwellMinutes DECIMAL(8,2),
    SortingTimeMinutes DECIMAL(8,2),
    ConsolidationFlag BIT DEFAULT 0,
    CreatedTimestamp DATETIME2 DEFAULT GETDATE()
);
GO

-- Create bridge table for many-to-many relationships
CREATE TABLE BridgeRoutesStorageZones (
    BridgeKey INT PRIMARY KEY IDENTITY(1,1),
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OperationKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    AllocationWeight DECIMAL(5,4) DEFAULT 1.0 -- For fractional attribution
);
GO
```

### Step 2: Configure Data Sources

Create a configuration file for data ingestion:

```json
{
  "dataSources": {
    "warehouseManagement": {
      "type": "SQL",
      "connectionString": "${WMS_CONNECTION_STRING}",
      "refreshInterval": "15min"
    },
    "fleetTelemetry": {
      "type": "REST_API",
      "endpoint": "${FLEET_API_ENDPOINT}",
      "authType": "Bearer",
      "token": "${FLEET_API_TOKEN}",
      "refreshInterval": "5min"
    },
    "supplierPortal": {
      "type": "SQL",
      "connectionString": "${ERP_CONNECTION_STRING}",
      "refreshInterval": "60min"
    },
    "weatherAPI": {
      "type": "REST_API",
      "endpoint": "https://api.weather.service/v1/current",
      "authType": "API_Key",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshInterval": "30min"
    }
  },
  "etl": {
    "batchSize": 5000,
    "errorHandling": "log_and_continue",
    "logRetentionDays": 30
  }
}
```

### Step 3: Deploy Incremental Load Procedures

```sql
-- Stored procedure for incremental warehouse operations load
CREATE OR ALTER PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentTimestamp DATETIME2 = GETDATE();
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Insert new operations from source system
        INSERT INTO FactWarehouseOperations (
            TimeKey, GeographyKey, ProductKey, SupplierKey,
            OperationType, OrderID, BatchNumber, QuantityUnits,
            CycleTimeMinutes, DwellTimeHours, StorageZone,
            PickPathDistance, LabourMinutes, EquipmentID
        )
        SELECT 
            t.TimeKey,
            g.GeographyKey,
            p.ProductKey,
            s.SupplierKey,
            src.OperationType,
            src.OrderID,
            src.BatchNumber,
            src.Quantity,
            DATEDIFF(MINUTE, src.StartTime, src.EndTime),
            DATEDIFF(HOUR, src.ArrivalTime, src.ProcessTime),
            src.StorageZone,
            src.PickPathMeters,
            src.LabourMinutes,
            src.EquipmentID
        FROM ExternalWMS.dbo.Operations src
        INNER JOIN DimTime t ON t.FullDateTime = DATEADD(MINUTE, 
            (DATEPART(MINUTE, src.StartTime) / 15) * 15, 
            CAST(CAST(src.StartTime AS DATE) AS DATETIME2) + 
            CAST(CAST(DATEPART(HOUR, src.StartTime) AS VARCHAR) + ':00:00' AS TIME))
        INNER JOIN DimGeography g ON g.LocationID = src.WarehouseID AND g.IsActive = 1
        INNER JOIN DimProductGravity p ON p.SKU = src.SKU AND p.IsActive = 1
        LEFT JOIN DimSupplierReliability s ON s.SupplierID = src.SupplierID AND s.IsActive = 1
        WHERE src.ProcessedTimestamp > @LastLoadTimestamp
            AND src.ProcessedTimestamp <= @CurrentTimestamp;
        
        COMMIT TRANSACTION;
        
        -- Log successful load
        INSERT INTO ETL_LoadLog (ProcedureName, LastLoadTimestamp, NewLoadTimestamp, RowsProcessed)
        VALUES ('usp_LoadWarehouseOperations', @LastLoadTimestamp, @CurrentTimestamp, @@ROWCOUNT);
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        -- Log error
        INSERT INTO ETL_ErrorLog (ProcedureName, ErrorMessage, ErrorTimestamp)
        VALUES ('usp_LoadWarehouseOperations', ERROR_MESSAGE(), GETDATE());
        
        THROW;
    END CATCH
END;
GO

-- Stored procedure for fleet trips incremental load
CREATE OR ALTER PROCEDURE usp_LoadFleetTrips
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentTimestamp DATETIME2 = GETDATE();
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        INSERT INTO FactFleetTrips (
            TimeKeyStart, TimeKeyEnd, OriginGeographyKey, DestinationGeographyKey,
            VehicleID, DriverID, DistanceKm, DurationMinutes, IdleTimeMinutes,
            LoadingTimeMinutes, UnloadingTimeMinutes, FuelConsumedLiters,
            AverageSpeed, MaxSpeed, LoadWeightKg, LoadVolumeM3,
            WeatherCondition, TrafficDelayMinutes, OnTimeDelivery
        )
        SELECT 
            ts.TimeKey AS TimeKeyStart,
            te.TimeKey AS TimeKeyEnd,
            go.GeographyKey AS OriginGeographyKey,
            gd.GeographyKey AS DestinationGeographyKey,
            src.VehicleID,
            src.DriverID,
            src.DistanceKm,
            DATEDIFF(MINUTE, src.StartTime, src.EndTime),
            src.IdleMinutes,
            src.LoadingMinutes,
            src.UnloadingMinutes,
            src.FuelLiters,
            src.DistanceKm / NULLIF(DATEDIFF(MINUTE, src.StartTime, src.EndTime) / 60.0, 0),
            src.MaxSpeedKmh,
            src.LoadKg,
            src.LoadM3,
            src.WeatherCondition,
            src.TrafficDelayMinutes,
            CASE WHEN src.ActualArrival <= src.ScheduledArrival THEN 1 ELSE 0 END
        FROM ExternalFleet.dbo.Trips src
        INNER JOIN DimTime ts ON ts.FullDateTime = DATEADD(MINUTE, 
            (DATEPART(MINUTE, src.StartTime) / 15) * 15, 
            CAST(CAST(src.StartTime AS DATE) AS DATETIME2) + 
            CAST(CAST(DATEPART(HOUR, src.StartTime) AS VARCHAR) + ':00:00' AS TIME))
        LEFT JOIN DimTime te ON te.FullDateTime = DATEADD(MINUTE, 
            (DATEPART(MINUTE, src.EndTime) / 15) * 15, 
            CAST(CAST(src.EndTime AS DATE) AS DATETIME2) + 
            CAST(CAST(DATEPART(HOUR, src.EndTime) AS VARCHAR) + ':00:00' AS TIME))
        INNER JOIN DimGeography go ON go.LocationID = src.OriginLocationID AND go.IsActive = 1
        INNER JOIN DimGeography gd ON gd.LocationID = src.DestinationLocationID AND gd.IsActive = 1
        WHERE src.ProcessedTimestamp > @LastLoadTimestamp
            AND src.ProcessedTimestamp <= @CurrentTimestamp;
        
        COMMIT TRANSACTION;
        
        INSERT INTO ETL_LoadLog (ProcedureName, LastLoadTimestamp, NewLoadTimestamp, RowsProcessed)
        VALUES ('usp_LoadFleetTrips', @LastLoadTimestamp, @CurrentTimestamp, @@ROWCOUNT);
        
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        
        INSERT INTO ETL_ErrorLog (ProcedureName, ErrorMessage, ErrorTimestamp)
        VALUES ('usp_LoadFleetTrips', ERROR_MESSAGE(), GETDATE());
        
        THROW;
    END CATCH
END;
GO
```

## Key SQL Views & Analytics

### Cross-Fact KPI View: Dwell Time vs Fleet Idling

```sql
CREATE OR ALTER VIEW vw_DwellTimeVsFleetIdling AS
SELECT 
    t.DateKey,
    t.FiscalPeriod,
    g.Region,
    p.Category,
    p.OptimalStorageZone,
    
    -- Warehouse metrics
    AVG(w.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(w.QuantityUnits) AS TotalUnitsProcessed,
    AVG(w.CycleTimeMinutes) AS AvgCycleTimeMinutes,
    
    -- Fleet metrics
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(f.FuelConsumedLiters) AS AvgFuelConsumed,
    AVG(CAST(f.OnTimeDelivery AS FLOAT)) AS OnTimeDeliveryRate,
    
    -- Cross-fact correlation
    CORR(w.DwellTimeHours, f.IdleTimeMinutes) OVER (PARTITION BY g.Region) AS DwellIdleCorrelation
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
LEFT JOIN BridgeRoutesStorageZones b ON w.OperationKey = b.OperationKey
LEFT JOIN FactFleetTrips f ON b.TripKey = f.TripKey
WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    AND w.OperationType IN ('Picking', 'Packing', 'Shipping')
GROUP BY t.DateKey, t.FiscalPeriod, g.Region, p.Category, p.OptimalStorageZone;
GO
```

### Predictive Bottleneck Detection View

```sql
CREATE OR ALTER VIEW vw_PredictiveBottlenecks AS
WITH OperationVelocity AS (
    SELECT 
        GeographyKey,
        ProductKey,
        StorageZone,
        DATEPART(HOUR, t.FullDateTime) AS HourOfDay,
        DATEPART(WEEKDAY, t.FullDateTime) AS DayOfWeek,
        AVG(CycleTimeMinutes) AS AvgCycleTime,
        STDEV(CycleTimeMinutes) AS StdDevCycleTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY GeographyKey, ProductKey, StorageZone, 
             DATEPART(HOUR, t.FullDateTime), DATEPART(WEEKDAY, t.FullDateTime)
),
FleetCongestion AS (
    SELECT 
        DestinationGeographyKey,
        DATEPART(HOUR, ts.FullDateTime) AS HourOfDay,
        DATEPART(WEEKDAY, ts.FullDateTime) AS DayOfWeek,
        AVG(TrafficDelayMinutes) AS AvgTrafficDelay,
        STDEV(TrafficDelayMinutes) AS StdDevTrafficDelay
    FROM FactFleetTrips f
    INNER JOIN DimTime ts ON f.TimeKeyStart = ts.TimeKey
    WHERE ts.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY DestinationGeographyKey, 
             DATEPART(HOUR, ts.FullDateTime), DATEPART(WEEKDAY, ts.FullDateTime)
)
SELECT 
    g.LocationName,
    p.ProductName,
    ov.StorageZone,
    ov.HourOfDay,
    ov.DayOfWeek,
    ov.AvgCycleTime,
    ov.StdDevCycleTime,
    fc.AvgTrafficDelay,
    -- Bottleneck risk score (0-100)
    CASE 
        WHEN ov.StdDevCycleTime > ov.AvgCycleTime * 0.5 
             AND fc.AvgTrafficDelay > 15 
        THEN 100
        WHEN ov.StdDevCycleTime > ov.AvgCycleTime * 0.3 
             OR fc.AvgTrafficDelay > 10 
        THEN 75
        WHEN ov.StdDevCycleTime > ov.AvgCycleTime * 0.2 
             OR fc.AvgTrafficDelay > 5 
        THEN 50
        ELSE 25
    END AS BottleneckRiskScore,
    ov.OperationCount AS SampleSize
FROM OperationVelocity ov
INNER JOIN DimGeography g ON ov.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON ov.ProductKey = p.ProductKey
LEFT JOIN FleetCongestion fc ON ov.GeographyKey = fc.DestinationGeographyKey
    AND ov.HourOfDay = fc.HourOfDay
    AND ov.DayOfWeek = fc.DayOfWeek
WHERE ov.OperationCount >= 10; -- Minimum sample size
GO
```

### Warehouse Gravity Zone Optimization

```sql
CREATE OR ALTER PROCEDURE usp_RecommendGravityZoneRealignment
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate actual vs optimal zone placement
    WITH ActualPerformance AS (
        SELECT 
            w.ProductKey,
            w.StorageZone,
            AVG(w.CycleTimeMinutes) AS AvgCycleTime,
            AVG(w.PickPathDistance) AS AvgPickPath,
            COUNT(*) AS PickCount
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE w.OperationType = 'Picking'
            AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY w.ProductKey, w.StorageZone
    ),
    OptimalZones AS (
        SELECT 
            ProductKey,
            CASE 
                WHEN GravityScore >= 75 THEN 'Zone_A_HighGravity'
                WHEN GravityScore >= 50 THEN 'Zone_B_MediumGravity'
                WHEN GravityScore >= 25 THEN 'Zone_C_LowGravity'
                ELSE 'Zone_D_Overflow'
            END AS OptimalZone
        FROM DimProductGravity
        WHERE IsActive = 1
    )
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore,
        ap.StorageZone AS CurrentZone,
        oz.OptimalZone AS RecommendedZone,
        ap.AvgCycleTime AS CurrentAvgCycleTime,
        ap.AvgPickPath AS CurrentAvgPickPath,
        ap.PickCount,
        -- Estimated improvement
        CASE 
            WHEN oz.OptimalZone < ap.StorageZone 
            THEN ap.AvgCycleTime * 0.8 -- 20% improvement estimate
            ELSE ap.AvgCycleTime * 1.1 -- 10% degradation if misplaced
        END AS EstimatedNewCycleTime,
        -- Priority score
        (p.GravityScore * ap.PickCount) / NULLIF(ap.AvgCycleTime, 0) AS RealignmentPriority
    FROM ActualPerformance ap
    INNER JOIN DimProductGravity p ON ap.ProductKey = p.ProductKey
    INNER JOIN OptimalZones oz ON ap.ProductKey = oz.ProductKey
    WHERE ap.StorageZone <> oz.OptimalZone
        AND ap.PickCount >= 10
    ORDER BY RealignmentPriority DESC;
END;
GO
```

## Power BI Configuration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_NAME}`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)

### DAX Measures for Cross-Fact Analysis

```dax
// Total Dwell Cost (combines warehouse dwell with storage cost rate)
Total Dwell Cost = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeHours] * 
    RELATED(DimProductGravity[ValueDensity]) * 
    0.05 // Storage cost per hour per $ of value
)

// Fleet Efficiency Score (composite metric)
Fleet Efficiency Score = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKm])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalDuration = SUM(FactFleetTrips[DurationMinutes])
VAR OnTimeRate = AVERAGE(FactFleetTrips[OnTimeDelivery])
RETURN
    (TotalDistance / NULLIF(TotalDuration, 0)) * 
    (1 - (TotalIdleTime / NULLIF(TotalDuration, 0))) * 
    OnTimeRate * 100

// Cross-Fact: Warehouse-to-Fleet Correlation
Warehouse Fleet Correlation = 
VAR WarehouseDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR FleetIdle = 
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        USERELATIONSHIP(BridgeRoutesStorageZones[TripKey], FactFleetTrips[TripKey])
    )
RETURN
    IF(
        ISBLANK(FleetIdle), 
        BLANK(),
        WarehouseDwell / NULLIF(FleetIdle, 0)
    )

// Predictive Bottleneck Alert Count
Bottleneck Alert Count = 
CALCULATE(
    COUNTROWS(vw_PredictiveBottlenecks),
    vw_PredictiveBottlenecks[BottleneckRiskScore] >= 75
)

// Gravity Zone Misalignment Percentage
Misalignment % = 
VAR TotalProducts = DISTINCTCOUNT(DimProductGravity[ProductKey])
VAR MisalignedProducts = 
    CALCULATE(
        DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
        FactWarehouseOperations[StorageZone] <> RELATED(DimProductGravity[OptimalStorageZone])
    )
RETURN
    DIVIDE(MisalignedProducts, TotalProducts, 0) * 100
```

### Row-Level Security (RLS)

```dax
// Table: SecurityRoles
// Define RLS rule for regional managers

[RegionFilter] = 
VAR UserEmail = USERPRINCIPALNAME()
VAR UserRegion = 
    LOOKUPVALUE(
        SecurityRoles[Region],
        SecurityRoles[Email], UserEmail
    )
RETURN
    DimGeography[Region] = UserRegion || UserRegion = "Global"
```

Apply this filter to `DimGeography` table in Power BI Desktop → Modeling → Manage Roles.

## Automated Alerting System

### SQL Agent Job for Threshold Monitoring

```sql
-- Create alert configuration table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    MetricName NVARCHAR(200) NOT NULL,
    ThresholdValue DECIMAL(18,4) NOT NULL,
    ComparisonOperator NVARCHAR(10) NOT NULL, -- '>', '<', '>=', '<=', '='
    AlertMessage NVARCHAR(1000) NOT NULL,
