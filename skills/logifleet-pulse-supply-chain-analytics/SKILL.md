---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server data warehousing and Power BI dashboards for supply chain optimization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI logistics dashboard"
  - "implement warehouse gravity zone analytics"
  - "create fleet telemetry fact tables"
  - "build cross-fact supply chain KPIs"
  - "set up real-time logistics alerts"
  - "integrate warehouse and fleet data models"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. Built on MS SQL Server with Power BI visualization, it provides multi-fact star schema architecture for real-time supply chain analytics, predictive bottleneck detection, and cross-modal KPI harmonization.

## Core Capabilities

- **Multi-fact star schema** with time-phased dimensions for warehouse and fleet operations
- **Warehouse Gravity Zones™** spatial optimization based on pick frequency and item value
- **Fleet triage engine** with proactive maintenance prioritization
- **Cross-fact KPI harmonization** linking inventory turnover to fuel consumption
- **Temporal elasticity modeling** for scenario simulation
- **Real-time dashboards** with 15-minute refresh cycles
- **Role-based access control** with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### 1. Deploy SQL Schema

```sql
-- Execute the main schema deployment script
-- Assumes you have a database named 'LogisticsDataWarehouse'

USE master;
GO

IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = 'LogisticsDataWarehouse')
BEGIN
    CREATE DATABASE LogisticsDataWarehouse;
END
GO

USE LogisticsDataWarehouse;
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot15Min TINYINT NOT NULL, -- 0-95 (15-min buckets in 24h)
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime;
CREATE UNIQUE NONCLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime);
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, Route Node, Customer Site
    Street NVARCHAR(200),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_DimGeography_Type ON DimGeography(LocationType) WHERE IsActive = 1;
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3), -- kg
    UnitVolume DECIMAL(10,3), -- cubic meters
    UnitValue DECIMAL(12,2), -- currency
    FragilityScore TINYINT DEFAULT 1, -- 1-10 scale
    PickFrequencyScore DECIMAL(5,2) DEFAULT 0, -- calculated metric
    GravityZone NVARCHAR(20), -- High, Medium, Low based on composite score
    IsActive BIT DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_DimProduct_GravityZone ON DimProductGravity(GravityZone) WHERE IsActive = 1;
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeDaysAvg DECIMAL(5,1),
    LeadTimeDaysStdDev DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier NVARCHAR(20), -- A, B, C, D
    IsActive BIT DEFAULT 1
);
GO

CREATE TABLE DimFleetVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) NOT NULL UNIQUE,
    VehicleType NVARCHAR(50), -- Truck, Van, Cargo Bike
    Manufacturer NVARCHAR(100),
    Model NVARCHAR(100),
    YearManufactured SMALLINT,
    MaxLoadCapacityKg DECIMAL(10,2),
    FuelType NVARCHAR(50), -- Diesel, Electric, Hybrid
    LastMaintenanceDate DATE,
    IsActive BIT DEFAULT 1
);
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2,
    DurationMinutes DECIMAL(8,2),
    QuantityHandled INT NOT NULL,
    DwellTimeHours DECIMAL(10,2), -- time in storage before operation
    ZoneAssigned NVARCHAR(50), -- Storage zone identifier
    OperatorID NVARCHAR(50),
    EquipmentUsed NVARCHAR(100), -- Forklift, Pallet Jack, etc.
    ErrorFlag BIT DEFAULT 0,
    ErrorDescription NVARCHAR(500)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations ON FactWarehouseOperations;
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimFleetVehicle(VehicleKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2,
    TripDurationMinutes DECIMAL(8,2),
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,3),
    IdleTimeMinutes DECIMAL(8,2),
    LoadingTimeMinutes DECIMAL(8,2),
    UnloadingTimeMinutes DECIMAL(8,2),
    LoadWeightKg DECIMAL(10,2),
    DriverID NVARCHAR(50),
    RouteID NVARCHAR(50),
    WeatherCondition NVARCHAR(50), -- Clear, Rain, Snow, Fog
    TrafficDelayMinutes DECIMAL(8,2) DEFAULT 0,
    CompletionStatus NVARCHAR(50) -- Completed, Delayed, Cancelled
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips ON FactFleetTrips;
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey);
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ArrivalTime DATETIME2 NOT NULL,
    DepartureTime DATETIME2,
    TransferDurationMinutes DECIMAL(8,2),
    QuantityTransferred INT NOT NULL,
    CrossDockZone NVARCHAR(50)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactCrossDock ON FactCrossDock;
GO
```

