---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse operations, fleet management, and cross-modal supply chain analytics
triggers:
  - set up LogiFleet Pulse supply chain dashboard
  - configure warehouse and fleet analytics
  - implement multi-fact star schema for logistics
  - create Power BI logistics intelligence report
  - build SQL data warehouse for supply chain
  - integrate warehouse operations with fleet telemetry
  - design logistics KPI dashboard
  - deploy real-time fleet optimization analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a cohesive MS SQL Server data warehouse with Power BI visualization layer. It implements a multi-fact star schema to enable cross-domain analytics like correlating warehouse dwell time with fleet idle costs, predicting bottlenecks, and optimizing storage zones based on product velocity.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Unified semantic layer across warehouse, fleet, and supplier data
- Real-time dashboards with 15-minute refresh cycles
- Predictive bottleneck detection and fleet triage
- Warehouse "gravity zone" optimization based on pick frequency and item value
- Cross-fact KPI harmonization (e.g., inventory turnover vs. fuel consumption)

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SSMS (SQL Server Management Studio) or Azure Data Studio
- Source systems: WMS, TMS, or telemetry APIs (optional for initial setup)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy the schema (run the provided SQL scripts in order)
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create relationships and indexes
-- 4. Create views and stored procedures
```

### Step 2: Create Core Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayOfWeekName NVARCHAR(20),
    FiscalPeriod INT,
    FiscalQuarter INT,
    FiscalYear INT,
    IsWeekend BIT DEFAULT 0,
    IsHoliday BIT DEFAULT 0,
    INDEX IX_DimTime_FullDateTime NONCLUSTERED (FullDateTime),
    INDEX IX_DimTime_DateKey NONCLUSTERED (DateKey)
);
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- Warehouse, Route Node, Distribution Center
    Address NVARCHAR(500),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,8),
    Longitude DECIMAL(11,8),
    TimeZone NVARCHAR(50),
    IsActive BIT DEFAULT 1,
    INDEX IX_DimGeography_LocationID NONCLUSTERED (LocationID),
    INDEX IX_DimGeography_LocationType NONCLUSTERED (LocationType)
);
GO

-- DimProduct: Product master with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(100) UNIQUE NOT NULL,
    ProductName NVARCHAR(300),
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    TemperatureZone NVARCHAR(20), -- Ambient, Refrigerated, Frozen
    GravityScore DECIMAL(5,2) DEFAULT 0, -- Calculated metric
    ABCClass CHAR(1), -- A = high velocity, B = medium, C = low
    StandardCost DECIMAL(12,2),
    ListPrice DECIMAL(12,2),
    IsActive BIT DEFAULT 1,
    INDEX IX_DimProduct_SKU NONCLUSTERED (SKU),
    INDEX IX_DimProduct_GravityScore NONCLUSTERED (GravityScore DESC)
);
GO

-- DimSupplier: Supplier reliability tracking
CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    Country NVARCHAR(100),
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2), -- Standard deviation
    DefectRate DECIMAL(5,4), -- Percentage
    ComplianceScore DECIMAL(5,2), -- 0-100 scale
    IsActive BIT DEFAULT 1,
    INDEX IX_DimSupplier_SupplierID NONCLUSTERED (SupplierID)
);
GO

-- DimFleet: Vehicle master data
CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) UNIQUE NOT NULL,
    VehicleType NVARCHAR(50), -- Truck, Van, Cargo Plane
    Make NVARCHAR(50),
    Model NVARCHAR(50),
    Year INT,
    Capacity DECIMAL(10,2), -- Weight capacity
    FuelType NVARCHAR(30),
    VIN NVARCHAR(100),
    MaintenanceStatus NVARCHAR(50),
    IsActive BIT DEFAULT 1,
    INDEX IX_DimFleet_VehicleID NONCLUSTERED (VehicleID)
);
GO
```

### Step 3: Create Fact Tables

