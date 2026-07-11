---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logistics analytics dashboard
  - configure supply chain data warehouse
  - implement fleet tracking analytics
  - build warehouse operations reporting
  - create power bi logistics dashboard
  - design multi-fact star schema for supply chain
  - integrate warehouse and fleet data models
  - deploy logifleet pulse analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics solution that unifies warehouse operations, fleet telemetry, and supply chain data into a single semantic layer. Built on MS SQL Server with Power BI visualizations, it uses a multi-fact star schema to enable cross-domain analytics like correlating warehouse dwell times with fleet idle costs, or inventory velocity with delivery route optimization.

**Key capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time dashboards with 15-minute refresh cycles
- Predictive bottleneck detection and fleet triage
- Warehouse gravity zone optimization
- Role-based access control with row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs

### Step 1: Deploy SQL Schema

```sql
-- 1. Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    QuarterOfYear TINYINT NOT NULL,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0,
    INDEX IX_DimTime_DateKey NONCLUSTERED (DateKey),
    INDEX IX_DimTime_HourOfDay NONCLUSTERED (HourOfDay)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    PostalCode VARCHAR(20),
    Country VARCHAR(100) NOT NULL,
    Region VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    INDEX IX_DimGeography_LocationID NONCLUSTERED (LocationID),
    INDEX IX_DimGeography_Country NONCLUSTERED (Country)
);

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(300) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    Weight DECIMAL(10,4),
    Volume DECIMAL(10,4),
    IsPerishable BIT DEFAULT 0,
    IsFragile BIT DEFAULT 0,
    UnitCost DECIMAL(18,4),
    UnitPrice DECIMAL(18,4),
    GravityScore DECIMAL(5,2) DEFAULT 50.0, -- Warehouse gravity zone metric
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow mover
    INDEX IX_DimProduct_SKU NONCLUSTERED (SKU),
    INDEX IX_DimProduct_Category NONCLUSTERED (Category),
    INDEX IX_DimProduct_GravityScore NONCLUSTERED (GravityScore)
);

CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) UNIQUE NOT NULL,
    VehicleType VARCHAR(50) NOT NULL, -- Truck, Van, Trailer
    Make VARCHAR(50),
    Model VARCHAR(50),
    Year INT,
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(30),
    CurrentMileage INT,
    LastMaintenanceDate DATE,
    Status VARCHAR(20) DEFAULT 'Active', -- Active, Maintenance, Retired
    INDEX IX_DimVehicle_VehicleID NONCLUSTERED (VehicleID),
    INDEX IX_DimVehicle_Status NONCLUSTERED (Status)
);

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200) NOT NULL,
    Country VARCHAR(100),
    ReliabilityScore DECIMAL(5,2), -- 0-100 score
    AvgLeadTimeDays INT,
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    INDEX IX_DimSupplier_SupplierID NONCLUSTERED (SupplierID)
);

-- 3. Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NULL,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100) UNIQUE NOT NULL,
    OrderID VARCHAR(100),
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    PickPathLength DECIMAL(10,2),
    StorageZone VARCHAR(50),
    HandlingCost DECIMAL(18,4),
    OperationDate DATETIME2 NOT NULL,
    INDEX IX_FactWH_TimeKey NONCLUSTERED (TimeKey),
    INDEX IX_FactWH_GeographyKey NONCLUSTERED (GeographyKey),
    INDEX IX_FactWH_ProductKey NONCLUSTERED (ProductKey),
    INDEX IX_FactWH_OperationType NONCLUSTERED (OperationType),
    CONSTRAINT FK_FactWH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_FactWH_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripID VARCHAR(100) UNIQUE NOT NULL,
    OrderID VARCHAR(100),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumed DECIMAL(10,4),
    FuelCost DECIMAL(18,4),
    LoadWeight DECIMAL(10,2),
    LoadValue DECIMAL(18,4),
    DelayMinutes INT DEFAULT 0,
    DelayReason VARCHAR(200),
    DriverID VARCHAR(50),
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2,
    INDEX IX_FactFleet_TimeKey NONCLUSTERED (TimeKey),
    INDEX IX_FactFleet_VehicleKey NONCLUSTERED (VehicleKey),
    INDEX IX_FactFleet_OriginKey NONCLUSTERED (OriginGeographyKey),
    INDEX IX_FactFleet_DestinationKey NONCLUSTERED (DestinationGeographyKey),
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE TABLE FactInventorySnapshot (
    SnapshotKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SnapshotDate DATE NOT NULL,
    QuantityOnHand INT NOT NULL,
    QuantityReserved INT DEFAULT 0,
    QuantityAvailable AS (QuantityOnHand - QuantityReserved),
    InventoryValue DECIMAL(18,4),
    DaysOfSupply INT,
    ReorderPoint INT,
    SafetyStock INT,
    INDEX IX_FactInv_TimeKey NONCLUSTERED (TimeKey),
    INDEX IX_FactInv_GeographyKey NONCLUSTERED (GeographyKey),
    INDEX IX_FactInv_ProductKey NONCLUSTERED (ProductKey),
    INDEX IX_FactInv_SnapshotDate NONCLUSTERED (SnapshotDate),
    CONSTRAINT FK_FactInv_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactInv_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactInv_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Step 2: Create Helper Stored Procedures

```sql
-- Procedure to calculate warehouse gravity scores
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(DISTINCT w.OperationKey) AS OperationCount,
            AVG(CAST(w.DwellTimeMinutes AS FLOAT)) AS AvgDwellTime,
            SUM(w.Quantity) AS TotalVolume,
            p.UnitPrice
        FROM DimProduct p
        LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
        WHERE w.OperationDate >= DATEADD(MONTH, -3, GETDATE())
        GROUP BY p.ProductKey, p.UnitPrice
    ),
    Normalized AS (
        SELECT 
            ProductKey,
            -- Frequency score (0-100)
            CASE 
                WHEN MAX(OperationCount) OVER() = 0 THEN 50
                ELSE (OperationCount * 100.0) / NULLIF(MAX(OperationCount) OVER(), 0)
            END AS FrequencyScore,
            -- Velocity score (inverse of dwell time)
            CASE 
                WHEN AvgDwellTime = 0 THEN 100
                WHEN MAX(AvgDwellTime) OVER() = 0 THEN 50
                ELSE 100 - ((AvgDwellTime * 100.0) / NULLIF(MAX(AvgDwellTime) OVER(), 0))
            END AS VelocityScore,
            -- Value score
            CASE 
                WHEN MAX(UnitPrice) OVER() = 0 THEN 50
                ELSE (UnitPrice * 100.0) / NULLIF(MAX(UnitPrice) OVER(), 0)
            END AS ValueScore
        FROM ProductMetrics
    )
    UPDATE p
    SET 
        GravityScore = (n.FrequencyScore * 0.4) + (n.VelocityScore * 0.4) + (n.ValueScore * 0.2),
        VelocityClass = CASE 
            WHEN (n.FrequencyScore * 0.4) + (n.VelocityScore * 0.4) + (n.ValueScore * 0.2) >= 70 THEN 'Fast'
            WHEN (n.FrequencyScore * 0.4) + (n.VelocityScore * 0.4) + (n.ValueScore * 0.2) >= 40 THEN 'Medium'
            ELSE 'Slow'
        END
    FROM DimProduct p
    INNER JOIN Normalized n ON p.ProductKey = n.ProductKey;
