---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing solution for logistics, fleet management, and supply chain intelligence with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "create a logistics data warehouse with Power BI"
  - "implement fleet tracking and warehouse analytics"
  - "build multi-fact star schema for supply chain"
  - "configure LogiFleet Pulse KPI dashboards"
  - "integrate warehouse and fleet telemetry data"
  - "query cross-modal logistics data model"
  - "deploy supply chain intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics platform for logistics operations built on MS SQL Server and Power BI. It implements a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for cross-modal supply chain intelligence.

**Core capabilities:**
- Multi-fact star schema with time-phased dimensions
- Real-time warehouse and fleet KPI tracking
- Predictive bottleneck detection and fleet triage
- Cross-fact query harmonization (e.g., linking inventory dwell time to fleet fuel costs)
- Power BI dashboards with role-based access control
- Warehouse Gravity Zones™ for spatial optimization

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS, telemetry APIs, or flat files

### Step 1: Deploy SQL Schema

```sql
-- Clone the repository and navigate to SQL scripts
-- Execute the main schema deployment script

USE master;
GO

-- Create the database
CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = N'LogiFleetPulse_Data',
    FILENAME = N'C:\SQLData\LogiFleetPulse.mdf',
    SIZE = 1GB,
    FILEGROWTH = 256MB
)
LOG ON 
(
    NAME = N'LogiFleetPulse_Log',
    FILENAME = N'C:\SQLData\LogiFleetPulse_log.ldf',
    SIZE = 256MB,
    FILEGROWTH = 64MB
);
GO

USE LogiFleetPulse;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayOfMonth TINYINT NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    QuarterNumber TINYINT NOT NULL,
    YearNumber SMALLINT NOT NULL,
    FiscalPeriod VARCHAR(20),
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Customer Site'
    StreetAddress VARCHAR(500),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    RegionName VARCHAR(100),
    ContinentName VARCHAR(50),
    TimeZone VARCHAR(50)
);

-- Create product hierarchy with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500) NOT NULL,
    CategoryLevel1 VARCHAR(100),
    CategoryLevel2 VARCHAR(100),
    CategoryLevel3 VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsPerishable BIT NOT NULL DEFAULT 0,
    IsFragile BIT NOT NULL DEFAULT 0,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass VARCHAR(20), -- 'High', 'Medium', 'Low'
    OptimalZoneType VARCHAR(50) -- 'Dock-Adjacent', 'Mid-Level', 'Remote'
);

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    LeadTimeDays_Avg DECIMAL(6,2),
    LeadTimeDays_StdDev DECIMAL(6,2),
    DefectRate_Percent DECIMAL(5,2),
    OnTimeDeliveryRate_Percent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    RiskTier VARCHAR(20) -- 'Low', 'Medium', 'High'
);

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    BatchNumber VARCHAR(100),
    QuantityUnits INT NOT NULL,
    DwellTimeMinutes INT, -- Time between receiving and shipping
    ProcessingTimeMinutes INT,
    OperatorID VARCHAR(50),
    ZoneID VARCHAR(50),
    ZoneGravityScore DECIMAL(5,2),
    ErrorFlag BIT NOT NULL DEFAULT 0,
    ErrorType VARCHAR(100),
    CostUSD DECIMAL(12,2)
);

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey_Start INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKey_End INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey_Origin INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    GeographyKey_Destination INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    RouteID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    SpeedAvg_KPH DECIMAL(6,2),
    SpeedMax_KPH DECIMAL(6,2),
    HarshBrakingEvents INT,
    HarshAccelerationEvents INT,
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    MaintenanceFlag BIT NOT NULL DEFAULT 0,
    CostUSD DECIMAL(12,2)
);

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey_Inbound INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TimeKey_Outbound INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey_Facility INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    QuantityUnits INT NOT NULL,
    DockDwellMinutes INT,
    TransferTimeMinutes INT,
    CostUSD DECIMAL(12,2)
);

-- Create indexes for optimal query performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Start ON FactFleetTrips(TimeKey_Start);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(RouteID);
```

### Step 2: Populate Time Dimension

