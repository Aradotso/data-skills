---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence engine with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure multi-fact star schema for fleet and warehouse
  - implement warehouse gravity zones and fleet telemetry tracking
  - create supply chain kpi dashboards with logifleet
  - build cross-modal logistics intelligence system
  - integrate warehouse and fleet data with power bi
  - set up predictive bottleneck detection for logistics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external signals into a multi-fact star schema data warehouse. Built on MS SQL Server with Power BI visualization, it enables cross-fact KPI analysis like correlating warehouse dwell time with fleet fuel consumption or predicting bottlenecks before they occur.

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions (15-minute granularity)
- Warehouse gravity zones mapping (spatial optimization)
- Fleet telemetry integration and predictive maintenance
- Cross-fact KPI harmonization (warehouse + fleet + supply chain)
- Real-time Power BI dashboards with role-based security
- Temporal elasticity modeling for scenario planning

**Primary Language:** SQL (MS SQL Server), Power BI (DAX/M)

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Enterprise or Standard edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telemetry APIs

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
    FILENAME = 'C:\SQLData\LogiFleetPulse_Data.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = 'LogiFleetPulse_Log',
    FILENAME = 'C:\SQLData\LogiFleetPulse_Log.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

USE LogiFleetPulse;
GO

-- Create schemas for organization
CREATE SCHEMA Warehouse AUTHORIZATION dbo;
CREATE SCHEMA Fleet AUTHORIZATION dbo;
CREATE SCHEMA Dim AUTHORIZATION dbo;
CREATE SCHEMA Bridge AUTHORIZATION dbo;
GO
```

### Step 2: Create Core Dimension Tables

```sql
-- DimTime: Time-phased dimension with 15-minute granularity
CREATE TABLE Dim.DimTime (
    TimeKey INT IDENTITY(1,1) PRIMARY KEY,
    FullDateTime DATETIME2(0) NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot15Min SMALLINT NOT NULL, -- 0-95 (15-min buckets per day)
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    QuarterNumber TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL,
    HolidayName NVARCHAR(100) NULL
);

CREATE UNIQUE NONCLUSTERED INDEX IX_DimTime_FullDateTime 
ON Dim.DimTime(FullDateTime);

-- DimGeography: Hierarchical location dimension
CREATE TABLE Dim.DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, DistributionCenter, RouteNode
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL,
    IsCurrent BIT NOT NULL DEFAULT 1
);

-- DimProductGravity: Products with gravity score for warehouse optimization
CREATE TABLE Dim.DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2), -- kg
    UnitVolume DECIMAL(10,3), -- cubic meters
    IsFragile BIT NOT NULL DEFAULT 0,
    RequiresColdStorage BIT NOT NULL DEFAULT 0,
    -- Gravity score components
    VelocityScore DECIMAL(5,2) NOT NULL, -- Pick frequency weight
    ValueScore DECIMAL(5,2) NOT NULL, -- Monetary value weight
    FragilityScore DECIMAL(5,2) NOT NULL,
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    OptimalZoneType NVARCHAR(50), -- FastMover, SlowMover, Bulk, etc.
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL,
    IsCurrent BIT NOT NULL DEFAULT 1
);