### 2. Create Stored Procedures for Data Loading

```sql
-- Procedure to calculate and update Product Gravity Scores
CREATE OR ALTER PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(DISTINCT w.OperationKey) AS PickCount,
            AVG(w.DwellTimeHours) AS AvgDwellTime,
            p.UnitValue,
            p.FragilityScore
        FROM DimProductGravity p
        LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
        WHERE w.OperationType = 'Picking'
          AND w.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY p.ProductKey, p.UnitValue, p.FragilityScore
    )
    UPDATE p
    SET 
        PickFrequencyScore = CASE 
            WHEN pm.PickCount IS NULL THEN 0
            WHEN pm.PickCount >= 100 THEN 10
            ELSE (pm.PickCount / 10.0)
        END,
        GravityZone = CASE
            WHEN (pm.PickCount >= 50 AND p.UnitValue > 100) OR p.FragilityScore >= 8 THEN 'High'
            WHEN pm.PickCount >= 20 OR (p.UnitValue > 50 AND p.FragilityScore >= 5) THEN 'Medium'
            ELSE 'Low'
        END
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
    
    PRINT 'Product gravity scores updated successfully.';
END
GO

-- Procedure to generate predictive bottleneck alerts
CREATE OR ALTER PROCEDURE sp_GenerateBottleneckAlerts
    @ThresholdMinutes INT = 30
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        g.LocationName AS WarehouseLocation,
        p.GravityZone,
        COUNT(w.OperationKey) AS OperationsInQueue,
        AVG(w.DurationMinutes) AS AvgProcessingTime,
        SUM(CASE WHEN w.DurationMinutes > @ThresholdMinutes THEN 1 ELSE 0 END) AS DelayedOperations,
        MAX(w.OperationStartTime) AS LatestOperationTime
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationStartTime >= DATEADD(HOUR, -4, GETDATE())
      AND w.OperationEndTime IS NULL -- Still in progress
    GROUP BY g.LocationName, p.GravityZone
    HAVING COUNT(w.OperationKey) > 10
       AND AVG(w.DurationMinutes) > (@ThresholdMinutes * 0.8);
END
GO

-- Procedure for fleet triage scoring
CREATE OR ALTER PROCEDURE sp_FleetTriageScore
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH TripMetrics AS (
        SELECT 
            v.VehicleID,
            v.VehicleType,
            AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
            AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelEfficiency,
            COUNT(CASE WHEN ft.CompletionStatus = 'Delayed' THEN 1 END) AS DelayedTrips,
            DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceMaintenance
        FROM DimFleetVehicle v
        LEFT JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
        WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY v.VehicleID, v.VehicleType, v.LastMaintenanceDate
    )
    SELECT 
        VehicleID,
        VehicleType,
        AvgIdleTime,
        AvgFuelEfficiency,
        DelayedTrips,
        DaysSinceMaintenance,
        -- Weighted triage score (higher = more urgent)
        (AvgIdleTime * 0.3) + 
        (DelayedTrips * 2.0) + 
        (DaysSinceMaintenance * 0.1) +
        (CASE WHEN AvgFuelEfficiency > 0.15 THEN 20 ELSE 0 END) AS TriageScore
    FROM TripMetrics
    ORDER BY TriageScore DESC;
END
GO
```

### 3. Configure Power BI Connection

Create a `config.json` file (DO NOT commit to version control):

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogisticsDataWarehouse",
    "authentication": "Windows",
    "connection_string": "Server=${SQL_SERVER_HOST};Database=LogisticsDataWarehouse;Trusted_Connection=True;"
  },
  "power_bi": {
    "refresh_interval_minutes": 15,
    "gateway_cluster_id": "${POWERBI_GATEWAY_ID}"
  },
  "external_apis": {
    "weather_api_key": "${WEATHER_API_KEY}",
    "traffic_api_key": "${TRAFFIC_API_KEY}"
  }
}
```

### 4. Load Sample Data for Testing

```sql
-- Populate DimTime with 15-minute granularity for the past 3 months
DECLARE @StartDate DATETIME2 = DATEADD(MONTH, -3, GETDATE());
DECLARE @EndDate DATETIME2 = GETDATE();

