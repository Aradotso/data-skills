---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics, fleet, and supply chain analytics with multi-fact star schema modeling
triggers:
  - "set up supply chain analytics dashboard"
  - "create logistics data warehouse schema"
  - "build fleet management analytics with Power BI"
  - "implement warehouse operations reporting"
  - "design multi-fact star schema for logistics"
  - "configure real-time supply chain KPI monitoring"
  - "integrate fleet telemetry with warehouse data"
  - "optimize logistics data model performance"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing solution for supply chain, logistics, and fleet management analytics. It provides:

- **Multi-fact star schema** architecture linking warehouse operations, fleet telemetry, cross-dock activities, and inventory management
- **Time-phased dimensions** with 15-minute granularity for real-time operational insights
- **Cross-fact KPI harmonization** to connect warehouse velocity with fleet performance metrics
- **Power BI dashboard templates** with role-based access and predictive analytics
- **Automated alerting** for KPI threshold breaches
- **Semantic layer** that unifies disparate data sources (WMS, TMS, telemetry, external APIs)

The platform is designed for 3PLs, retail chains, distributors, and any organization managing complex supply chain operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Appropriate permissions to create databases, tables, stored procedures, and views

### Step 1: Deploy the SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Day INT NOT NULL,
    DayOfWeek INT NOT NULL,
    Hour INT NOT NULL,
    Minute15Bucket INT NOT NULL, -- 0, 15, 30, 45
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
)

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime)
CREATE NONCLUSTERED INDEX IX_DimTime_Date ON DimTime(Date)
GO

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    AddressLine1 NVARCHAR(200),
    AddressLine2 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    ValidFrom DATETIME2 DEFAULT GETDATE(),
    ValidTo DATETIME2 NULL
)

CREATE NONCLUSTERED INDEX IX_DimGeography_LocationID ON DimGeography(LocationID)
CREATE NONCLUSTERED INDEX IX_DimGeography_LocationType ON DimGeography(LocationType)
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    ProductCategory NVARCHAR(100),
    ProductSubCategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2), -- kg
    UnitVolume DECIMAL(10,4), -- cubic meters
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    PickFrequencyScore DECIMAL(5,2), -- Historical picks per week
    ValueScore DECIMAL(10,2), -- Unit value in base currency
    GravityScore AS (PickFrequencyScore * 0.5 + ValueScore * 0.3 + CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) PERSISTED,
    RecommendedZone VARCHAR(50), -- High-Gravity, Medium-Gravity, Low-Gravity
    LastUpdated DATETIME2 DEFAULT GETDATE()
)

CREATE NONCLUSTERED INDEX IX_DimProductGravity_SKU ON DimProductGravity(SKU)
CREATE NONCLUSTERED INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC)
GO

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200) NOT NULL,
    SupplierCountry NVARCHAR(100),
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2), -- Standard deviation
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- A, B, C, D
    IsPreferredVendor BIT DEFAULT 0,
    LastEvaluationDate DATETIME2
)