-- DimSupplierReliability: Supplier performance metrics
CREATE TABLE Dim.DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode NVARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200) NOT NULL,
    SupplierCountry NVARCHAR(100),
    AvgLeadTimeDays DECIMAL(6,2),
    LeadTimeVariance DECIMAL(6,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityRating NVARCHAR(20), -- Excellent, Good, Fair, Poor
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE NULL,
    IsCurrent BIT NOT NULL DEFAULT 1
);
```

### Step 3: Create Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE Warehouse.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimProductGravity(ProductKey),
    SupplierKey INT NULL FOREIGN KEY REFERENCES Dim.DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderNumber NVARCHAR(50),
    Quantity INT NOT NULL,
    DwellTimeMinutes INT NULL, -- Time from receiving to putaway or picking to shipping
    CycleTimeMinutes INT NULL, -- Time to complete operation
    StorageZone NVARCHAR(50),
    AssignedGravityZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50), -- Forklift, pallet jack, etc.
    IsDelayed BIT NOT NULL DEFAULT 0,
    DelayReasonCode NVARCHAR(50),
    TotalCost DECIMAL(12,2)
);

CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time 
ON Warehouse.FactWarehouseOperations(TimeKey) 
INCLUDE (GeographyKey, ProductKey, Quantity);

CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Product 
ON Warehouse.FactWarehouseOperations(ProductKey) 
INCLUDE (TimeKey, DwellTimeMinutes);

-- FactFleetTrips: Fleet telemetry and trip data
CREATE TABLE Fleet.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimTime(TimeKey),
    EndTimeKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    RouteID NVARCHAR(50),
    TotalDistanceKm DECIMAL(10,2),
    TotalDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    AverageSpeed DECIMAL(6,2),
    MaxSpeed DECIMAL(6,2),
    HarshBrakingEvents INT DEFAULT 0,
    RapidAccelerationEvents INT DEFAULT 0,
    IsDelayed BIT NOT NULL DEFAULT 0,
    DelayReasonCode NVARCHAR(50),
    WeatherCondition NVARCHAR(50),
    TrafficCondition NVARCHAR(50),
    TotalCost DECIMAL(12,2)
);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_StartTime 
ON Fleet.FactFleetTrips(StartTimeKey) 
INCLUDE (VehicleID, FuelConsumedLiters, IdleTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Vehicle 
ON Fleet.FactFleetTrips(VehicleID) 
INCLUDE (StartTimeKey, TotalDistanceKm);

-- FactCrossDock: Cross-dock transfer operations
CREATE TABLE Warehouse.FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimTime(TimeKey),
    OutboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES Fleet.FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES Fleet.FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    CrossDockDurationMinutes INT NOT NULL,
    IsDirectTransfer BIT NOT NULL, -- No intermediate storage
    TotalCost DECIMAL(12,2)
);
```

### Step 4: Create Bridge Tables for Many-to-Many Relationships

```sql
-- Bridge table for routes to storage zones
CREATE TABLE Bridge.RouteToZone (
    RouteID NVARCHAR(50) NOT NULL,
    StorageZone NVARCHAR(50) NOT NULL,
    PickSequenceOrder INT,
    TypicalDwellTimeMinutes INT,
    PRIMARY KEY (RouteID, StorageZone)
);

-- Bridge table for products to multiple storage locations
CREATE TABLE Bridge.ProductToLocation (
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimProductGravity(ProductKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES Dim.DimGeography(GeographyKey),
    StorageZone NVARCHAR(50) NOT NULL,
    QuantityOnHand INT DEFAULT 0,
    ReorderPoint INT,
    LastReplenishmentDate DATE,
    PRIMARY KEY (ProductKey, GeographyKey, StorageZone)
);
```

## Key Stored Procedures & ETL Patterns

### Incremental Load for Warehouse Operations

```sql
CREATE PROCEDURE Warehouse.usp_LoadWarehouseOperations
    @LoadFromDate DATETIME2,
    @LoadToDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge pattern for incremental loads from staging
    MERGE Warehouse.FactWarehouseOperations AS target
    USING (
        SELECT 
            dt.TimeKey,
            dg.GeographyKey,
            dp.ProductKey,
            ds.SupplierKey,
            stg.OperationType,
            stg.OrderNumber,
            stg.Quantity,
            DATEDIFF(MINUTE, stg.ReceiveTimestamp, stg.PutawayTimestamp) AS DwellTimeMinutes,
            DATEDIFF(MINUTE, stg.OperationStartTime, stg.OperationEndTime) AS CycleTimeMinutes,
            stg.StorageZone,
            dp.OptimalZoneType AS AssignedGravityZone,
            stg.OperatorID,
            stg.EquipmentID,
            CASE WHEN stg.PlannedCompletionTime < stg.OperationEndTime THEN 1 ELSE 0 END AS IsDelayed,
            stg.DelayReasonCode,
            stg.TotalCost
        FROM Staging.WarehouseOperations stg
        INNER JOIN Dim.DimTime dt ON CAST(stg.OperationStartTime AS DATE) = CAST(dt.FullDateTime AS DATE)
            AND DATEPART(HOUR, stg.OperationStartTime) = dt.HourOfDay
            AND (DATEPART(MINUTE, stg.OperationStartTime) / 15) = (dt.TimeSlot15Min % 4)
        INNER JOIN Dim.DimGeography dg ON stg.WarehouseCode = dg.LocationCode AND dg.IsCurrent = 1
        INNER JOIN Dim.DimProductGravity dp ON stg.SKU = dp.SKU AND dp.IsCurrent = 1
        LEFT JOIN Dim.DimSupplierReliability ds ON stg.SupplierCode = ds.SupplierCode AND ds.IsCurrent = 1
        WHERE stg.OperationStartTime >= @LoadFromDate
            AND stg.OperationStartTime < @LoadToDate
            AND stg.IsProcessed = 0
    ) AS source
    ON 1 = 0 -- Always insert for fact table
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, GeographyKey, ProductKey, SupplierKey, OperationType, OrderNumber,
                Quantity, DwellTimeMinutes, CycleTimeMinutes, StorageZone, AssignedGravityZone,
                OperatorID, EquipmentID, IsDelayed, DelayReasonCode, TotalCost)
        VALUES (source.TimeKey, source.GeographyKey, source.ProductKey, source.SupplierKey,
                source.OperationType, source.OrderNumber, source.Quantity, source.DwellTimeMinutes,
                source.CycleTimeMinutes, source.StorageZone, source.AssignedGravityZone,
                source.OperatorID, source.EquipmentID, source.IsDelayed, source.DelayReasonCode,
                source.TotalCost);
    
    -- Mark staging records as processed
    UPDATE Staging.WarehouseOperations
    SET IsProcessed = 1, ProcessedTimestamp = GETDATE()
    WHERE OperationStartTime >= @LoadFromDate
        AND OperationStartTime < @LoadToDate
        AND IsProcessed = 0;
END;
GO
```