```sql
-- Generate time dimension data for 2 years at 15-minute intervals
DECLARE @StartDate DATETIME2 = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2026-12-31 23:45:00';
DECLARE @CurrentTime DATETIME2 = @StartDate;
DECLARE @TimeKey INT = 1;

WHILE @CurrentTime <= @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey, FullDateTime, DateKey, TimeOfDay, 
        HourOfDay, DayOfWeek, DayOfMonth, WeekOfYear,
        MonthNumber, QuarterNumber, YearNumber,
        IsWeekend, IsHoliday
    )
    VALUES (
        @TimeKey,
        @CurrentTime,
        CAST(FORMAT(@CurrentTime, 'yyyyMMdd') AS INT),
        CAST(@CurrentTime AS TIME),
        DATEPART(HOUR, @CurrentTime),
        DATEPART(WEEKDAY, @CurrentTime),
        DATEPART(DAY, @CurrentTime),
        DATEPART(WEEK, @CurrentTime),
        DATEPART(MONTH, @CurrentTime),
        DATEPART(QUARTER, @CurrentTime),
        DATEPART(YEAR, @CurrentTime),
        CASE WHEN DATEPART(WEEKDAY, @CurrentTime) IN (1,7) THEN 1 ELSE 0 END,
        0 -- Holiday flag - update separately with holiday list
    );
    
    SET @CurrentTime = DATEADD(MINUTE, 15, @CurrentTime);
    SET @TimeKey = @TimeKey + 1;
END;
```

### Step 3: Create Data Loading Stored Procedures

```sql
-- Stored procedure for incremental warehouse operations loading
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadDateTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last load time if not specified
    IF @LoadDateTime IS NULL
        SET @LoadDateTime = ISNULL(
            (SELECT MAX(FullDateTime) FROM DimTime dt 
             INNER JOIN FactWarehouseOperations f ON dt.TimeKey = f.TimeKey),
            '1900-01-01'
        );
    
    -- Insert from staging or external source
    -- This is a template - adjust to your WMS data source
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, BatchNumber, QuantityUnits,
        DwellTimeMinutes, ProcessingTimeMinutes,
        OperatorID, ZoneID, ZoneGravityScore,
        ErrorFlag, ErrorType, CostUSD
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        ds.SupplierKey,
        src.OperationType,
        src.BatchNumber,
        src.Quantity,
        DATEDIFF(MINUTE, src.ReceiveTime, src.ShipTime),
        DATEDIFF(MINUTE, src.StartTime, src.EndTime),
        src.OperatorID,
        src.ZoneID,
        dp.GravityScore,
        src.ErrorFlag,
        src.ErrorType,
        src.CostUSD
    FROM StagingWarehouseOps src
    INNER JOIN DimTime dt ON dt.FullDateTime = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '1900-01-01', src.OperationTime) / 15) * 15, '1900-01-01')
    INNER JOIN DimGeography dg ON dg.LocationID = src.WarehouseID
    INNER JOIN DimProductGravity dp ON dp.SKU = src.SKU
    LEFT JOIN DimSupplierReliability ds ON ds.SupplierID = src.SupplierID
    WHERE src.OperationTime > @LoadDateTime;
    
    PRINT 'Warehouse operations loaded: ' + CAST(@@ROWCOUNT AS VARCHAR(20));
END;
GO

-- Stored procedure for fleet trip loading
CREATE PROCEDURE usp_LoadFleetTrips
    @LoadDateTime DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    IF @LoadDateTime IS NULL
        SET @LoadDateTime = ISNULL(
            (SELECT MAX(dt.FullDateTime) FROM DimTime dt 
             INNER JOIN FactFleetTrips f ON dt.TimeKey = f.TimeKey_Start),
            '1900-01-01'
        );
    
    INSERT INTO FactFleetTrips (
        TimeKey_Start, TimeKey_End,
        GeographyKey_Origin, GeographyKey_Destination,
        VehicleID, DriverID, RouteID,
        DistanceKM, DurationMinutes, IdleTimeMinutes,
        FuelConsumedLiters, LoadWeightKG,
        SpeedAvg_KPH, SpeedMax_KPH,
        HarshBrakingEvents, HarshAccelerationEvents,
        WeatherCondition, TrafficDelayMinutes,
        MaintenanceFlag, CostUSD
    )
    SELECT 
        dt_start.TimeKey,
        dt_end.TimeKey,
        dg_origin.GeographyKey,
        dg_dest.GeographyKey,
        src.VehicleID,
        src.DriverID,
        src.RouteID,
        src.DistanceKM,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime),
        src.IdleTimeMinutes,
        src.FuelLiters,
        src.LoadWeightKG,
        src.AvgSpeedKPH,
        src.MaxSpeedKPH,
        src.HarshBrakingCount,
        src.HarshAccelCount,
        src.WeatherCondition,
        src.TrafficDelayMinutes,
        CASE WHEN src.MaintenanceAlertFlag = 1 THEN 1 ELSE 0 END,
        src.TripCostUSD
    FROM StagingFleetTrips src
    INNER JOIN DimTime dt_start ON dt_start.FullDateTime = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '1900-01-01', src.StartTime) / 15) * 15, '1900-01-01')
    INNER JOIN DimTime dt_end ON dt_end.FullDateTime = DATEADD(MINUTE, 
        (DATEDIFF(MINUTE, '1900-01-01', src.EndTime) / 15) * 15, '1900-01-01')
    INNER JOIN DimGeography dg_origin ON dg_origin.LocationID = src.OriginLocationID
    INNER JOIN DimGeography dg_dest ON dg_dest.LocationID = src.DestinationLocationID
    WHERE src.StartTime > @LoadDateTime;
    
    PRINT 'Fleet trips loaded: ' + CAST(@@ROWCOUNT AS VARCHAR(20));
END;
GO
```