```sql
-- FactWarehouseOperations: Warehouse activity tracking
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID NVARCHAR(100),
    BatchID NVARCHAR(100),
    QuantityHandled INT,
    DwellTimeMinutes INT, -- Time spent in warehouse before next operation
    PickRate DECIMAL(8,2), -- Units per hour
    PackTimeMinutes INT,
    ZoneID NVARCHAR(20), -- Storage zone identifier
    GravityZoneTier INT, -- 1 = high gravity, 5 = low gravity
    OperatorID NVARCHAR(50),
    EquipmentID NVARCHAR(50), -- Forklift, pallet jack, etc.
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_FactWarehouse_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey),
    INDEX IX_FactWarehouse_Time NONCLUSTERED (TimeKey),
    INDEX IX_FactWarehouse_Product NONCLUSTERED (ProductKey),
    INDEX IX_FactWarehouse_OperationType NONCLUSTERED (OperationType)
);
GO

-- FactFleetTrips: Fleet telemetry and trip details
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL, -- Trip start time
    FleetKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DriverID NVARCHAR(50),
    TripID NVARCHAR(100) UNIQUE NOT NULL,
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,3),
    AverageSpeed DECIMAL(6,2),
    MaxSpeed DECIMAL(6,2),
    LoadWeight DECIMAL(10,2),
    LoadVolume DECIMAL(10,2),
    WeatherCondition NVARCHAR(50), -- Clear, Rain, Snow, etc.
    TrafficDelay NVARCHAR(50), -- None, Minor, Moderate, Severe
    TireAlert BIT DEFAULT 0,
    EngineAlert BIT DEFAULT 0,
    MaintenanceScore DECIMAL(5,2), -- 0-100, lower = needs attention
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Fleet FOREIGN KEY (FleetKey) REFERENCES DimFleet(FleetKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactFleet_Time NONCLUSTERED (TimeKey),
    INDEX IX_FactFleet_Fleet NONCLUSTERED (FleetKey),
    INDEX IX_FactFleet_TripID NONCLUSTERED (TripID)
);
GO

-- FactCrossDock: Cross-docking operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID NVARCHAR(100),
    OutboundTripID NVARCHAR(100),
    QuantityTransferred INT,
    DockDwellMinutes INT, -- Time between inbound and outbound
    CONSTRAINT FK_FactCrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactCrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    INDEX IX_FactCrossDock_Time NONCLUSTERED (TimeKey)
);
GO
```

### Step 4: Create Calculated Gravity Score Stored Procedure

```sql
-- Update product gravity scores based on warehouse operations
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity score: weighted sum of pick frequency, value, and fragility
    UPDATE p
    SET GravityScore = COALESCE(
        (w.PickFrequency * 0.5) + 
        (w.ValueScore * 0.3) + 
        (CASE WHEN p.IsFragile = 1 THEN 20 ELSE 0 END * 0.2),
        0
    ),
    ABCClass = CASE 
        WHEN GravityScore >= 70 THEN 'A'
        WHEN GravityScore >= 40 THEN 'B'
        ELSE 'C'
    END
    FROM DimProduct p
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 100.0 / NULLIF((SELECT COUNT(*) FROM FactWarehouseOperations), 0) AS PickFrequency,
            (AVG(CAST(ProductKey AS FLOAT)) / (SELECT MAX(ProductKey) FROM DimProduct)) * 100 AS ValueScore
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(day, -90, GETDATE()))
        GROUP BY ProductKey
    ) w ON p.ProductKey = w.ProductKey;
    
END;
GO
```

### Step 5: Create Analytical Views

```sql
-- View: Cross-fact KPI combining warehouse dwell and fleet idle
CREATE VIEW vw_DwellToIdleCorrelation AS
SELECT 
    dt.FiscalYear,
    dt.FiscalQuarter,
    dg.Region,
    dp.ProductCategory,
    AVG(fw.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    AVG(ff.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    COUNT(DISTINCT fw.OrderID) AS OrderCount,
    SUM(fw.QuantityHandled) AS TotalUnitsHandled,
    SUM(ff.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(dp.GravityScore) AS AvgProductGravity
FROM FactWarehouseOperations fw
INNER JOIN DimTime dt ON fw.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fw.GeographyKey = dg.GeographyKey
INNER JOIN DimProduct dp ON fw.ProductKey = dp.ProductKey
LEFT JOIN FactFleetTrips ff ON fw.OrderID = ff.TripID 
    AND ABS(DATEDIFF(hour, dt.FullDateTime, (SELECT FullDateTime FROM DimTime WHERE TimeKey = ff.TimeKey))) <= 24
GROUP BY 
    dt.FiscalYear,
    dt.FiscalQuarter,
    dg.Region,
    dp.ProductCategory;
GO

-- View: Fleet maintenance priority queue
CREATE VIEW vw_FleetMaintenancePriority AS
SELECT TOP 100
    df.VehicleID,
    df.VehicleType,
    df.MaintenanceStatus,
    AVG(ff.MaintenanceScore) AS AvgMaintenanceScore,
    SUM(CASE WHEN ff.TireAlert = 1 THEN 1 ELSE 0 END) AS TireAlerts,
    SUM(CASE WHEN ff.EngineAlert = 1 THEN 1 ELSE 0 END) AS EngineAlerts,
    SUM(ff.DistanceKM) AS TotalDistanceKM,
    AVG(ff.IdleTimeMinutes) AS AvgIdleTime,
    -- Priority score: lower maintenance score + more alerts = higher priority
    (100 - AVG(ff.MaintenanceScore)) * 0.6 + 
    (SUM(CASE WHEN ff.TireAlert = 1 THEN 1 ELSE 0 END) * 5) * 0.2 +
    (SUM(CASE WHEN ff.EngineAlert = 1 THEN 1 ELSE 0 END) * 10) * 0.2 AS PriorityScore
FROM DimFleet df
INNER JOIN FactFleetTrips ff ON df.FleetKey = ff.FleetKey
WHERE df.IsActive = 1
    AND ff.TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(day, -30, GETDATE()))
GROUP BY 
    df.VehicleID,
    df.VehicleType,
    df.MaintenanceStatus
ORDER BY PriorityScore DESC;
GO
```