### Calculate Warehouse Gravity Zone Recommendations

```sql
CREATE PROCEDURE Warehouse.usp_CalculateGravityZoneRecommendations
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity score based on recent pick frequency
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            SUM(Quantity) AS TotalQuantity,
            AVG(CycleTimeMinutes) AS AvgCycleTime
        FROM Warehouse.FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT MAX(TimeKey) - 2880 FROM Dim.DimTime) -- Last 30 days
        GROUP BY ProductKey
    ),
    ScoreCalculation AS (
        SELECT 
            dp.ProductKey,
            dp.SKU,
            dp.ProductName,
            COALESCE(pv.PickCount, 0) AS RecentPickCount,
            -- Normalize to 0-100 scale
            CASE 
                WHEN COALESCE(pv.PickCount, 0) >= 100 THEN 100
                ELSE COALESCE(pv.PickCount, 0)
            END AS VelocityScore,
            dp.ValueScore,
            dp.FragilityScore,
            dp.OptimalZoneType AS CurrentZoneType
        FROM Dim.DimProductGravity dp
        LEFT JOIN ProductVelocity pv ON dp.ProductKey = pv.ProductKey
        WHERE dp.IsCurrent = 1
    )
    SELECT 
        ProductKey,
        SKU,
        ProductName,
        VelocityScore,
        ValueScore,
        FragilityScore,
        (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) AS NewGravityScore,
        CurrentZoneType,
        CASE 
            WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) >= 75 THEN 'HighGravity_FastMover'
            WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) >= 50 THEN 'MediumGravity_Regular'
            ELSE 'LowGravity_SlowMover'
        END AS RecommendedZoneType,
        CASE 
            WHEN CurrentZoneType != 
                CASE 
                    WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) >= 75 THEN 'HighGravity_FastMover'
                    WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) >= 50 THEN 'MediumGravity_Regular'
                    ELSE 'LowGravity_SlowMover'
                END
            THEN 'Rezone Recommended'
            ELSE 'Optimal'
        END AS RecommendationStatus
    FROM ScoreCalculation
    ORDER BY NewGravityScore DESC;
END;
GO
```

### Predictive Bottleneck Detection