### Step 4: Calculate Product Gravity Scores

```sql
-- Update gravity scores based on velocity, value, and fragility
CREATE PROCEDURE usp_CalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate velocity class from recent operations
    WITH VelocityCalc AS (
        SELECT 
            ProductKey,
            COUNT(*) as OperationCount,
            SUM(QuantityUnits) as TotalUnits,
            AVG(DwellTimeMinutes) as AvgDwellTime
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT TimeKey FROM DimTime 
                         WHERE FullDateTime >= DATEADD(DAY, -90, GETDATE()))
        GROUP BY ProductKey
    )
    UPDATE dp
    SET 
        VelocityClass = CASE 
            WHEN vc.OperationCount >= 100 THEN 'Fast'
            WHEN vc.OperationCount >= 20 THEN 'Medium'
            ELSE 'Slow'
        END,
        GravityScore = (
            -- Velocity component (0-50 points)
            CASE 
                WHEN vc.OperationCount >= 100 THEN 50
                WHEN vc.OperationCount >= 20 THEN 30
                ELSE 10
            END +
            -- Value component (0-30 points) - requires cost data
            20 + -- Placeholder, integrate with pricing data
            -- Fragility component (0-20 points)
            CASE 
                WHEN dp.IsFragile = 1 THEN 20
                WHEN dp.IsPerishable = 1 THEN 15
                ELSE 5
            END
        ) / 100.0 * 100, -- Normalize to 0-100
        OptimalZoneType = CASE
            WHEN vc.OperationCount >= 100 AND dp.IsFragile = 1 THEN 'Dock-Adjacent'
            WHEN vc.OperationCount >= 100 THEN 'Mid-Level-Priority'
            WHEN vc.OperationCount < 20 THEN 'Remote'
            ELSE 'Mid-Level'
        END
    FROM DimProductGravity dp
    INNER JOIN VelocityCalc vc ON dp.ProductKey = vc.ProductKey;
    
    PRINT 'Product gravity scores updated';
END;
GO
```

## Power BI Configuration

### Import Data Model

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Enter server and database name
4. Select all dimension and fact tables
5. Power BI will auto-detect relationships based on foreign keys

### Create Calculated Measures

```dax
// Total Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time (hours)
Avg Dwell Time Hours = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 60

// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
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

// Cross-Fact KPI: Dwell Time Impact on Fleet Cost
Dwell Impact on Fleet Cost = 
VAR HighDwellSKUs = 
    CALCULATETABLE(
        VALUES(DimProductGravity[SKU]),
        FactWarehouseOperations[DwellTimeMinutes] > 4320 // 3 days
    )
VAR FleetCostForHighDwell = 
    CALCULATE(
        SUM(FactFleetTrips[CostUSD]),
        TREATAS(HighDwellSKUs, DimProductGravity[SKU])
    )
RETURN FleetCostForHighDwell

// Predictive Bottleneck Score (simplified)
Bottleneck Risk Score = 
VAR CurrentUtilization = [Fleet Utilization %]
VAR AvgDwellTime = [Avg Dwell Time Hours]
VAR ErrorRate = DIVIDE(
    CALCULATE(COUNTROWS(FactWarehouseOperations), FactWarehouseOperations[ErrorFlag] = TRUE()),
    [Total Operations]
)
RETURN 
    (CurrentUtilization * 0.4) + 
    (AvgDwellTime * 5 * 0.3) + 
    (ErrorRate * 100 * 0.3)

// Time Intelligence - Prior Period Comparison
Operations Prior Period = 
CALCULATE(
    [Total Operations],
    DATEADD(DimTime[FullDateTime], -1, MONTH)
)

Operations Growth % = 
DIVIDE(
    [Total Operations] - [Operations Prior Period],
    [Operations Prior Period],
    0
) * 100
```