END;
GO

-- Procedure for cross-fact KPI calculation
CREATE PROCEDURE usp_GetCrossFactKPIs
    @StartDate DATE,
    @EndDate DATE,
    @GeographyKey INT = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        g.LocationName,
        g.Country,
        
        -- Warehouse KPIs
        COUNT(DISTINCT w.OperationID) AS TotalOperations,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        SUM(w.Quantity) AS TotalUnitsProcessed,
        SUM(w.HandlingCost) AS TotalHandlingCost,
        
        -- Fleet KPIs
        COUNT(DISTINCT f.TripID) AS TotalTrips,
        SUM(f.DistanceKm) AS TotalDistanceKm,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        SUM(f.FuelCost) AS TotalFuelCost,
        SUM(f.DelayMinutes) AS TotalDelayMinutes,
        
        -- Inventory KPIs
        AVG(i.QuantityOnHand) AS AvgInventoryLevel,
        AVG(i.DaysOfSupply) AS AvgDaysOfSupply,
        
        -- Cross-fact calculated metrics
        CASE 
            WHEN SUM(f.DistanceKm) > 0 THEN SUM(f.FuelCost) / SUM(f.DistanceKm)
            ELSE 0 
        END AS CostPerKm,
        
        CASE 
            WHEN COUNT(DISTINCT w.OperationID) > 0 
            THEN SUM(w.HandlingCost) / COUNT(DISTINCT w.OperationID)
            ELSE 0 
        END AS CostPerOperation,
        
        CASE 
            WHEN SUM(w.Quantity) > 0 
            THEN (AVG(w.DwellTimeMinutes) * SUM(f.IdleTimeMinutes)) / SUM(w.Quantity)
            ELSE 0 
        END AS EfficiencyIndex
        
    FROM DimGeography g
    LEFT JOIN FactWarehouseOperations w ON g.GeographyKey = w.GeographyKey
        AND w.OperationDate BETWEEN @StartDate AND @EndDate
    LEFT JOIN FactFleetTrips f ON (g.GeographyKey = f.OriginGeographyKey OR g.GeographyKey = f.DestinationGeographyKey)
        AND f.TripStartTime BETWEEN @StartDate AND @EndDate
    LEFT JOIN FactInventorySnapshot i ON g.GeographyKey = i.GeographyKey
        AND i.SnapshotDate BETWEEN @StartDate AND @EndDate
    WHERE (@GeographyKey IS NULL OR g.GeographyKey = @GeographyKey)
    GROUP BY g.GeographyKey, g.LocationName, g.Country
    ORDER BY TotalOperations DESC;
