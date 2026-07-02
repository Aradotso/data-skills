---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and supply chain KPI harmonization
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure multi-fact star schema for fleet tracking
  - implement warehouse gravity zones analytics
  - create cross-modal supply chain dashboard
  - build logistics intelligence sql server database
  - integrate fleet telemetry with warehouse operations
  - design predictive bottleneck detection system
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform built on MS SQL Server and Power BI. It implements a multi-fact star schema data warehouse that unifies warehouse operations, fleet telemetry, inventory management, and supply chain KPIs into a single semantic layer. The platform enables cross-fact queries, predictive bottleneck detection, and real-time operational dashboards.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Warehouse Gravity Zones™ spatial optimization
- Real-time fleet triage and maintenance prioritization
- Temporal elasticity modeling and scenario simulation
- Role-based Power BI dashboards with natural language queries

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telematics APIs, ERP systems

### Step 1: Deploy SQL Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimensional tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourBucket TINYINT NOT NULL, -- 0-23
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    FiscalPeriod VARCHAR(7) NOT NULL, -- YYYY-MM
    IsWeekend BIT NOT NULL,
    INDEX IX_DimTime_DateTime NONCLUSTERED (DateTime),
    INDEX IX_DimTime_Date NONCLUSTERED (Date)
)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID VARCHAR(50) NOT NULL UNIQUE,
    NodeName VARCHAR(100) NOT NULL,
    NodeType VARCHAR(20) NOT NULL, -- Warehouse, Hub, Customer, Route
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    INDEX IX_DimGeography_NodeID NONCLUSTERED (NodeID),
    INDEX IX_DimGeography_Country NONCLUSTERED (Country)
)
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsPerishable BIT NOT NULL DEFAULT 0,
    IsFragile BIT NOT NULL DEFAULT 0,
    GravityScore DECIMAL(5,2) NOT NULL DEFAULT 1.00, -- Calculated velocity metric
    OptimalStorageTemp DECIMAL(5,2),
    ShelfLifeDays INT,
    INDEX IX_DimProduct_SKU NONCLUSTERED (SKU),
    INDEX IX_DimProduct_GravityScore NONCLUSTERED (GravityScore DESC)
)
GO

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL UNIQUE,
    VehicleType VARCHAR(50) NOT NULL, -- Truck, Van, Trailer
    Make VARCHAR(50),
    Model VARCHAR(50),
    YearManufactured INT,
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(20),
    HomeWarehouseKey INT,
    IsActive BIT NOT NULL DEFAULT 1,
    INDEX IX_DimFleet_VehicleID NONCLUSTERED (VehicleID)
)
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    Country VARCHAR(100),
    AvgLeadTimeDays INT,
    ReliabilityScore DECIMAL(5,2), -- 0-100
    DefectRate DECIMAL(5,4), -- Percentage
    INDEX IX_DimSupplier_SupplierID NONCLUSTERED (SupplierID)
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    OperationID VARCHAR(50) NOT NULL UNIQUE,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeSeconds INT,
    OperatorID VARCHAR(50),
    ZoneID VARCHAR(20),
    BinLocation VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    INDEX IX_FactWarehouse_Time NONCLUSTERED (TimeKey),
    INDEX IX_FactWarehouse_Product NONCLUSTERED (ProductKey),
    INDEX IX_FactWarehouse_Geography NONCLUSTERED (GeographyKey)
)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID VARCHAR(50) NOT NULL UNIQUE,
    FleetKey INT NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    AverageSpeed DECIMAL(5,2),
    MaxSpeed DECIMAL(5,2),
    LoadWeight DECIMAL(10,2),
    DriverID VARCHAR(50),
    DelayMinutes INT DEFAULT 0,
    DelayReason VARCHAR(100),
    FOREIGN KEY (FleetKey) REFERENCES DimFleet(FleetKey),
    FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_FactTrips_Fleet NONCLUSTERED (FleetKey),
    INDEX IX_FactTrips_StartTime NONCLUSTERED (StartTimeKey),
    INDEX IX_FactTrips_Origin NONCLUSTERED (OriginGeographyKey)
)
GO