### Row-Level Security

```dax
// Create security table mapping users to regions
// In Power BI: Modeling → Manage Roles → Create Role

[UserRegionAccess] = 
VAR CurrentUser = USERPRINCIPALNAME()
VAR UserRegions = 
    CALCULATETABLE(
        VALUES(SecurityUserRegions[RegionName]),
        SecurityUserRegions[UserEmail] = CurrentUser
    )
RETURN
    DimGeography[RegionName] IN UserRegions
```

## Key Queries & Analysis Patterns

### Cross-Fact Analysis: High Dwell Items Affecting Fleet Routes

```sql
-- Find products with high warehouse dwell time that also appear in delayed fleet trips
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.VelocityClass,
    AVG(fwo.DwellTimeMinutes) / 60.0 AS AvgDwellHours,
    COUNT(DISTINCT fft.TripKey) AS AffectedTrips,
    AVG(fft.TrafficDelayMinutes) AS AvgTrafficDelay,
    SUM(fft.CostUSD) AS TotalFleetCostUSD
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN FactFleetTrips fft ON fft.TimeKey_Start BETWEEN 
    fwo.TimeKey AND DATEADD(HOUR, 24, fwo.TimeKey)
WHERE fwo.DwellTimeMinutes > 2880 -- More than 2 days
    AND fft.TrafficDelayMinutes > 30
GROUP BY dp.SKU, dp.ProductName, dp.VelocityClass
HAVING COUNT(DISTINCT fft.TripKey) > 5
ORDER BY TotalFleetCostUSD DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend zone reassignments based on gravity score mismatch
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.OptimalZoneType,
    fwo.ZoneID AS CurrentZone,
    AVG(fwo.ProcessingTimeMinutes) AS AvgProcessingTime,
    COUNT(*) AS OperationCount,
    CASE 
        WHEN dp.OptimalZoneType = 'Dock-Adjacent' AND fwo.ZoneID NOT LIKE 'DA%' 
            THEN 'MOVE TO DOCK-ADJACENT'
        WHEN dp.OptimalZoneType = 'Remote' AND fwo.ZoneID LIKE 'DA%' 
            THEN 'MOVE TO REMOTE'
        ELSE 'OK'
    END AS RecommendedAction
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE fwo.TimeKey >= (SELECT TimeKey FROM DimTime 
                      WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.OptimalZoneType, fwo.ZoneID
HAVING RecommendedAction != 'OK'
ORDER BY dp.GravityScore DESC;
```

### Fleet Predictive Maintenance Queue

```sql
-- Generate prioritized maintenance list based on telemetry + revenue impact
WITH VehicleMetrics AS (
    SELECT 
        VehicleID,
        AVG(HarshBrakingEvents + HarshAccelerationEvents) AS AvgRoughHandling,
        MAX(SpeedMax_KPH) AS MaxSpeedRecorded,
        SUM(DistanceKM) AS TotalDistanceKM,
        SUM(IdleTimeMinutes) AS TotalIdleMinutes,
        COUNT(*) AS TripCount
    FROM FactFleetTrips
    WHERE TimeKey_Start >= (SELECT TimeKey FROM DimTime 
                           WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
    GROUP BY VehicleID
),
VehicleRevenue AS (
    SELECT 
        fft.VehicleID,
        SUM(fwo.QuantityUnits * 100) AS EstimatedRevenueUSD -- Placeholder: $100 per unit
    FROM FactFleetTrips fft
    INNER JOIN FactCrossDock fcd ON fft.TripKey = fcd.OutboundTripKey
    INNER JOIN FactWarehouseOperations fwo ON fcd.ProductKey = fwo.ProductKey
    WHERE fft.TimeKey_Start >= (SELECT TimeKey FROM DimTime 
                               WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
    GROUP BY fft.VehicleID
)
SELECT 
    vm.VehicleID,
    vm.TotalDistanceKM,
    vm.AvgRoughHandling,
    vm.TotalIdleMinutes,
    vr.EstimatedRevenueUSD,
    -- Priority score: rough handling + distance + idle time, weighted by revenue
    (
        (vm.AvgRoughHandling * 10) + 
        (vm.TotalDistanceKM / 1000) + 
        (vm.TotalIdleMinutes / 60)
    ) * (vr.EstimatedRevenueUSD / 10000.0) AS MaintenancePriorityScore,
    CASE 
        WHEN vm.AvgRoughHandling > 5 THEN 'CRITICAL - DRIVER TRAINING'
        WHEN vm.TotalDistanceKM > 10000 THEN 'SCHEDULED MAINTENANCE DUE'
        WHEN vm.TotalIdleMinutes > 300 THEN 'IDLE OPTIMIZATION NEEDED'
        ELSE 'MONITOR'
    END AS RecommendedAction
FROM VehicleMetrics vm
LEFT JOIN VehicleRevenue vr ON vm.VehicleID = vr.VehicleID
ORDER BY MaintenancePriorityScore DESC;
```