END;
GO
```

### Step 3: Configure Power BI Connection

Create a `config.json` file (do not commit to version control):

```json
{
  "sqlServer": {
    "server": "your-server.database.windows.net",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "refreshSchedule": {
    "intervalMinutes": 15,
    "enabled": true
  },
  "security": {
    "rowLevelSecurity": true,
    "defaultRole": "User"
  }
}
```

**Environment variables to set:**
- `SQL_USERNAME` - Database username
- `SQL_PASSWORD` - Database password
- `POWERBI_WORKSPACE` - Power BI workspace ID (optional)

## Key SQL Queries and Patterns

### Pattern 1: Cross-Fact Analysis - Warehouse Dwell vs Fleet Idle

```sql
-- Correlate warehouse dwell time with fleet idle time by product
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(w.OperationKey) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Picking'
        AND w.OperationDate >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY p.SKU, p.ProductName
),
FleetIdle AS (
    SELECT 
        p.SKU,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(f.TripKey) AS TripCount
    FROM FactFleetTrips f
    INNER JOIN FactWarehouseOperations w ON f.OrderID = w.OrderID
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    WHERE f.TripStartTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY p.SKU
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellTime,
    wd.OperationCount,
    fi.AvgIdleTime,
    fi.TripCount,
    -- Correlation metric
    (wd.AvgDwellTime * fi.AvgIdleTime) AS CorrelationIndex,
    -- Opportunity score (higher = more optimization potential)
    CASE 
        WHEN wd.AvgDwellTime > 120 AND fi.AvgIdleTime > 30 THEN 'High'
        WHEN wd.AvgDwellTime > 60 OR fi.AvgIdleTime > 15 THEN 'Medium'
        ELSE 'Low'
    END AS OptimizationPotential
FROM WarehouseDwell wd
LEFT JOIN FleetIdle fi ON wd.SKU = fi.SKU
WHERE fi.TripCount > 0
ORDER BY CorrelationIndex DESC;
```

### Pattern 2: Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks based on trending metrics
WITH HourlyMetrics AS (
    SELECT 
        t.DateKey,
        t.HourOfDay,
        g.LocationName,
        COUNT(w.OperationKey) AS OperationVolume,
        AVG(w.ProcessingTimeMinutes) AS AvgProcessingTime,
        STDEV(w.ProcessingTimeMinutes) AS ProcessingTimeStdDev
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    WHERE w.OperationDate >= DATEADD(DAY, -7, GETDATE())
    GROUP BY t.DateKey, t.HourOfDay, g.LocationName
),
Trending AS (
    SELECT 
        LocationName,
        HourOfDay,
        AVG(OperationVolume) AS AvgVolume,
        AVG(AvgProcessingTime) AS AvgTime,
        AVG(ProcessingTimeStdDev) AS AvgVariability,
        -- Detect upward trend in processing time
        CASE 
            WHEN AVG(AvgProcessingTime) > (
                SELECT AVG(AvgProcessingTime) * 1.2 
                FROM HourlyMetrics hm2 
                WHERE hm2.LocationName = HourlyMetrics.LocationName
            ) THEN 1
            ELSE 0
        END AS IsTrendingSlower
    FROM HourlyMetrics
    GROUP BY LocationName, HourOfDay
)
SELECT 
    LocationName,
    HourOfDay,
    AvgVolume,
    AvgTime,
    AvgVariability,
    -- Bottleneck risk score (0-100)
    (AvgVolume * 0.3) + (AvgTime * 0.4) + (AvgVariability * 0.2) + (IsTrendingSlower * 100 * 0.1) AS BottleneckRiskScore,
    CASE 
        WHEN (AvgVolume * 0.3) + (AvgTime * 0.4) + (AvgVariability * 0.2) + (IsTrendingSlower * 100 * 0.1) >= 70 THEN 'Critical'
        WHEN (AvgVolume * 0.3) + (AvgTime * 0.4) + (AvgVariability * 0.2) + (IsTrendingSlower * 100 * 0.1) >= 50 THEN 'Warning'
        ELSE 'Normal'
    END AS Status
FROM Trending
WHERE IsTrendingSlower = 1 OR AvgTime > 15
ORDER BY BottleneckRiskScore DESC;
```

### Pattern 3: Fleet Triage Priority Queue

```sql
-- Generate maintenance priority based on revenue impact
WITH VehicleMetrics AS (
    SELECT 
        v.VehicleID,
        v.VehicleType,
        v.CurrentMileage,
        v.LastMaintenanceDate,
        DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
        SUM(f.LoadValue) AS TotalLoadValue,
        SUM(f.DelayMinutes) AS TotalDelayMinutes,
        COUNT(f.TripKey) AS TripCount,
        AVG(f.FuelConsumed / NULLIF(f.DistanceKm, 0)) AS FuelEfficiency
    FROM DimVehicle v
    LEFT JOIN FactFleetTrips f ON v.VehicleKey = f.VehicleKey
    WHERE f.TripStartTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID, v.VehicleType, v.CurrentMileage, v.LastMaintenanceDate
)
SELECT 
    VehicleID,
    VehicleType,
    CurrentMileage,
    DaysSinceLastMaintenance,
    TotalLoadValue,
    TotalDelayMinutes,
    TripCount,
    FuelEfficiency,
    -- Priority score combining multiple factors
    (
        (DaysSinceLastMaintenance / 365.0 * 25) +  -- Age factor
        (TotalDelayMinutes / 60.0 * 20) +          -- Delay impact
        (CASE WHEN FuelEfficiency > 0.15 THEN 30 ELSE 0 END) + -- Efficiency flag
        ((TotalLoadValue / NULLIF(TripCount, 0)) / 1000.0 * 25) -- Revenue impact
    ) AS MaintenancePriority,
    CASE 
        WHEN DaysSinceLastMaintenance > 180 OR TotalDelayMinutes > 300 THEN 'Urgent'
        WHEN DaysSinceLastMaintenance > 90 OR TotalDelayMinutes > 120 THEN 'High'
        WHEN DaysSinceLastMaintenance > 60 THEN 'Medium'
        ELSE 'Low'
    END AS PriorityLevel
FROM VehicleMetrics
ORDER BY MaintenancePriority DESC;
```

### Pattern 4: Warehouse Gravity Zone Recommendations

```sql
-- Recommend storage zone reassignments based on gravity scores
WITH CurrentZones AS (
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.VelocityClass,
        w.StorageZone,
        COUNT(*) AS PickCount,
        AVG(w.PickPathLength) AS AvgPathLength
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'Picking'
        AND w.OperationDate >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY p.SKU, p.ProductName, p.GravityScore, p.VelocityClass, w.StorageZone
),
ZoneOptimization AS (
    SELECT 
        SKU,
        ProductName,
        GravityScore,
        VelocityClass,
        StorageZone AS CurrentZone,
        PickCount,
        AvgPathLength,
        -- Recommended zone based on gravity score
        CASE 
            WHEN GravityScore >= 70 THEN 'Zone-A-Fast'  -- Near shipping
            WHEN GravityScore >= 40 THEN 'Zone-B-Medium' -- Mid-warehouse
            ELSE 'Zone-C-Slow'                           -- Deep storage
        END AS RecommendedZone,
        -- Estimated path length reduction
        CASE 
            WHEN GravityScore >= 70 AND StorageZone NOT LIKE '%Zone-A%' THEN AvgPathLength * 0.6
            WHEN GravityScore >= 40 AND StorageZone NOT LIKE '%Zone-B%' THEN AvgPathLength * 0.8
            ELSE AvgPathLength
        END AS EstimatedPathLength
    FROM CurrentZones
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    CurrentZone,
    RecommendedZone,
    PickCount,
    AvgPathLength,
    EstimatedPathLength,
    (AvgPathLength - EstimatedPathLength) AS PathReduction,
    (AvgPathLength - EstimatedPathLength) * PickCount AS TotalPathSavings,
    CASE 
        WHEN CurrentZone <> RecommendedZone THEN 'Reassign'
        ELSE 'Current OK'
    END AS Action
FROM ZoneOptimization
WHERE CurrentZone <> RecommendedZone
ORDER BY TotalPathSavings DESC;
```

## Data Loading Patterns

### Incremental Load from External Sources

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assume external staging table exists
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationID, OrderID, Quantity,
        DwellTimeMinutes, ProcessingTimeMinutes, PickPathLength,
        StorageZone, HandlingCost, OperationDate
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.OperationID,
        stg.OrderID,
        stg.Quantity,
        stg.DwellTimeMinutes,
        stg.ProcessingTimeMinutes,
        stg.PickPathLength,
        stg.StorageZone,
        stg.HandlingCost,
        stg.OperationDate
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON CAST(stg.OperationDate AS DATE) = CAST(t.FullDateTime AS DATE)
        AND DATEPART(HOUR, stg.OperationDate) = t.HourOfDay
        AND DATEPART(MINUTE, stg.OperationDate) / 15 * 15 = DATEPART(MINUTE, t.TimeOfDay)
    INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
    INNER JOIN DimProduct p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplier s ON stg.SupplierID = s.SupplierID
    WHERE stg.OperationDate > @LastLoadTimestamp
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f 
            WHERE f.OperationID = stg.OperationID
        );
    
    -- Update last load timestamp
    UPDATE ETLConfig SET LastLoadTimestamp = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Measure: Total Efficiency Score
TotalEfficiencyScore = 
VAR WarehouseCost = SUM(FactWarehouseOperations[HandlingCost])
VAR FleetCost = SUM(FactFleetTrips[FuelCost])
VAR TotalUnits = SUM(FactWarehouseOperations[Quantity])
VAR TotalDistance = SUM(FactFleetTrips[DistanceKm])

RETURN
IF(
    TotalUnits > 0 && TotalDistance > 0,
    100 - (
        (WarehouseCost / TotalUnits * 0.5) + 
        (FleetCost / TotalDistance * 0.5)
    ),
    BLANK()
)

// Measure: Dwell-Idle Correlation
DwellIdleCorrelation = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])

RETURN
IF(
    NOT ISBLANK(AvgDwell) && NOT ISBLANK(AvgIdle),
    AvgDwell * AvgIdle / 100,
    BLANK()
)

// Measure: Predictive Bottleneck Index
BottleneckIndex = 
VAR CurrentVolume = COUNTROWS(FactWarehouseOperations)
VAR HistoricalAvg = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DATESINPERIOD(DimTime[