### Step 6: Data Loading Stored Procedure Template

```sql
-- Incremental load from staging table to fact table
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Use last load time or default to 24 hours ago
    DECLARE @LoadFromDate DATETIME2 = COALESCE(@LastLoadDateTime, DATEADD(hour, -24, GETDATE()));
    
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        GeographyKey,
        ProductKey,
        SupplierKey,
        OperationType,
        OrderID,
        BatchID,
        QuantityHandled,
        DwellTimeMinutes,
        PickRate,
        PackTimeMinutes,
        ZoneID,
        GravityZoneTier,
        OperatorID,
        EquipmentID
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        ds.SupplierKey,
        stg.OperationType,
        stg.OrderID,
        stg.BatchID,
        stg.QuantityHandled,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        CASE WHEN DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) > 0 
             THEN stg.QuantityHandled * 60.0 / DATEDIFF(MINUTE, stg.StartTime, stg.EndTime)
             ELSE 0 END AS PickRate,
        stg.PackTimeMinutes,
        stg.ZoneID,
        stg.GravityZoneTier,
        stg.OperatorID,
        stg.EquipmentID
    FROM Staging_WarehouseOperations stg
    INNER JOIN DimTime dt ON DATEADD(MINUTE, (DATEPART(MINUTE, stg.StartTime) / 15) * 15 - DATEPART(MINUTE, stg.StartTime), stg.StartTime) = dt.FullDateTime
    INNER JOIN DimGeography dg ON stg.LocationID = dg.LocationID
    INNER JOIN DimProduct dp ON stg.SKU = dp.SKU
    LEFT JOIN DimSupplier ds ON stg.SupplierID = ds.SupplierID
    WHERE stg.StartTime >= @LoadFromDate
        AND stg.IsProcessed = 0;
    
    -- Mark as processed
    UPDATE Staging_WarehouseOperations
    SET IsProcessed = 1, ProcessedDateTime = GETDATE()
    WHERE StartTime >= @LoadFromDate AND IsProcessed = 0;
    
    -- Update gravity scores after load
    EXEC usp_UpdateProductGravityScores;
    
END;
GO
```

## Power BI Configuration

### Step 1: Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server name and database: `LogiFleetPulse`
4. Connection mode: **Import** (for smaller datasets) or **DirectQuery** (for real-time, large datasets)
5. Load tables: `FactWarehouseOperations`, `FactFleetTrips`, `FactCrossDock`, and all `Dim*` tables
6. Load views: `vw_DwellToIdleCorrelation`, `vw_FleetMaintenancePriority`

### Step 2: Configure Relationships in Power BI

Power BI should auto-detect most relationships. Verify:

```
FactWarehouseOperations[TimeKey] → DimTime[TimeKey] (Many-to-One)
FactWarehouseOperations[GeographyKey] → DimGeography[GeographyKey] (Many-to-One)
FactWarehouseOperations[ProductKey] → DimProduct[ProductKey] (Many-to-One)
FactWarehouseOperations[SupplierKey] → DimSupplier[SupplierKey] (Many-to-One)

FactFleetTrips[TimeKey] → DimTime[TimeKey] (Many-to-One)
FactFleetTrips[FleetKey] → DimFleet[FleetKey] (Many-to-One)
FactFleetTrips[OriginGeographyKey] → DimGeography[GeographyKey] (Many-to-One, inactive)
FactFleetTrips[DestinationGeographyKey] → DimGeography[GeographyKey] (Many-to-One, active)

FactCrossDock[TimeKey] → DimTime[TimeKey] (Many-to-One)
FactCrossDock[GeographyKey] → DimGeography[GeographyKey] (Many-to-One)
FactCrossDock[ProductKey] → DimProduct[ProductKey] (Many-to-One)
```

### Step 3: Create DAX Measures

```dax
// Total Warehouse Dwell Time (Hours)
Total Dwell Hours = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Average Fleet Idle Percentage
Avg Idle Pct = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Fuel Efficiency (KM per Liter)
Fuel Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters]),
    0
)

// Cross-Dock Velocity (Units per Hour)
CrossDock Velocity = 
DIVIDE(
    SUM(FactCrossDock[QuantityTransferred]),
    SUMX(FactCrossDock, FactCrossDock[DockDwellMinutes] / 60),
    0
)

// High Priority Maintenance Count
High Priority Vehicles = 
CALCULATE(
    DISTINCTCOUNT(FactFleetTrips[FleetKey]),
    FactFleetTrips[MaintenanceScore] < 40
)

// Bottleneck Index (composite score)
Bottleneck Index = 
VAR HighDwellItems = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[DwellTimeMinutes] > 120
    )
VAR HighIdleTrips = 
    CALCULATE(
        COUNT(FactFleetTrips[TripKey]),
        DIVIDE(FactFleetTrips[IdleTimeMinutes], FactFleetTrips[DurationMinutes], 0) > 0.2
    )
VAR TotalOps = COUNT(FactWarehouseOperations[OperationKey])
VAR TotalTrips = COUNT(FactFleetTrips[TripKey])
RETURN
    (HighDwellItems * 1.0 / NULLIF(TotalOps, 0) * 50) + 
    (HighIdleTrips * 1.0 / NULLIF(TotalTrips, 0) * 50)

// Gravity Zone Efficiency
Gravity Efficiency = 
AVERAGEX(
    SUMMARIZE(
        FactWarehouseOperations,
        DimProduct[ABCClass],
        FactWarehouseOperations[GravityZoneTier]
    ),
    IF(
        DimProduct[ABCClass] = "A" && FactWarehouseOperations[GravityZoneTier] <= 2, 100,
        IF(DimProduct[ABCClass] = "B" && FactWarehouseOperations[GravityZoneTier] <= 3, 75,
        IF(DimProduct[ABCClass] = "C" && FactWarehouseOperations[GravityZoneTier] >= 4, 50, 25)
        )
    )
)
```

### Step 4: Configure Row-Level Security (RLS)

```dax
// In Power BI, create roles under Modeling → Manage roles

// Role: Regional Manager (sees only their region)
[Region] = USERPRINCIPALNAME()

// Role: Warehouse Supervisor (sees only their warehouse)
[LocationID] IN { 
    LOOKUPVALUE(
        'UserWarehouseAccess'[LocationID],
        'UserWarehouseAccess'[UserEmail], USERPRINCIPALNAME()
    )
}

// Role: Executive (sees all data, no filter)
1 = 1
```

Create a bridge table `UserWarehouseAccess` in SQL:

```sql
CREATE TABLE UserWarehouseAccess (
    UserEmail NVARCHAR(200),
    LocationID NVARCHAR(50),
    AccessLevel NVARCHAR(50)
);

INSERT INTO UserWarehouseAccess VALUES 
('supervisor1@company.com', 'WH-001', 'Supervisor'),
('supervisor2@company.com', 'WH-002', 'Supervisor'),
('regional.mgr@company.com', 'WH-001', 'Regional'),
('regional.mgr@company.com', 'WH-002', 'Regional');
```

## Key Usage Patterns

### Pattern 1: Daily Bottleneck Detection

```sql
-- Query to identify potential bottlenecks in the last 24 hours
SELECT 
    dg.LocationName,
    dp.ProductCategory,
    COUNT(*) AS OperationCount,
    AVG(fw.DwellTimeMinutes) AS AvgDwellMinutes,
    MAX(fw.DwellTimeMinutes) AS MaxDwellMinutes,
    COUNT(CASE WHEN fw.DwellTimeMinutes > 120 THEN 1 END) AS HighDwellCount,
    AVG(dp.GravityScore) AS AvgGravityScore
FROM FactWarehouseOperations fw
INNER JOIN DimTime dt ON fw.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fw.GeographyKey = dg.GeographyKey
INNER JOIN DimProduct dp ON fw.ProductKey = dp.ProductKey
WHERE dt.FullDateTime >= DATEADD(hour, -24, GETDATE())
GROUP BY dg.LocationName, dp.ProductCategory
HAVING AVG(fw.DwellTimeMinutes) > 90
ORDER BY AvgDwellMinutes DESC;
```

### Pattern 2: Fleet Fuel Cost Analysis

```sql
-- Calculate fuel cost per route, assuming fuel price
DECLARE @FuelPricePerLiter DECIMAL(6,3) = 1.45; -- Update with current price

SELECT 
    origin.LocationName AS Origin,
    dest.LocationName AS Destination,
    COUNT(*) AS TripCount,
    AVG(ff.DistanceKM) AS AvgDistanceKM,
    SUM(ff.FuelConsumedLiters) AS TotalFuelLiters,
    SUM(ff.FuelConsumedLiters) * @FuelPricePerLiter AS TotalFuelCost,
    AVG(ff.IdleTimeMinutes) AS AvgIdleMinutes,
    AVG(ff.DistanceKM / NULLIF(ff.FuelConsumedLiters, 0)) AS AvgKmPerLiter
FROM FactFleetTrips ff
INNER JOIN DimTime dt ON ff.TimeKey = dt.TimeKey
INNER JOIN DimGeography origin ON ff.OriginGeographyKey = origin.GeographyKey
INNER JOIN DimGeography dest ON ff.DestinationGeographyKey = dest.GeographyKey
WHERE dt.FiscalQuarter = DATEPART(quarter, GETDATE())
    AND dt.FiscalYear = YEAR(GETDATE())
GROUP BY origin.LocationName, dest.LocationName
ORDER BY TotalFuelCost DESC;
```

### Pattern 3: ABC Class Reallocation Recommendation

```sql
-- Find products in wrong gravity zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.ABCClass,
    AVG(fw.GravityZoneTier) AS CurrentAvgZone,
    CASE dp.ABCClass
        WHEN 'A' THEN 1.5
        WHEN 'B' THEN 3.0
        WHEN 'C' THEN 4.5
    END AS RecommendedZone,
    COUNT(*) AS OperationCount,
    AVG(fw.DwellTimeMinutes) AS AvgDwellMinutes
FROM FactWarehouseOperations fw
INNER JOIN DimProduct dp ON fw.ProductKey = dp.ProductKey
INNER JOIN DimTime dt ON fw.TimeKey = dt.TimeKey
WHERE dt.FullDateTime >= DATEADD(day, -30, GETDATE())
    AND fw.OperationType IN ('Picking', 'Packing')
GROUP BY dp.SKU, dp.ProductName, dp.ABCClass
HAVING ABS(AVG(fw.GravityZoneTier) - 
    CASE dp.ABCClass
        WHEN 'A' THEN 1.5
        WHEN 'B' THEN 3.0
        WHEN 'C' THEN 4.5
    END) > 1.0
ORDER BY OperationCount DESC;
```

### Pattern 4: Automated Alerting

```sql
-- Create alert stored procedure
CREATE PROCEDURE usp_CheckKPIThresholdsAndAlert
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertLog TABLE (
        AlertType NVARCHAR(100),
        Severity NVARCHAR(20),
        Message NVARCHAR(500),
        Value DECIMAL(12,2)
    );
    
    -- Check 1: High fleet idle time
    INSERT INTO @AlertLog
    SELECT 
        'Fleet Idle Time',
        'High',
        'Vehicle ' + df.VehicleID + ' has exceeded 20% idle time',
        AVG(CAST(ff.IdleTimeMinutes AS DECIMAL) / NULLIF(ff.DurationMinutes, 0) * 100)
    FROM FactFleetTrips ff
    INNER JOIN DimFleet df ON ff.FleetKey = df.FleetKey
    INNER JOIN DimTime dt ON ff.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(hour, -24, GETDATE())
    GROUP BY df.VehicleID
    HAVING AVG(CAST(ff.IdleTimeMinutes AS DECIMAL) / NULLIF(ff.DurationMinutes, 0)) > 0.20;
    
    -- Check 2: High warehouse dwell time
    INSERT INTO @AlertLog
    SELECT 
        'Warehouse Dwell Time',
        'Critical',
        'Location ' + dg.LocationName + ' has items with >3 hour dwell',
        AVG(fw.DwellTimeMinutes)
    FROM FactWarehouseOperations fw
    INNER JOIN DimGeography dg ON fw.GeographyKey = dg.GeographyKey
    INNER JOIN DimTime dt ON fw.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(hour, -24, GETDATE())
        AND fw.DwellTimeMinutes > 