;WITH TimeSlots AS (
    SELECT @StartDate AS SlotTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, SlotTime)
    FROM TimeSlots
    WHERE SlotTime < @EndDate
)
INSERT INTO DimTime (DateTime, DateKey, TimeSlot15Min, HourOfDay, DayOfWeek, DayOfMonth, MonthOfYear, Quarter, FiscalYear, IsWeekend)
SELECT 
    SlotTime,
    CONVERT(INT, FORMAT(SlotTime, 'yyyyMMdd')),
    ((DATEPART(HOUR, SlotTime) * 4) + (DATEPART(MINUTE, SlotTime) / 15)),
    DATEPART(HOUR, SlotTime),
    DATEPART(WEEKDAY, SlotTime),
    DATEPART(DAY, SlotTime),
    DATEPART(MONTH, SlotTime),
    DATEPART(QUARTER, SlotTime),
    CASE 
        WHEN DATEPART(MONTH, SlotTime) >= 4 THEN YEAR(SlotTime)
        ELSE YEAR(SlotTime) - 1
    END,
    CASE WHEN DATEPART(WEEKDAY, SlotTime) IN (1,7) THEN 1 ELSE 0 END
FROM TimeSlots
OPTION (MAXRECURSION 0);

-- Insert sample warehouse locations
INSERT INTO DimGeography (LocationCode, LocationName, LocationType, City, Region, Country, Continent, Latitude, Longitude)
VALUES 
    ('WH-NYC-01', 'New York Distribution Center', 'Warehouse', 'New York', 'NY', 'USA', 'North America', 40.7128, -74.0060),
    ('WH-CHI-01', 'Chicago Logistics Hub', 'Warehouse', 'Chicago', 'IL', 'USA', 'North America', 41.8781, -87.6298),
    ('WH-LA-01', 'Los Angeles Fulfillment', 'Warehouse', 'Los Angeles', 'CA', 'USA', 'North America', 34.0522, -118.2437);

-- Insert sample products
INSERT INTO DimProductGravity (SKU, ProductName, ProductCategory, UnitWeight, UnitVolume, UnitValue, FragilityScore)
VALUES 
    ('SKU-ELEC-001', 'Smartphone XR Pro', 'Electronics', 0.2, 0.001, 899.99, 9),
    ('SKU-FURN-001', 'Office Chair Deluxe', 'Furniture', 15.0, 0.5, 349.99, 4),
    ('SKU-FOOD-001', 'Organic Protein Bar Box', 'Food', 0.5, 0.002, 24.99, 2);

-- Insert sample fleet vehicles
INSERT INTO DimFleetVehicle (VehicleID, VehicleType, Manufacturer, Model, YearManufactured, MaxLoadCapacityKg, FuelType, LastMaintenanceDate)
VALUES 
    ('TRK-001', 'Truck', 'Freightliner', 'Cascadia', 2022, 15000, 'Diesel', '2025-12-15'),
    ('VAN-001', 'Van', 'Mercedes-Benz', 'Sprinter', 2023, 3500, 'Diesel', '2026-01-10');
```

## Key Queries & Patterns

### Cross-Fact KPI: Inventory Turnover vs. Fleet Efficiency

```sql
-- Link warehouse dwell time to fleet delivery performance
WITH WarehouseDwellMetrics AS (
    SELECT 
        p.ProductCategory,
        AVG(w.DwellTimeHours) AS AvgDwellTimeHours,
        COUNT(*) AS TotalOperations
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductCategory
),
FleetEfficiencyMetrics AS (
    SELECT 
        AVG(f.IdleTimeMinutes / NULLIF(f.TripDurationMinutes, 0) * 100) AS AvgIdlePercentage,
        AVG(f.FuelConsumedLiters / NULLIF(f.DistanceKm, 0)) AS AvgFuelPerKm
    FROM FactFleetTrips f
    WHERE f.TripStartTime >= DATEADD(DAY, -30, GETDATE())
)
SELECT 
    w.ProductCategory,
    w.AvgDwellTimeHours,
    f.AvgIdlePercentage,
    f.AvgFuelPerKm,
    -- Composite efficiency score
    (w.AvgDwellTimeHours * f.AvgIdlePercentage) AS CombinedInefficiencyIndex