CREATE NONCLUSTERED INDEX IX_DimSupplier_ID ON DimSupplierReliability(SupplierID)
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100) UNIQUE NOT NULL,
    OrderID VARCHAR(100),
    QuantityUnits INT NOT NULL,
    DurationMinutes DECIMAL(8,2),
    StorageZone VARCHAR(50),
    StorageAisle VARCHAR(20),
    StorageBin VARCHAR(20),
    DwellTimeHours DECIMAL(10,2), -- Time spent in storage before next movement
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50), -- Forklift, pallet jack, etc.
    ErrorCount INT DEFAULT 0,
    ReworkRequired BIT DEFAULT 0,
    Notes NVARCHAR(500),
    CreatedAt DATETIME2 DEFAULT GETDATE()
)

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_OperationType ON FactWarehouseOperations(OperationType)
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripID VARCHAR(100) UNIQUE NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes DECIMAL(8,2),
    IdleTimeMinutes DECIMAL(8,2),
    FuelLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,4),
    AvgSpeedKmh DECIMAL(5,2),
    MaxSpeedKmh DECIMAL(5,2),
    HarshBrakingCount INT DEFAULT 0,
    HarshAccelerationCount INT DEFAULT 0,
    WeatherCondition VARCHAR(50), -- Clear, Rain, Snow, Fog
    TrafficDelayMinutes DECIMAL(8,2),
    OnTimeArrival BIT,
    FuelEfficiencyKmPerLiter AS (CASE WHEN FuelLiters > 0 THEN DistanceKm / FuelLiters ELSE NULL END) PERSISTED,
    CreatedAt DATETIME2 DEFAULT GETDATE()
)

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Destination ON FactFleetTrips(DestinationGeographyKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
GO

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripID VARCHAR(100),
    OutboundTripID VARCHAR(100),
    QuantityUnits INT NOT NULL,
    DockDurationMinutes DECIMAL(8,2),
    InboundArrivalTime DATETIME2,
    OutboundDepartureTime DATETIME2,
    CrossDockZone VARCHAR(50),
    TemperatureC DECIMAL(5,2), -- For temperature-controlled items
    QualityCheckPassed BIT DEFAULT 1,
    CreatedAt DATETIME2 DEFAULT GETDATE()
)

CREATE NONCLUSTERED INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactCrossDock_Geography ON FactCrossDock(GeographyKey)
CREATE NONCLUSTERED INDEX IX_FactCrossDock_Product ON FactCrossDock(ProductKey)
GO
```

### Step 2: Create Bridge Tables for Many-to-Many Relationships

```sql
-- Bridge table linking routes to multiple storage zones
CREATE TABLE BridgeRoutesZones (
    BridgeKey INT PRIMARY KEY IDENTITY(1,1),
    TripKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OperationKey BIGINT NOT NULL FOREIGN KEY REFERENCES FactWarehouseOperations(OperationKey),
    LinkType VARCHAR(50) NOT NULL, -- Pickup, Delivery, Transfer
    LinkSequence INT NOT NULL,
    LinkTimestamp DATETIME2 NOT NULL
)

CREATE NONCLUSTERED INDEX IX_Bridge_Trip ON BridgeRoutesZones(TripKey)
CREATE NONCLUSTERED INDEX IX_Bridge_Operation ON BridgeRoutesZones(OperationKey)
GO
```

### Step 3: Create Aggregation Views for Cross-Fact KPIs

```sql
-- View: Warehouse velocity vs fleet performance
CREATE VIEW vw_WarehouseFleetCrossKPI AS
SELECT 
    t.Date,
    g.LocationName AS WarehouseName,
    p.ProductCategory,
    -- Warehouse metrics
    COUNT(DISTINCT wo.OperationKey) AS TotalOperations,
    AVG(wo.DurationMinutes) AS AvgOperationDurationMin,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(wo.QuantityUnits) AS TotalUnitsProcessed,
    -- Fleet metrics
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTimeMin,
    AVG(ft.FuelEfficiencyKmPerLiter) AS AvgFuelEfficiency,
    SUM(ft.TrafficDelayMinutes) AS TotalTrafficDelayMin,
    -- Cross-fact KPI: Dwell time impact on delivery
    AVG(wo.DwellTimeHours) * AVG(ft.IdleTimeMinutes) AS DwellIdleImpactScore
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN BridgeRoutesZones brz ON wo.OperationKey = brz.OperationKey
LEFT JOIN FactFleetTrips ft ON brz.TripKey = ft.TripKey
WHERE g.LocationType = 'Warehouse'
GROUP BY t.Date, g.LocationName, p.ProductCategory
GO

-- View: Product gravity zone optimization recommendations
CREATE VIEW vw_ProductGravityRecommendations AS
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZone AS CurrentRecommendedZone,
    wo.StorageZone AS ActualStorageZone,
    AVG(wo.DurationMinutes) AS AvgPickTimeMin,
    COUNT(wo.OperationKey) AS PickFrequency,
    CASE 
        WHEN p.GravityScore > 50 AND wo.StorageZone NOT LIKE '%High-Gravity%' THEN 'Move to High-Gravity Zone'
        WHEN p.GravityScore < 20 AND wo.StorageZone NOT LIKE '%Low-Gravity%' THEN 'Move to Low-Gravity Zone'
        ELSE 'Optimally Placed'
    END AS OptimizationAction
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
WHERE wo.OperationType = 'Picking'
    AND wo.CreatedAt >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZone, wo.StorageZone
GO
```

### Step 4: Create Stored Procedures for Incremental Loading

```sql
-- Stored procedure: Incremental load for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external staging table or API
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        OperationID, OrderID, QuantityUnits, DurationMinutes, 
        StorageZone, DwellTimeHours, OperatorID, EquipmentID
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        stg.OperationType,
        stg.OperationID,
        stg.OrderID,
        stg.QuantityUnits,
        stg.DurationMinutes,
        stg.StorageZone,
        stg.DwellTimeHours,
        stg.OperatorID,
        stg.EquipmentID
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime dt ON CAST(stg.OperationTimestamp AS DATE) = dt.Date 
        AND DATEPART(HOUR, stg.OperationTimestamp) = dt.Hour
        AND (DATEPART(MINUTE, stg.OperationTimestamp) / 15) * 15 = dt.Minute15Bucket
    INNER JOIN DimGeography dg ON stg.LocationID = dg.LocationID
    INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU
    WHERE stg.OperationTimestamp > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations 
            WHERE OperationID = stg.OperationID
        )
    
    RETURN @@ROWCOUNT
END
GO

-- Stored procedure: Automated KPI alert generation
CREATE PROCEDURE sp_GenerateKPIAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertTable TABLE (
        AlertType VARCHAR(100),
        AlertMessage NVARCHAR(500),
        Severity VARCHAR(20),
        MetricValue DECIMAL(10,2)
    )
    
    -- Check fleet idle time threshold (>15% of trip duration)
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time Breach' AS AlertType,
        'Vehicle ' + VehicleID + ' exceeded 15% idle time threshold' AS AlertMessage,
        'High' AS Severity,
        (IdleTimeMinutes / NULLIF(DurationMinutes, 0)) * 100 AS MetricValue
    FROM FactFleetTrips
    WHERE CreatedAt >= DATEADD(HOUR, -1, GETDATE())
        AND (IdleTimeMinutes / NULLIF(DurationMinutes, 0)) > 0.15
    
    -- Check warehouse dwell time (>72 hours)
    INSERT INTO @AlertTable
    SELECT 
        'Excessive Dwell Time' AS AlertType,
        'SKU ' + p.SKU + ' at ' + g.LocationName + ' has dwell time > 72 hours' AS AlertMessage,
        'Medium' AS Severity,
        wo.DwellTimeHours AS MetricValue
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.CreatedAt >= DATEADD(HOUR, -1, GETDATE())
        AND wo.DwellTimeHours > 72
        AND p.IsPerishable = 1
    
    -- Return alerts for external processing (email, Teams, etc.)
    SELECT * FROM @AlertTable
    ORDER BY 
        CASE Severity 
            WHEN 'High' THEN 1 
            WHEN 'Medium' THEN 2 
            ELSE 3 
        END
END
GO
```

### Step 5: Configure Power BI Connection

Create a `config.json` file (do not commit to version control):

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectoryIntegrated",
    "connection_timeout": 30
  },
  "refresh_schedule": {
    "interval_minutes": 15,
    "enabled": true
  },
  "row_level_security": {
    "enabled": true,
    "role_table": "SecurityRoles"
  }
}
```

Use environment variables for sensitive data:

```sql
-- Create security table for Power BI RLS
CREATE TABLE SecurityRoles (
    UserEmail VARCHAR(200) PRIMARY KEY,
    RoleName VARCHAR(50) NOT NULL, -- Executive, Supervisor, User
    GeographyAccess VARCHAR(MAX), -- Comma-separated location IDs
    AllowAllGeographies BIT DEFAULT 0
)

-- Example: Insert roles
INSERT INTO SecurityRoles VALUES ('manager@company.com', 'Executive', NULL, 1)
INSERT INTO SecurityRoles VALUES ('supervisor@company.com', 'Supervisor', 'WARE001,WARE002', 0)
INSERT INTO SecurityRoles VALUES ('user@company.com', 'User', 'WARE001', 0)
GO
```

## Power BI Configuration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server name from `${SQL_SERVER_HOST}`
4. Select DirectQuery mode for real-time dashboards
5. Select tables: `FactWarehouseOperations`, `FactFleetTrips`, `FactCrossDock`, all `Dim*` tables, and views

### Create Relationships in Power BI Model

Power BI should auto-detect relationships, but verify:

```
DimTime (TimeKey) → FactWarehouseOperations (TimeKey) [Many-to-One]
DimTime (TimeKey) → FactFleetTrips (TimeKey) [Many-to-One]
DimGeography (GeographyKey) → FactWarehouseOperations (GeographyKey) [Many-to-One]
DimGeography (GeographyKey) → FactFleetTrips (OriginGeographyKey) [Many-to-One]
DimGeography (GeographyKey) → FactFleetTrips (DestinationGeographyKey) [Many-to-One]
DimProductGravity (ProductKey) → FactWarehouseOperations (ProductKey) [Many-to-One]
```

### Create DAX Measures for Key KPIs

```dax
// Total Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time
Avg Dwell Time (Hours) = AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// On-Time Delivery Rate
On-Time Delivery % = 
DIVIDE(
    CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeArrival] = TRUE()),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Cross-Fact: Dwell Impact on Fuel Efficiency
Dwell Fuel Impact = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgFuelEff = AVERAGE(FactFleetTrips[FuelEfficiencyKmPerLiter])
RETURN AvgDwell * AvgFuelEff

// Warehouse Gravity Zone Compliance
Gravity Compliance % = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[StorageZone] = RELATED(DimProductGravity[RecommendedZone])
    ),
    COUNTROWS(FactWarehouseOperations),
    0
) * 100

// Time Intelligence: MoM Growth
MoM Dwell Time Growth = 
VAR CurrentMonth = [Avg Dwell Time (Hours)]
VAR PreviousMonth = 
    CALCULATE(
        [Avg Dwell Time (Hours)],
        DATEADD(DimTime[Date], -1, MONTH)
    )
RETURN 
    DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth, 0) * 100
```

### Implement Row-Level Security in Power BI

1. Go to Modeling → Manage Roles
2. Create role "RestrictedUser"
3. Add DAX filter to `DimGeography`:

```dax
[LocationID] IN 
    SELECTCOLUMNS(
        FILTER(
            SecurityRoles,
            SecurityRoles[UserEmail] = USERPRINCIPALNAME()
        ),
        "Access", SecurityRoles[GeographyAccess]
    )
    || 
    LOOKUPVALUE(
        SecurityRoles[AllowAllGeographies],
        SecurityRoles[UserEmail], USERPRINCIPALNAME()
    ) = TRUE()
```

## Common Usage Patterns

### Pattern 1: Daily Warehouse Performance Report

```sql
-- Query: Get yesterday's warehouse performance by zone
SELECT 
    g.LocationName,
    wo.StorageZone,
    COUNT(DISTINCT wo.OperationKey) AS TotalOps,
    AVG(wo.DurationMinutes) AS AvgDurationMin,
    SUM(wo.QuantityUnits) AS TotalUnits,
    SUM(CASE WHEN wo.ErrorCount > 0 THEN 1 ELSE 0 END) AS ErrorOps,
    AVG(wo.DwellTimeHours) AS AvgDwellHours
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
WHERE t.Date = CAST(DATEADD(DAY, -1, GETDATE()) AS DATE)
GROUP BY g.LocationName, wo.StorageZone
ORDER BY TotalOps DESC
```

### Pattern 2: Fleet Fuel Efficiency Analysis

```sql
-- Query: Compare fuel efficiency by vehicle type and weather
SELECT 
    ft.VehicleID,
    ft.WeatherCondition,
    COUNT(ft.TripKey) AS TripCount,
    AVG(ft.DistanceKm) AS AvgDistanceKm,
    AVG(ft.FuelEfficiencyKmPerLiter) AS AvgFuelEfficiency,
    SUM(ft.TrafficDelayMinutes) AS TotalDelayMin
FROM FactFleetTrips ft
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY ft.VehicleID, ft.WeatherCondition
HAVING COUNT(ft.TripKey) >= 5
ORDER BY AvgFuelEfficiency DESC
```

### Pattern 3: Cross-Fact Analysis — Warehouse to Fleet Handoff

```sql
-- Query: Analyze time between warehouse shipping and fleet pickup
SELECT 
    g.LocationName AS Warehouse,
    p.ProductCategory,
    AVG(DATEDIFF(MINUTE, wo.CreatedAt, ft.CreatedAt)) AS AvgHandoffTimeMin,
    COUNT(DISTINCT wo.OperationKey) AS ShipmentCount,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin
FROM FactWarehouseOperations wo
INNER JOIN BridgeRoutesZones brz ON wo.OperationKey = brz.OperationKey
INNER JOIN FactFleetTrips ft ON brz.TripKey = ft.TripKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Shipping'
    AND brz.LinkType = 'Pickup'
    AND wo.CreatedAt >= DATEADD(DAY, -7, GETDATE())
GROUP BY g.LocationName, p.ProductCategory
HAVING COUNT(DISTINCT wo.OperationKey) >= 10
ORDER BY AvgHandoffTimeMin DESC
```

### Pattern 4: Predictive Bottleneck Detection

```sql
-- Query: Identify potential bottlenecks using historical patterns
WITH BottleneckMetrics AS (
    SELECT 
        g.LocationName,
        DATEPART(HOUR, t.DateTime) AS HourOfDay,
        DATEPART(WEEKDAY, t.DateTime) AS DayOfWeek,
        AVG(wo.DurationMinutes) AS AvgDuration,
        STDEV(wo.DurationMinutes) AS StdDevDuration,
        COUNT(wo.OperationKey) AS OperationVolume
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
    GROUP BY g.LocationName, DATEPART(HOUR, t.DateTime), DATEPART(WEEKDAY, t.DateTime)
)
SELECT 
    LocationName,
    HourOfDay,
    DayOfWeek,
    AvgDuration,
    StdDevDuration,
    OperationVolume,
    CASE 
        WHEN StdDevDuration > (AvgDuration * 0.5) AND OperationVolume > 100 THEN 'High Risk'
        WHEN StdDevDuration > (AvgDuration * 0.3) THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM BottleneckMetrics
WHERE OperationVolume >= 50
ORDER BY 
    CASE 
        WHEN StdDevDuration > (AvgDuration * 0.5) AND OperationVolume > 100 THEN 1
        WHEN StdDevDuration > (AvgDuration * 0.3) THEN 2
        ELSE 3
    END,
    OperationVolume DESC
```

## Automated Alert Configuration

### Set Up SQL Server Agent Job

```sql
-- Create scheduled job to run alert stored procedure every 15 minutes
USE msdb
GO

EXEC sp_add_job 
    @job_name = 'LogiFleet_KPI_Alerts',
    @enabled = 1,
    @description = 'Generate and send KPI alerts for logistics operations'
GO

EXEC sp_add_jobstep
    @job_name = 'LogiFleet_KPI_Alerts',
    @step_name = 'Run Alert SP',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC sp_GenerateKPIAlerts',
    @on_success_action = 1
GO

EXEC sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15,
    @active_start_time = 000000
GO

EXEC sp_attach_schedule
    @job_name = 'LogiFleet_KPI_Alerts',
    @schedule_name = 'Every15Minutes'
GO

EXEC sp_add_jobserver
    @job_name = 'LogiFleet_KPI_Alerts',
    @server_name = N'(local)'
GO
```

### Integrate Alerts with Email (Database Mail)

```sql
-- Configure Database Mail (run once)
EXEC sp_configure 'show advanced options', 1
RECONFIGURE
EXEC sp_configure 'Database Mail