CREATE TABLE FactInventory (
    InventoryKey BIGINT PRIMARY KEY IDENTITY(1,1),
    SnapshotTimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    QuantityOnHand INT NOT NULL,
    QuantityReserved INT NOT NULL DEFAULT 0,
    QuantityAvailable AS (QuantityOnHand - QuantityReserved),
    AgingDays INT NOT NULL,
    ZoneID VARCHAR(20),
    BinLocation VARCHAR(50),
    FOREIGN KEY (SnapshotTimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey),
    INDEX IX_FactInventory_Time NONCLUSTERED (SnapshotTimeKey),
    INDEX IX_FactInventory_Product NONCLUSTERED (ProductKey),
    INDEX IX_FactInventory_Geography NONCLUSTERED (GeographyKey)
)
GO
```

### Step 2: Create Bridge Tables for Many-to-Many Relationships

```sql
-- Bridge table linking trips to products shipped
CREATE TABLE BridgeTripProduct (
    TripKey BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    QuantityShipped INT NOT NULL,
    PRIMARY KEY (TripKey, ProductKey),
    FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO
```

### Step 3: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime with 15-minute granularity
CREATE PROCEDURE PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    DECLARE @EndDateTime DATETIME = DATEADD(DAY, 1, @EndDate);
    
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        INSERT INTO DimTime (
            DateTime, Date, TimeOfDay, HourBucket, MinuteBucket,
            DayOfWeek, DayName, FiscalPeriod, IsWeekend
        )
        VALUES (
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            FORMAT(@CurrentDateTime, 'yyyy-MM'),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

-- Execute to populate 2 years of time data
EXEC PopulateDimTime '2025-01-01', '2026-12-31';
GO
```

### Step 4: Create Key Views and Stored Procedures

```sql
-- Cross-fact view for unified KPI analysis
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    t.Date,
    t.FiscalPeriod,
    g.Country,
    g.Region,
    g.NodeName,
    p.Category AS ProductCategory,
    
    -- Warehouse metrics
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.Quantity ELSE 0 END) AS TotalPickedUnits,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.ProcessingTimeSeconds ELSE NULL END) AS AvgPickTimeSeconds,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    
    -- Fleet metrics
    SUM(ft.DistanceKm) AS TotalDistanceKm,
    SUM(ft.FuelConsumedLiters) AS TotalFuelConsumed,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    SUM(ft.DelayMinutes) AS TotalDelayMinutes,
    
    -- Inventory metrics
    AVG(inv.QuantityOnHand) AS AvgInventoryLevel,
    AVG(inv.AgingDays) AS AvgInventoryAgingDays
    
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.StartTimeKey
LEFT JOIN FactInventory inv ON t.TimeKey = inv.SnapshotTimeKey
LEFT JOIN DimGeography g ON COALESCE(wo.GeographyKey, ft.OriginGeographyKey, inv.GeographyKey) = g.GeographyKey
LEFT JOIN DimProduct p ON COALESCE(wo.ProductKey, inv.ProductKey) = p.ProductKey

WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())

GROUP BY 
    t.Date,
    t.FiscalPeriod,
    g.Country,
    g.Region,
    g.NodeName,
    p.Category
GO

-- Calculate Product Gravity Score based on velocity and value
CREATE PROCEDURE UpdateProductGravityScores AS
BEGIN
    UPDATE p
    SET GravityScore = (
        -- Velocity component (pick frequency)
        (SELECT COUNT(*) * 1.0 
         FROM FactWarehouseOperations wo 
         WHERE wo.ProductKey = p.ProductKey 
           AND wo.OperationType = 'Picking'
           AND wo.TimeKey >= (SELECT TOP 1 TimeKey FROM DimTime WHERE Date >= DATEADD(DAY, -30, GETDATE()))
        ) / 30.0 -- Daily average picks
        +
        -- Value component (high value = higher gravity)
        CASE WHEN p.IsFragile = 1 THEN 10.0 ELSE 0.0 END
        +
        CASE WHEN p.IsPerishable = 1 THEN 15.0 ELSE 0.0 END
    )
    FROM DimProduct p
END
GO

-- Predictive bottleneck detection procedure
CREATE PROCEDURE DetectBottlenecks
    @ThresholdDwellMinutes INT = 120,
    @ThresholdIdlePercent DECIMAL(5,2) = 15.0
AS
BEGIN
    SELECT 
        'Warehouse Dwell Time' AS BottleneckType,
        g.NodeName AS Location,
        p.SKU,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OccurrenceCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
      AND wo.DwellTimeMinutes > @ThresholdDwellMinutes
    GROUP BY g.NodeName, p.SKU
    HAVING COUNT(*) > 3
    
    UNION ALL
    
    SELECT 
        'Fleet Idle Time' AS BottleneckType,
        g.NodeName AS Location,
        f.VehicleID AS Identifier,
        AVG((ft.IdleTimeMinutes * 100.0) / NULLIF(ft.DurationMinutes, 0)) AS AvgIdlePercent,
        COUNT(*) AS OccurrenceCount
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
    INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
    WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY g.NodeName, f.VehicleID
    HAVING AVG((ft.IdleTimeMinutes * 100.0) / NULLIF(ft.DurationMinutes, 0)) > @ThresholdIdlePercent
    
    ORDER BY AvgDwellMinutes DESC, AvgIdlePercent DESC
END
GO
```

## Power BI Configuration

### Connecting Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server details:
   ```
   Server: YOUR_SQL_SERVER_NAME
   Database: LogiFleetPulse
   ```
4. Use Windows Authentication or SQL Server credentials (stored in environment variables)
5. Select tables: All dimension tables (Dim*) and fact tables (Fact*)

### DAX Measures for Cross-Fact KPIs

```dax
// Measure: Total Fleet Idle Time Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Measure: Warehouse Pick Efficiency
Pick Efficiency = 
DIVIDE(
    SUM(FactWarehouseOperations[Quantity]),
    SUM(FactWarehouseOperations[ProcessingTimeSeconds]) / 3600,
    0
) // Units per hour

// Measure: Cross-Fact KPI - Dwell Time vs Fuel Waste Correlation
Dwell-Fuel Impact Score = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdlePercent = [Fleet Idle %]
RETURN
    (AvgDwell / 60) * AvgIdlePercent // Higher score = more correlation

// Measure: Inventory Turnover Rate
Inventory Turnover = 
VAR ShippedQty = CALCULATE(
    SUM(FactWarehouseOperations[Quantity]),
    FactWarehouseOperations[OperationType] = "Shipping"
)
VAR AvgInventory = AVERAGE(FactInventory[QuantityOnHand])
RETURN
    DIVIDE(ShippedQty, AvgInventory, 0)

// Measure: Predictive Bottleneck Index (composite score)
Bottleneck Index = 
VAR DwellScore = 
    DIVIDE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        120, // Threshold
        0
    )
VAR IdleScore = [Fleet Idle %] / 15 // Threshold 15%
VAR InventoryAgeScore = 
    DIVIDE(
        AVERAGE(FactInventory[AgingDays]),
        30, // Threshold
        0
    )
RETURN
    (DwellScore + IdleScore + InventoryAgeScore) / 3 * 100

// Time Intelligence: Month-over-Month Change
MoM Fleet Distance = 
VAR CurrentMonth = SUM(FactFleetTrips[DistanceKm])
VAR PreviousMonth = 
    CALCULATE(
        SUM(FactFleetTrips[DistanceKm]),
        DATEADD(DimTime[Date], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth, 0) * 100
```

### Row-Level Security (RLS)

```dax
// Create role: Regional Manager (only sees their region)
[Region] = USERNAME()

// Create role: Warehouse Supervisor (only sees their warehouse)
[NodeName] = LOOKUPVALUE(
    UserWarehouseMapping[WarehouseName],
    UserWarehouseMapping[Username],
    USERNAME()
)
```

Apply roles in Power BI Desktop:
- Modeling → Manage Roles → New
- Add DAX filter expression
- Test with "View as Role" feature

## Data Integration Patterns

### ETL Pattern: Incremental Load from Telematics API

```sql
-- Stored procedure for incremental fleet data load
CREATE PROCEDURE LoadFleetTelemetry
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage table for raw API data
    CREATE TABLE #StagingFleetData (
        VehicleID VARCHAR(50),
        TripID VARCHAR(50),
        StartDateTime DATETIME,
        EndDateTime DATETIME,
        OriginNodeID VARCHAR(50),
        DestinationNodeID VARCHAR(50),
        DistanceKm DECIMAL(10,2),
        FuelConsumedLiters DECIMAL(10,2),
        IdleTimeMinutes INT,
        DriverID VARCHAR(50)
    )
    
    -- INSERT logic here would connect to external API
    -- Using OPENROWSET or external table for JSON/CSV import
    -- Example placeholder:
    -- INSERT INTO #StagingFleetData SELECT * FROM OPENROWSET(...)
    
    -- Transform and load into fact table
    INSERT INTO FactFleetTrips (
        TripID, FleetKey, StartTimeKey, EndTimeKey,
        OriginGeographyKey, DestinationGeographyKey,
        DistanceKm, DurationMinutes, FuelConsumedLiters,
        IdleTimeMinutes, DriverID
    )
    SELECT 
        s.TripID,
        f.FleetKey,
        tStart.TimeKey,
        tEnd.TimeKey,
        gOrigin.GeographyKey,
        gDest.GeographyKey,
        s.DistanceKm,
        DATEDIFF(MINUTE, s.StartDateTime, s.EndDateTime),
        s.FuelConsumedLiters,
        s.IdleTimeMinutes,
        s.DriverID
    FROM #StagingFleetData s
    INNER JOIN DimFleet f ON s.VehicleID = f.VehicleID
    INNER JOIN DimTime tStart ON s.StartDateTime = tStart.DateTime
    INNER JOIN DimTime tEnd ON s.EndDateTime = tEnd.DateTime
    INNER JOIN DimGeography gOrigin ON s.OriginNodeID = gOrigin.NodeID
    INNER JOIN DimGeography gDest ON s.DestinationNodeID = gDest.NodeID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactFleetTrips ft WHERE ft.TripID = s.TripID
    )
    
    DROP TABLE #StagingFleetData
END
GO
```

### Real-Time Dashboard Refresh Pattern

```sql
-- Create indexed view for dashboard performance
CREATE VIEW vw_FleetDashboardRealTime
WITH SCHEMABINDING
AS
SELECT 
    f.VehicleID,
    g.NodeName AS CurrentLocation,
    t.DateTime AS LastUpdateTime,
    ft.DistanceKm,
    ft.IdleTimeMinutes,
    ft.DelayMinutes
FROM dbo.FactFleetTrips ft
INNER JOIN dbo.DimFleet f ON ft.FleetKey = f.FleetKey
INNER JOIN dbo.DimTime t ON ft.EndTimeKey = t.TimeKey
INNER JOIN dbo.DimGeography g ON ft.DestinationGeographyKey = g.GeographyKey
WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
GO

-- Create clustered index on view for performance
CREATE UNIQUE CLUSTERED INDEX IX_FleetDashboard 
ON vw_FleetDashboardRealTime (VehicleID, LastUpdateTime)
GO
```

## Common Usage Patterns

### Pattern 1: Warehouse Gravity Zone Analysis

```sql
-- Query to identify products needing zone reallocation
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    wo.ZoneID AS CurrentZone,
    AVG(wo.ProcessingTimeSeconds) AS AvgPickTimeSeconds,
    COUNT(*) AS PickFrequency,
    CASE 
        WHEN p.GravityScore > 20 THEN 'High Gravity - Move to Zone A'
        WHEN p.GravityScore > 10 THEN 'Medium Gravity - Zone B'
        ELSE 'Low Gravity - Zone C'
    END AS RecommendedZone
FROM DimProduct p
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE wo.OperationType = 'Picking'
  AND t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, wo.ZoneID
HAVING COUNT(*) > 10
ORDER BY p.GravityScore DESC
```

### Pattern 2: Fleet Maintenance Prioritization

```sql
-- Proactive maintenance queue based on revenue risk
WITH FleetRisk AS (
    SELECT 
        f.VehicleID,
        f.VehicleType,
        AVG(ft.IdleTimeMinutes * 1.0 / NULLIF(ft.DurationMinutes, 0)) AS IdleRatio,
        SUM(ft.DistanceKm) AS TotalDistanceRecent,
        COUNT(CASE WHEN ft.DelayMinutes > 30 THEN 1 END) AS DelayIncidents,
        -- Calculate revenue at risk (simplified)
        SUM(ft.LoadWeight * 100) AS ApproxRevenueAtRisk -- $100 per kg
    FROM DimFleet f
    INNER JOIN FactFleetTrips ft ON f.FleetKey = ft.FleetKey
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -14, GETDATE())
      AND f.IsActive = 1
    GROUP BY f.VehicleID, f.VehicleType
)
SELECT 
    VehicleID,
    VehicleType,
    ROUND(IdleRatio * 100, 2) AS IdlePercent,
    TotalDistanceRecent AS RecentKm,
    DelayIncidents,
    ApproxRevenueAtRisk,
    -- Composite risk score
    (IdleRatio * 50 + DelayIncidents * 10 + TotalDistanceRecent / 1000) * 
    (ApproxRevenueAtRisk / 1000) AS MaintenancePriorityScore
FROM FleetRisk
ORDER BY MaintenancePriorityScore DESC
```

### Pattern 3: Cross-Fact Correlation Analysis

```sql
-- Analyze correlation between warehouse dwell time and fleet delays
SELECT 
    g.Region,
    p.Category,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    AVG(ft.DelayMinutes) AS AvgFleetDelayMinutes,
    COUNT(DISTINCT wo.OperationID) AS WarehouseOperations,
    COUNT(DISTINCT ft.TripID) AS FleetTrips,
    -- Simple correlation indicator
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > 120 AND AVG(ft.DelayMinutes) > 30 
        THEN 'High Correlation - Investigate'
        ELSE 'Normal'
    END AS CorrelationFlag
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON g.GeographyKey = ft.OriginGeographyKey 
    AND t.Date = (SELECT Date FROM DimTime WHERE TimeKey = ft.StartTimeKey)
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY g.Region, p.Category
HAVING COUNT(DISTINCT wo.OperationID) > 50
ORDER BY AvgWarehouseDwellMinutes DESC
```

### Pattern 4: Inventory Aging Alert

```sql
-- Identify slow-moving inventory at risk of obsolescence
SELECT 
    g.NodeName AS Warehouse,
    p.SKU,
    p.ProductName,
    inv.QuantityOnHand,
    inv.AgingDays,
    p.ShelfLifeDays,
    CASE 
        WHEN p.ShelfLifeDays IS NOT NULL AND inv.AgingDays > (p.ShelfLifeDays * 0.7)
        THEN 'URGENT - Approaching Expiry'
        WHEN inv.AgingDays > 90
        THEN 'High Risk - Slow Mover'
        WHEN inv.AgingDays > 60
        THEN 'Medium Risk'
        ELSE 'Normal'
    END AS RiskLevel,
    -- Recommended action
    CASE 
        WHEN inv.AgingDays > 90 THEN 'Discount & Clear'
        WHEN p.IsPerishable = 1 AND inv.AgingDays > 30 THEN 'Priority Ship'
        ELSE 'Monitor'
    END AS RecommendedAction
FROM FactInventory inv
INNER JOIN DimProduct p ON inv.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON inv.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON inv.SnapshotTimeKey = t.TimeKey
WHERE t.DateTime = (SELECT MAX(DateTime) FROM DimTime) -- Latest snapshot
  AND inv.QuantityOnHand > 0
  AND (inv.AgingDays > 60 OR (p.IsPerishable = 1 AND inv.AgingDays > 21))
ORDER BY 
    CASE RiskLevel
        WHEN 'URGENT - Approaching Expiry' THEN 1
        WHEN 'High Risk - Slow Mover' THEN 2
        ELSE 3
    END,
    inv.AgingDays DESC
```

## Configuration Management

### Environment Variables

Store connection strings and API credentials in environment variables:

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="tcp:yourserver.database.windows.net,1433"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USER="logifleet_app"
export LOGIFLEET_SQL_PASSWORD="stored_in_vault"

# Telematics API
export LOGIFLEET_TELEMETRY_API_ENDPOINT="https://api.telemetryprovider.com/v2"
export LOGIFLEET_TELEMETRY_API_KEY="stored_in_vault"

# WMS Integration
export LOGIFLEET_WMS_CONNECTION_STRING="Provider=SQLOLEDB;Server=wms-server;Database=WMS_Prod"

# Power BI Refresh Token (for automated refresh)
export LOGIFLEET_POWERBI_WORKSPACE_ID="your-workspace-guid"
export LOGIFLEET_POWERBI_DATASET_ID="your-dataset-guid"
```

### Connection String in SQL Scripts

```sql
-- Use SQL Server configuration for external data sources
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://yours