FROM WarehouseDwellMetrics w
CROSS JOIN FleetEfficiencyMetrics f
ORDER BY CombinedInefficiencyIndex DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be reassigned to different gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone AS CurrentZone,
    COUNT(w.OperationKey) AS PicksLast90Days,
    AVG(w.DurationMinutes) AS AvgPickDuration,
    p.UnitValue,
    CASE 
        WHEN COUNT(w.OperationKey) >= 50 AND p.UnitValue > 100 THEN 'High'
        WHEN COUNT(w.OperationKey) >= 20 THEN 'Medium'
        ELSE 'Low'
    END AS RecommendedZone
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
WHERE w.OperationType = 'Picking'
  AND w.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityZone, p.UnitValue
HAVING CASE 
    WHEN COUNT(w.OperationKey) >= 50 AND p.UnitValue > 100 THEN 'High'
    WHEN COUNT(w.OperationKey) >= 20 THEN 'Medium'
    ELSE 'Low'
END <> p.GravityZone;
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouse zones likely to experience congestion in next 2 hours
WITH RecentActivity AS (
    SELECT 
        w.ZoneAssigned,
        COUNT(*) AS CurrentOperations,
        AVG(w.DurationMinutes) AS AvgDuration,
        STDEV(w.DurationMinutes) AS StdDevDuration
    FROM FactWarehouseOperations w
    WHERE w.OperationStartTime >= DATEADD(HOUR, -2, GETDATE())
      AND w.OperationEndTime IS NULL
    GROUP BY w.ZoneAssigned
),
HistoricalBaseline AS (
    SELECT 
        w.ZoneAssigned,
        AVG(COUNT(*)) OVER (PARTITION BY w.ZoneAssigned) AS HistoricalAvgOperations,
        AVG(AVG(w.DurationMinutes)) OVER (PARTITION BY w.ZoneAssigned) AS HistoricalAvgDuration
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
      AND t.HourOfDay = DATEPART(HOUR, GETDATE())
    GROUP BY w.ZoneAssigned, CAST(t.DateTime AS DATE)
)
SELECT 
    ra.ZoneAssigned,
    ra.CurrentOperations,
    hb.HistoricalAvgOperations,
    ra.AvgDuration,
    hb.HistoricalAvgDuration,
    -- Bottleneck probability score
    (ra.CurrentOperations / NULLIF(hb.HistoricalAvgOperations, 0)) * 
    (ra.AvgDuration / NULLIF(hb.HistoricalAvgDuration, 0)) AS BottleneckRisk
FROM RecentActivity ra
LEFT JOIN HistoricalBaseline hb ON ra.ZoneAssigned = hb.ZoneAssigned
WHERE (ra.CurrentOperations / NULLIF(hb.HistoricalAvgOperations, 0)) > 1.5
   OR (ra.AvgDuration / NULLIF(hb.HistoricalAvgDuration, 0)) > 1.3
ORDER BY BottleneckRisk DESC;
```

### Fleet Maintenance Priority Queue

```sql
-- Generate maintenance priority list based on trip metrics and vehicle age
EXEC sp_FleetTriageScore;
```

## Power BI Dashboard Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Measure: Warehouse-to-Fleet Efficiency Ratio
WarehouseFleetEfficiency = 
VAR AvgWarehouseDuration = 
    AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR AvgFleetIdle = 
    AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(AvgWarehouseDuration, AvgFleetIdle, 0)

// Measure: Gravity Zone Compliance %
GravityZoneCompliance = 
VAR CorrectZones = 
    CALCULATE(
        COUNTROWS(DimProductGravity),
        DimProductGravity[GravityZone] = DimProductGravity[RecommendedZone]
    )
VAR TotalProducts = COUNTROWS(DimProductGravity)
RETURN
    DIVIDE(CorrectZones, TotalProducts, 0)

// Measure: Predictive Bottleneck Index
BottleneckIndex = 
VAR CurrentLoad = COUNTROWS(FactWarehouseOperations)
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[OperationKey]),
        DATESINPERIOD(DimTime[DateTime], MAX(DimTime[DateTime]), -30, DAY)
    )
RETURN
    DIVIDE(CurrentLoad, HistoricalAvg, 1) * 100

// Measure: Fleet Carbon Footprint (kg CO2)
FleetCarbonFootprint = 
SUMX(
    FactFleetTrips,
    FactFleetTrips[FuelConsumedLiters] * 
    SWITCH(
        RELATED(DimFleetVehicle[FuelType]),
        "Diesel", 2.68,
        "Gasoline", 2.31,
        "Electric", 0,
        2.5
    )
)
```