```sql
CREATE VIEW Warehouse.vw_BottleneckPrediction
AS
WITH HistoricalBaseline AS (
    SELECT 
        dt.HourOfDay,
        dt.DayOfWeek,
        wo.StorageZone,
        AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
        STDEV(wo.CycleTimeMinutes) AS StdDevCycleTime,
        COUNT(*) AS OperationCount
    FROM Warehouse.FactWarehouseOperations wo
    INNER JOIN Dim.DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.TimeKey >= (SELECT MAX(TimeKey) - 8640 FROM Dim.DimTime) -- Last 90 days
    GROUP BY dt.HourOfDay, dt.DayOfWeek, wo.StorageZone
),
CurrentPerformance AS (
    SELECT 
        dt.HourOfDay,
        dt.DayOfWeek,
        wo.StorageZone,
        AVG(wo.CycleTimeMinutes) AS CurrentAvgCycleTime,
        COUNT(*) AS CurrentOperationCount
    FROM Warehouse.FactWarehouseOperations wo
    INNER JOIN Dim.DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.TimeKey >= (SELECT MAX(TimeKey) - 96 FROM Dim.DimTime) -- Last day
    GROUP BY dt.HourOfDay, dt.DayOfWeek, wo.StorageZone
)
SELECT 
    cp.HourOfDay,
    cp.DayOfWeek,
    cp.StorageZone,
    hb.AvgCycleTime AS BaselineCycleTime,
    cp.CurrentAvgCycleTime,
    hb.OperationCount AS BaselineVolume,
    cp.CurrentOperationCount AS CurrentVolume,
    -- Bottleneck score: combines cycle time deviation and volume increase
    CASE 
        WHEN cp.CurrentAvgCycleTime > hb.AvgCycleTime + (2 * hb.StdDevCycleTime)
            AND cp.CurrentOperationCount > hb.OperationCount * 1.2
        THEN 'Critical'
        WHEN cp.CurrentAvgCycleTime > hb.AvgCycleTime + hb.StdDevCycleTime
            AND cp.CurrentOperationCount > hb.OperationCount * 1.1
        THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckStatus,
    ((cp.CurrentAvgCycleTime - hb.AvgCycleTime) / NULLIF(hb.AvgCycleTime, 0)) * 100 AS PercentSlowdown,
    ((cp.CurrentOperationCount - hb.OperationCount) / NULLIF(hb.OperationCount, 0)) * 100 AS PercentVolumeIncrease
FROM CurrentPerformance cp
INNER JOIN HistoricalBaseline hb 
    ON cp.HourOfDay = hb.HourOfDay
    AND cp.DayOfWeek = hb.DayOfWeek
    AND cp.StorageZone = hb.StorageZone
WHERE cp.CurrentAvgCycleTime > hb.AvgCycleTime;
GO
```

## Power BI Integration & DAX Measures

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-sql-server.database.windows.net` (use environment variable)
4. Database: `LogiFleetPulse`
5. Data Connectivity Mode: **Import** (for best performance) or **DirectQuery** (for real-time)
6. Select tables: All `Dim.*`, `Warehouse.*`, `Fleet.*`, and `Bridge.*` tables

### Key DAX Measures for Cross-Fact Analysis

```dax
// Total Warehouse Dwell Time (Hours)
Total Dwell Time = 
SUMX(
    Warehouse.FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Average Fleet Idle Time Percentage
Avg Fleet Idle % = 
DIVIDE(
    SUM(Fleet.FactFleetTrips[IdleTimeMinutes]),
    SUM(Fleet.FactFleetTrips[TotalDurationMinutes]),
    0
) * 100

// Cross-Fact: Dwell Time Impact on Fleet Utilization
// Correlates high warehouse dwell with downstream fleet delays
Dwell-Fleet Correlation Score = 
VAR HighDwellProducts = 
    CALCULATETABLE(
        VALUES(DimProductGravity[ProductKey]),
        Warehouse.FactWarehouseOperations[DwellTimeMinutes] > 240 -- Over 4 hours
    )
VAR FleetTripsWithHighDwellProducts = 
    CALCULATE(
        COUNTROWS(Fleet.FactFleetTrips),
        Fleet.FactFleetTrips[IsDelayed] = TRUE(),
        FILTER(
            Bridge.ProductToLocation,
            Bridge.ProductToLocation[ProductKey] IN HighDwellProducts
        )
    )
VAR TotalDelayedTrips = 
    CALCULATE(
        COUNTROWS(Fleet.FactFleetTrips),
        Fleet.FactFleetTrips[IsDelayed] = TRUE()
    )
RETURN
DIVIDE(FleetTripsWithHighDwellProducts, TotalDelayedTrips, 0) * 100

// Warehouse Gravity Zone Efficiency
Gravity Zone Efficiency % = 
VAR OptimalZonePicks = 
    CALCULATE(
        COUNTROWS(Warehouse.FactWarehouseOperations),
        Warehouse.FactWarehouseOperations[StorageZone] = Warehouse.FactWarehouseOperations[AssignedGravityZone]
    )
VAR TotalPicks = COUNTROWS(Warehouse.FactWarehouseOperations)
RETURN
DIVIDE(OptimalZonePicks, TotalPicks, 0) * 100

// Fuel Cost per Delivery (Cross-Fact)
Fuel Cost Per Delivery = 
VAR TotalFuelCost = 
    SUMX(
        Fleet.FactFleetTrips,
        Fleet.FactFleetTrips[FuelConsumedLiters] * 1.5 -- $1.50 per liter, use param
    )
VAR TotalDeliveries = 
    CALCULATE(
        COUNTROWS(Warehouse.FactWarehouseOperations),
        Warehouse.FactWarehouseOperations[OperationType] = "Shipping"
    )
RETURN
DIVIDE(TotalFuelCost, TotalDeliveries, 0)

// Predictive Bottleneck Index (0-100 scale)
Bottleneck Risk Index = 
VAR CurrentCycleTime = AVERAGE(Warehouse.FactWarehouseOperations[CycleTimeMinutes])
VAR BaselineCycleTime = 
    CALCULATE(
        AVERAGE(Warehouse.FactWarehouseOperations[CycleTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -90, DAY)
    )
VAR CycleTimeRatio = DIVIDE(CurrentCycleTime, BaselineCycleTime, 1)
VAR DelayRate = 
    DIVIDE(
        CALCULATE(COUNTROWS(Warehouse.FactWarehouseOperations), Warehouse.FactWarehouseOperations[IsDelayed] = TRUE()),
        COUNTROWS(Warehouse.FactWarehouseOperations),
        0
    )
RETURN
MINX({100}, (CycleTimeRatio - 1) * 50 + DelayRate * 100)
```

### Creating Row-Level Security

```dax
// In Power BI Desktop, Modeling tab → Manage Roles → Create Role "WarehouseManager"
[GeographyKey] IN LOOKUPVALUE(
    'UserPermissions'[GeographyKey],
    'UserPermissions'[UserEmail], USERPRINCIPALNAME(),
    'UserPermissions'[Role], "WarehouseManager"
)

// For executive access (all data)
1 = 1
```

## Configuration Patterns

### External Data Source Connection (Polybase Example)

```sql
-- Enable external data sources for streaming telemetry
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '$(MASTER_KEY_PASSWORD)';
GO

CREATE DATABASE SCOPED CREDENTIAL TelemetryAPICredential
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = '$(TELEMETRY_API_SAS_TOKEN)';
GO

CREATE EXTERNAL DATA SOURCE TelemetryBlobStorage
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://$(STORAGE_ACCOUNT_NAME).blob.core.windows.net/telemetry',
    CREDENTIAL = TelemetryAPICredential
);
GO

-- External table for real-time fleet telemetry ingestion
CREATE EXTERNAL TABLE Fleet.ExtFleetTelemetry (
    VehicleID NVARCHAR(50),
    Timestamp DATETIME2,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Speed DECIMAL(6,2),
    FuelLevel DECIMAL(5,2),
    EngineTemp DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = TelemetryBlobStorage,
    LOCATION = 'live-feed/',
    FILE_FORMAT = JSONFormat
);
```

### Automated Alerting Configuration

```sql
CREATE TABLE Config.AlertThresholds (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    MetricName NVARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ComparisonOperator NVARCHAR(10) NOT NULL, -- '>', '<', '>=', '<=', '='
    AlertSeverity NVARCHAR(20) NOT NULL, -- Critical, Warning, Info
    NotificationEmail NVARCHAR(200),
    IsActive BIT NOT NULL DEFAULT 1
);

-- Sample thresholds
INSERT INTO Config.AlertThresholds (MetricName, ThresholdValue, ComparisonOperator, AlertSeverity, NotificationEmail)
VALUES 
    ('FleetIdleTimePercent', 15.0, '>', 'Warning', '$(ALERT_EMAIL_FLEET)'),
    ('WarehouseDwellTimeHours', 72.0, '>', 'Critical', '$(ALERT_EMAIL_WAREHOUSE)'),
    ('BottleneckRiskIndex', 75.0, '>', 'Warning', '$(ALERT_EMAIL_OPS)'),
    ('FuelCostPerDelivery', 25.0, '>', 'Info', '$(ALERT_EMAIL_FINANCE)');

-- Stored procedure to check thresholds and send alerts
CREATE PROCEDURE Config.usp_CheckAlertThresholds
AS
BEGIN
    -- Check fleet idle time
    INSERT INTO Audit.AlertLog (AlertID, MetricValue, AlertTimestamp)
    SELECT 
        at.AlertID,
        (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.TotalDurationMinutes), 0)) AS MetricValue,
        GETDATE()
    FROM Config.AlertThresholds at
    CROSS APPLY (
        SELECT IdleTimeMinutes, TotalDurationMinutes
        FROM Fleet.FactFleetTrips
        WHERE StartTimeKey >= (SELECT MAX(TimeKey) - 96 FROM Dim.DimTime) -- Last day
    ) ft
    WHERE at.MetricName = 'FleetIdleTimePercent'
        AND at.IsActive = 1
        AND (SUM(ft.IdleTimeMinutes