### Time-Phased Simulation Query

```sql
-- Simulate impact of 95% warehouse capacity vs. current utilization
DECLARE @CurrentCapacity DECIMAL(5,2) = 0.80; -- 80%
DECLARE @SimulatedCapacity DECIMAL(5,2) = 0.95; -- 95%

WITH CurrentPerformance AS (
    SELECT 
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
        AVG(fwo.ProcessingTimeMinutes) AS AvgProcessingTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations fwo
    WHERE fwo.TimeKey >= (SELECT TimeKey FROM DimTime 
                         WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()))
),
CapacityImpactModel AS (
    SELECT 
        AvgDwellTime * (@SimulatedCapacity / @CurrentCapacity) AS ProjectedDwellTime,
        AvgProcessingTime * (@SimulatedCapacity / @CurrentCapacity) AS ProjectedProcessingTime,
        OperationCount AS CurrentOps,
        OperationCount * (1 + (@SimulatedCapacity - @CurrentCapacity)) AS ProjectedOps
    FROM CurrentPerformance
)
SELECT 
    cp.AvgDwellTime AS Current_AvgDwellTime_Min,
    cim.ProjectedDwellTime AS Simulated_AvgDwellTime_Min,
    cp.AvgProcessingTime AS Current_AvgProcessingTime_Min,
    cim.ProjectedProcessingTime AS Simulated_AvgProcessingTime_Min,
    (cim.ProjectedDwellTime - cp.AvgDwellTime) AS DwellTime_DeltaMinutes,
    CASE 
        WHEN cim.ProjectedDwellTime > cp.AvgDwellTime * 1.2 
            THEN 'WARN: Dwell time will increase >20%'
        ELSE 'Acceptable impact'
    END AS SimulationResult
FROM CurrentPerformance cp
CROSS JOIN CapacityImpactModel cim;
```

## Configuration & Environment Variables

### Database Connection

Store connection strings securely using environment variables:

```bash
# .env file (never commit to version control)
LOGIFLEET_SQL_SERVER=your-server.database.windows.net
LOGIFLEET_SQL_DATABASE=LogiFleetPulse
LOGIFLEET_SQL_USER=sqluser
LOGIFLEET_SQL_PASSWORD=your_secure_password_here
LOGIFLEET_SQL_ENCRYPT=true
```

### Power BI Connection Parameters

```powerquery
// In Power BI Advanced Editor
let
    Server = Environment.GetEnvironmentVariable("LOGIFLEET_SQL_SERVER"),
    Database = Environment.GetEnvironmentVariable("LOGIFLEET_SQL_DATABASE"),
    Source = Sql.Database(Server, Database)
in
    Source
```

### External API Integration (Weather, Traffic)

```sql
-- Example: Create external table for weather data (requires PolyBase)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = 'https://api.weatherprovider.com',
    CREDENTIAL = WeatherAPICredential
);

-- Reference in queries
SELECT 
    fft.TripKey,
    fft.RouteID,
    weather.Condition,
    weather.Temperature,
    fft.TrafficDelayMinutes
FROM FactFleetTrips fft
CROSS APPLY OPENJSON(
    (SELECT * FROM OPENROWSET(BULK 'weather/history?date=' + CAST(dt.