### Power BI Template Setup

1. Open `LogiFleet_Pulse_Master.pbit`
2. Enter SQL Server connection details when prompted
3. Verify relationships in Model view:
   - `FactWarehouseOperations[TimeKey]` → `DimTime[TimeKey]`
   - `FactFleetTrips[VehicleKey]` → `DimFleetVehicle[VehicleKey]`
   - `FactCrossDock[InboundTripKey]` → `FactFleetTrips[TripKey]`
4. Configure row-level security:
   - Create role "WarehouseManager" with filter: `DimGeography[LocationType] = "Warehouse"`
   - Create role "FleetManager" with filter: `DimGeography[LocationType] = "Route Node"`

## Common Patterns

### Pattern 1: Time-Phased Dimension Joins

```sql
-- Join fact tables via time dimension for temporal correlation
SELECT 
    t.DateTime,
    w.OperationType,
    COUNT(DISTINCT w.OperationKey) AS WarehouseOps,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdle
FROM DimTime t
LEFT JOIN FactWarehouseOperations w ON t.TimeKey = w.TimeKey
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY t.DateTime, w.OperationType
ORDER BY t.DateTime;
```

### Pattern 2: Bridge Table for Many-to-Many

```sql
-- Create bridge table to link routes with multiple warehouses
CREATE TABLE BridgeRouteWarehouse (
    RouteID NVARCHAR(50) NOT NULL,
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    StopSequence TINYINT NOT NULL,
    PRIMARY KEY (RouteID, GeographyKey)
);

-- Query using bridge
SELECT 
    f.RouteID,
    g.LocationName AS WarehouseStop,
    br.StopSequence,
    SUM(f.IdleTimeMinutes) AS TotalIdleAtStop
FROM FactFleetTrips f
INNER JOIN BridgeRouteWarehouse br ON f.RouteID = br.RouteID
INNER JOIN DimGeography g ON br.GeographyKey = g.GeographyKey
GROUP BY f.RouteID, g.LocationName, br.StopSequence
ORDER BY f.RouteID, br.StopSequence;
```

### Pattern 3: Slowly Changing Dimension (Type 2)

```sql
-- Add SCD Type 2 columns to DimProductGravity
ALTER TABLE DimProductGravity ADD 
    EffectiveStartDate DATETIME2 DEFAULT GETDATE(),
    EffectiveEndDate DATETIME2 DEFAULT '9999-12-31',
    IsCurrent BIT DEFAULT 1;

-- Update procedure for gravity zone changes
CREATE OR ALTER PROCEDURE sp_UpdateProductGravityWithHistory
    @ProductKey INT,
    @NewGravityZone NVARCHAR(20)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Expire current record
    UPDATE DimProductGravity
    SET EffectiveEndDate = GETDATE(),
        IsCurrent = 0
    WHERE ProductKey = @ProductKey
      AND IsCurrent = 1;
    
    -- Insert new record
    INSERT INTO DimProductGravity (SKU, ProductName, ProductCategory, GravityZone, EffectiveStartDate, IsCurrent)
    SELECT SKU, ProductName, ProductCategory, @NewGravityZone, GETDATE(), 1
    FROM DimProductGravity
    WHERE ProductKey = @ProductKey
      AND EffectiveEndDate = GETDATE();
END
GO
```

## Troubleshooting

### Issue: Power BI Refresh Fails with Timeout

**Solution**: Implement incremental refresh policy

```sql
-- Create partitioned view for incremental loading
CREATE VIEW vw_FactWarehouseOperations_Incremental
AS
SELECT *
FROM FactWarehouseOperations
WHERE OperationStartTime >= DATEADD(DAY, -7, GETDATE());
```

In Power BI:
