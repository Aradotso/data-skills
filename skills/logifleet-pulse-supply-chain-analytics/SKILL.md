---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehousing with multi-fact star schema for fleet, warehouse, and supply chain KPI harmonization
triggers:
  - "set up LogiFleet Pulse analytics"
  - "deploy supply chain data warehouse"
  - "configure Power BI logistics dashboard"
  - "create multi-fact star schema for logistics"
  - "integrate warehouse and fleet telemetry data"
  - "build cross-modal supply chain analytics"
  - "implement logistics KPI harmonization"
  - "query warehouse gravity zones"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing solution for logistics and supply chain analytics. It provides a multi-fact star schema architecture that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for cross-modal KPI analysis.

## What It Does

- **Multi-Fact Data Warehouse**: Combines warehouse operations, fleet trips, cross-dock activities into a unified star schema
- **Time-Phased Dimensions**: 15-minute granularity time buckets with role-playing date dimensions
- **Cross-Fact KPI Harmonization**: Link inventory turnover with fuel burn rates, dwell time with route efficiency
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and replenishment lead time
- **Power BI Integration**: Pre-built dashboards with role-based access and real-time refresh
- **Predictive Analytics**: Bottleneck detection, fleet triage scoring, temporal elasticity modeling

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Developer, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SSMS (SQL Server Management Studio)
- Access to source systems: WMS, TMS, telemetry APIs

### Step 1: Deploy SQL Schema

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    IsBusinessHour BIT NOT NULL,
    FiscalPeriod VARCHAR(10),
    INDEX IX_DateTime (FullDateTime),
    INDEX IX_DateKey (DateKey)
);

-- Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode VARCHAR(20) NOT NULL UNIQUE,
    LocationName NVARCHAR(100) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- Warehouse, Route Node, Distribution Center
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1,
    INDEX IX_LocationCode (LocationCode),
    INDEX IX_Type_Active (LocationType, IsActive)
);

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency metric
    ValueScore DECIMAL(5,2), -- Unit value metric
    FragilityScore DECIMAL(5,2), -- Handling complexity
    GravityZone VARCHAR(10), -- A, B, C, D (A = highest gravity)
    RecommendedStorageZone VARCHAR(50),
    LastGravityCalculation DATETIME2,
    INDEX IX_SKU (SKU),
    INDEX IX_GravityZone (GravityZone)
);

-- Create supplier dimension
CREATE TABLE DimSupplier (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierCode VARCHAR(20) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    LeadTimeDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier VARCHAR(10), -- Platinum, Gold, Silver, Bronze
    INDEX IX_Code (SupplierCode)
);

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID VARCHAR(50),
    Quantity INT NOT NULL,
    DurationMinutes DECIMAL(8,2),
    DwellHours DECIMAL(10,2),
    ZoneAssigned VARCHAR(50),
    WorkerID VARCHAR(20),
    EquipmentID VARCHAR(20),
    CompletedTimestamp DATETIME2,
    INDEX IX_Time (TimeKey),
    INDEX IX_Geography (GeographyKey),
    INDEX IX_Product (ProductKey),
    INDEX IX_OpType (OperationType),
    INDEX IX_Timestamp (CompletedTimestamp),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(20) NOT NULL,
    DriverID VARCHAR(20),
    RouteID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleMinutes DECIMAL(8,2),
    LoadingMinutes DECIMAL(8,2),
    UnloadingMinutes DECIMAL(8,2),
    TotalTripMinutes DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    AvgSpeedKMH DECIMAL(6,2),
    DelayMinutes DECIMAL(8,2),
    DelayReason VARCHAR(100),
    CompletedTimestamp DATETIME2,
    INDEX IX_Time (TimeKey),
    INDEX IX_Origin (OriginGeographyKey),
    INDEX IX_Destination (DestinationGeographyKey),
    INDEX IX_Vehicle (VehicleID),
    INDEX IX_Timestamp (CompletedTimestamp),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    Quantity INT NOT NULL,
    DockDwellMinutes DECIMAL(8,2),
    TransferType VARCHAR(20), -- Direct, Staged, Quality Hold
    CompletedTimestamp DATETIME2,
    INDEX IX_Time (TimeKey),
    INDEX IX_Geography (GeographyKey),
    INDEX IX_Product (ProductKey),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate time dimension
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @EndDateTime DATETIME2 = DATEADD(DAY, 1, @EndDate);
    
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        DECLARE @TimeKey INT = 
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMdd') AS INT) * 10000 +
            DATEPART(HOUR, @CurrentDateTime) * 100 +
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15;
        
        INSERT INTO DimTime (
            TimeKey, FullDateTime, DateKey, TimeOfDay, 
            Hour, MinuteBucket, DayOfWeek, IsBusinessHour, FiscalPeriod
        )
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMdd') AS INT),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            DATEPART(WEEKDAY, @CurrentDateTime),
            CASE 
                WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
                     AND DATEPART(WEEKDAY, @CurrentDateTime) BETWEEN 2 AND 6
                THEN 1 ELSE 0 
            END,
            CONCAT('FY', YEAR(@CurrentDateTime), 'Q', DATEPART(QUARTER, @CurrentDateTime))
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate
EXEC PopulateTimeDimension '2025-01-01', '2027-12-31';
```

### Step 3: Create Views for Cross-Fact Analysis

```sql
-- View: Unified logistics performance
CREATE VIEW vw_UnifiedLogisticsPerformance AS
SELECT 
    t.FullDateTime,
    t.FiscalPeriod,
    g.LocationName,
    g.Region,
    g.Country,
    p.SKU,
    p.ProductName,
    p.GravityZone,
    
    -- Warehouse metrics
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.Quantity ELSE 0 END) AS TotalPickedUnits,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.DurationMinutes ELSE NULL END) AS AvgPickTimeMinutes,
    AVG(wo.DwellHours) AS AvgDwellHours,
    
    -- Fleet metrics (joined via geography)
    COUNT(DISTINCT ft.VehicleID) AS UniqueVehicles,
    SUM(ft.DistanceKM) AS TotalDistanceKM,
    SUM(ft.FuelConsumedLiters) AS TotalFuelLiters,
    AVG(ft.IdleMinutes) AS AvgIdleMinutes,
    SUM(ft.DelayMinutes) AS TotalDelayMinutes
    
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey 
    AND (g.GeographyKey = ft.OriginGeographyKey OR g.GeographyKey = ft.DestinationGeographyKey)

GROUP BY 
    t.FullDateTime, t.FiscalPeriod, g.LocationName, g.Region, g.Country,
    p.SKU, p.ProductName, p.GravityZone;
GO

-- View: Gravity zone optimization recommendations
CREATE VIEW vw_GravityZoneOptimization AS
SELECT 
    p.SKU,
    p.ProductName,
    p.CurrentGravityZone = p.GravityZone,
    p.RecommendedStorageZone,
    RecentPickFrequency = COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0),
    AvgDwellHours = AVG(wo.DwellHours),
    TotalPickTime = SUM(wo.DurationMinutes),
    SuggestedGravityZone = CASE
        WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 10 THEN 'A'
        WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 5 THEN 'B'
        WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 2 THEN 'C'
        ELSE 'D'
    END,
    NeedsRezone = CASE 
        WHEN p.GravityZone <> CASE
            WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 10 THEN 'A'
            WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 5 THEN 'B'
            WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 2 THEN 'C'
            ELSE 'D'
        END THEN 1 ELSE 0
    END
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    AND wo.OperationType = 'Picking'
LEFT JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityZone, p.RecommendedStorageZone;
GO
```

### Step 4: Create Stored Procedures for Data Loading

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assumes staging table exists: stg_WarehouseOperations
    -- with columns matching source WMS system
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OrderID, Quantity, DurationMinutes, DwellHours,
        ZoneAssigned, WorkerID, EquipmentID, CompletedTimestamp
    )
    SELECT 
        TimeKey = (CAST(FORMAT(s.CompletedTimestamp, 'yyyyMMdd') AS INT) * 10000 +
                   DATEPART(HOUR, s.CompletedTimestamp) * 100 +
                   (DATEPART(MINUTE, s.CompletedTimestamp) / 15) * 15),
        GeographyKey = g.GeographyKey,
        ProductKey = p.ProductKey,
        s.OperationType,
        s.OrderID,
        s.Quantity,
        s.DurationMinutes,
        s.DwellHours,
        s.ZoneAssigned,
        s.WorkerID,
        s.EquipmentID,
        s.CompletedTimestamp
    FROM stg_WarehouseOperations s
    INNER JOIN DimGeography g ON s.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.CompletedTimestamp > @LastLoadTimestamp
    AND NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.OrderID = s.OrderID 
        AND f.OperationType = s.OperationType
        AND f.CompletedTimestamp = s.CompletedTimestamp
    );
    
    SELECT @@ROWCOUNT AS RowsInserted;
END;
GO

-- Incremental load for fleet trips
CREATE PROCEDURE LoadFleetTrips
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, OriginGeographyKey, DestinationGeographyKey,
        VehicleID, DriverID, RouteID, DistanceKM, FuelConsumedLiters,
        IdleMinutes, LoadingMinutes, UnloadingMinutes, TotalTripMinutes,
        LoadWeightKG, AvgSpeedKMH, DelayMinutes, DelayReason, CompletedTimestamp
    )
    SELECT 
        TimeKey = (CAST(FORMAT(s.CompletedTimestamp, 'yyyyMMdd') AS INT) * 10000 +
                   DATEPART(HOUR, s.CompletedTimestamp) * 100 +
                   (DATEPART(MINUTE, s.CompletedTimestamp) / 15) * 15),
        OriginGeographyKey = go.GeographyKey,
        DestinationGeographyKey = gd.GeographyKey,
        s.VehicleID,
        s.DriverID,
        s.RouteID,
        s.DistanceKM,
        s.FuelConsumedLiters,
        s.IdleMinutes,
        s.LoadingMinutes,
        s.UnloadingMinutes,
        s.TotalTripMinutes,
        s.LoadWeightKG,
        s.AvgSpeedKMH,
        s.DelayMinutes,
        s.DelayReason,
        s.CompletedTimestamp
    FROM stg_FleetTrips s
    INNER JOIN DimGeography go ON s.OriginLocationCode = go.LocationCode
    INNER JOIN DimGeography gd ON s.DestinationLocationCode = gd.LocationCode
    WHERE s.CompletedTimestamp > @LastLoadTimestamp;
    
    SELECT @@ROWCOUNT AS RowsInserted;
END;
GO
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-sql-server.database.windows.net` (or local instance)
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)

### DAX Measures for Cross-Fact KPIs

```dax
// Total Operations Count
Total Operations = COUNTROWS(FactWarehouseOperations)

// Avg Dwell Time
Avg Dwell Time (Hours) = 
AVERAGE(FactWarehouseOperations[DwellHours])

// Fleet Efficiency Score
Fleet Efficiency Score = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
VAR TotalIdle = SUM(FactFleetTrips[IdleMinutes])
VAR TotalTrip = SUM(FactFleetTrips[TotalTripMinutes])
VAR IdlePercent = DIVIDE(TotalIdle, TotalTrip, 0)
RETURN
    (1 - IdlePercent) * 100

// Cross-Fact: Dwell Impact on Delivery Delay
Dwell-to-Delay Correlation = 
VAR AvgDwell = CALCULATE(AVERAGE(FactWarehouseOperations[DwellHours]))
VAR AvgDelay = CALCULATE(AVERAGE(FactFleetTrips[DelayMinutes]))
RETURN
    IF(AvgDwell > 48, AvgDelay * 1.2, AvgDelay) // 20% penalty if dwell > 48hrs

// Gravity Zone Performance Index
Gravity Zone Index = 
VAR ZoneAPickTime = CALCULATE(
    AVERAGE(FactWarehouseOperations[DurationMinutes]),
    DimProductGravity[GravityZone] = "A"
)
VAR ZoneDPickTime = CALCULATE(
    AVERAGE(FactWarehouseOperations[DurationMinutes]),
    DimProductGravity[GravityZone] = "D"
)
RETURN
    DIVIDE(ZoneDPickTime, ZoneAPickTime, 1) // Ideal ratio > 2

// Predictive Bottleneck Index
Bottleneck Risk Score = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellHours])
VAR HistoricalAvg = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellHours]),
    DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -30, DAY)
)
VAR CurrentIdle = AVERAGE(FactFleetTrips[IdleMinutes])
VAR HistoricalIdleAvg = CALCULATE(
    AVERAGE(FactFleetTrips[IdleMinutes]),
    DATESINPERIOD(DimTime[FullDateTime], MAX(DimTime[FullDateTime]), -30, DAY)
)
RETURN
    ((CurrentDwell / HistoricalAvg) * 0.6) + ((CurrentIdle / HistoricalIdleAvg) * 0.4)
```

### Row-Level Security (RLS)

```dax
// Create role: Regional Manager
[Region] = USERNAME() 
// Assumes user email domain matches region: "EMEA\john.doe" → filter Region = "EMEA"

// Create role: Warehouse Supervisor
DimGeography[LocationCode] IN { 
    "WH-001", "WH-002" 
} // Limit to specific warehouses
```

Apply roles in Power BI Desktop → Modeling → Manage Roles → Create → Define filter.

## Key Queries & Analysis Patterns

### Pattern 1: Cross-Fact Analysis (Dwell Time vs Fleet Utilization)

```sql
-- Find SKUs with high warehouse dwell time that correlate with delivery delays
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone,
    AvgDwellHours = AVG(wo.DwellHours),
    TotalShipments = COUNT(DISTINCT wo.OrderID),
    
    -- Correlated fleet data
    AvgDeliveryDelayMinutes = AVG(ft.DelayMinutes),
    TotalFleetTrips = COUNT(DISTINCT ft.TripKey),
    AvgIdleTime = AVG(ft.IdleMinutes)
    
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN DimTime t1 ON wo.TimeKey = t1.TimeKey
LEFT JOIN FactFleetTrips ft ON wo.GeographyKey = ft.OriginGeographyKey
    AND CAST(t1.FullDateTime AS DATE) = CAST(t2.FullDateTime AS DATE)
INNER JOIN DimTime t2 ON ft.TimeKey = t2.TimeKey

WHERE t1.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
AND wo.OperationType = 'Shipping'

GROUP BY p.SKU, p.ProductName, p.GravityZone
HAVING AVG(wo.DwellHours) > 48 -- High dwell threshold

ORDER BY AvgDeliveryDelayMinutes DESC;
```

### Pattern 2: Temporal Elasticity Simulation

```sql
-- Simulate impact of 95% warehouse capacity on fleet utilization
WITH WarehouseLoad AS (
    SELECT 
        t.DateKey,
        CurrentCapacityPercent = 
            (SUM(wo.Quantity) * 100.0) / 
            (SELECT SUM(Quantity) FROM FactWarehouseOperations WHERE OperationType = 'Receiving'),
        TotalOperations = COUNT(wo.OperationKey)
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.DateKey
),
FleetLoad AS (
    SELECT 
        t.DateKey,
        AvgTripDuration = AVG(ft.TotalTripMinutes),
        AvgIdlePercent = (SUM(ft.IdleMinutes) * 100.0) / SUM(ft.TotalTripMinutes)
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.DateKey
)
SELECT 
    w.DateKey,
    w.CurrentCapacityPercent,
    w.TotalOperations,
    f.AvgTripDuration,
    f.AvgIdlePercent,
    
    -- Simulation: at 95% capacity, predict 15% increase in idle time
    SimulatedIdlePercent = CASE 
        WHEN w.CurrentCapacityPercent < 95 
        THEN f.AvgIdlePercent * (1 + ((95 - w.CurrentCapacityPercent) * 0.015))
        ELSE f.AvgIdlePercent
    END,
    
    PredictedImpact = CASE 
        WHEN w.CurrentCapacityPercent < 95 
        THEN 'Capacity increase will likely raise fleet idle time'
        ELSE 'Already at/above threshold'
    END
    
FROM WarehouseLoad w
INNER JOIN FleetLoad f ON w.DateKey = f.DateKey

ORDER BY w.DateKey DESC;
```

### Pattern 3: Gravity Zone Optimization Query

```sql
-- Identify products that should be reassigned to different gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.CurrentGravityZone = p.GravityZone,
    
    -- Calculate velocity score from operations data
    PicksPerDay = COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0),
    AvgPickTime = AVG(wo.DurationMinutes),
    AvgDwell = AVG(wo.DwellHours),
    
    -- Recommend new zone
    RecommendedZone = CASE
        WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 10 THEN 'A'
        WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 5 THEN 'B'
        WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 2 THEN 'C'
        ELSE 'D'
    END,
    
    ZoneChangePriority = CASE
        WHEN ABS(
            ASCII(p.GravityZone) - 
            ASCII(CASE
                WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 10 THEN 'A'
                WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 5 THEN 'B'
                WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 2 THEN 'C'
                ELSE 'D'
            END)
        ) > 1 THEN 'HIGH'
        WHEN ABS(
            ASCII(p.GravityZone) - 
            ASCII(CASE
                WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 10 THEN 'A'
                WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 5 THEN 'B'
                WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 2 THEN 'C'
                ELSE 'D'
            END)
        ) = 1 THEN 'MEDIUM'
        ELSE 'LOW'
    END

FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    AND wo.OperationType = 'Picking'
LEFT JOIN DimTime t ON wo.TimeKey = t.TimeKey

WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())

GROUP BY p.SKU, p.ProductName, p.GravityZone

HAVING p.GravityZone <> CASE
    WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 10 THEN 'A'
    WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 5 THEN 'B'
    WHEN COUNT(wo.OperationKey) / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0) > 2 THEN 'C'
    ELSE 'D'
END

ORDER BY ZoneChangePriority DESC, PicksPerDay DESC;
```

## Configuration Files

### config
