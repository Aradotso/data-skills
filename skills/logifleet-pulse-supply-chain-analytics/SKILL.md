---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing solution for real-time logistics intelligence, fleet optimization, and warehouse analytics
triggers:
  - "set up logifleet pulse logistics dashboard"
  - "configure supply chain analytics warehouse"
  - "deploy logifleet sql schema"
  - "create power bi logistics visualization"
  - "implement fleet telemetry tracking"
  - "build warehouse gravity zone optimization"
  - "query cross-fact logistics KPIs"
  - "set up real-time supply chain monitoring"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing solution for logistics intelligence. It provides a multi-modal analytics engine that fuses warehouse operations, fleet telemetry, inventory management, and external data signals into a unified semantic layer with multi-fact star schema architecture.

## What It Does

- **Unified Data Model**: Multi-fact star schema linking warehouse operations, fleet trips, cross-dock activities, and supply chain metrics
- **Real-Time Dashboards**: Power BI dashboards with 15-minute refresh cycles for operational awareness
- **Cross-Fact KPI Analysis**: Harmonized metrics across warehouse, fleet, and supply chain dimensions
- **Predictive Analytics**: Bottleneck detection, fleet triage, and temporal elasticity modeling
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item value, and replenishment lead time
- **Role-Based Security**: Row-level security for multi-tenant and hierarchical access control

## Installation

### Prerequisites

- MS SQL Server 2019+ (2022 recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Source data connections (WMS, TMS, telematics APIs)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Execute the schema deployment script

USE master;
GO

-- Create database
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
    FILENAME = N'C:\SQLData\LogiFleetPulse_Log.ldf',
    SIZE = 512MB,
    FILEGROWTH = 128MB
);
GO

USE LogiFleetPulse;
GO

-- Create schemas
CREATE SCHEMA fact AUTHORIZATION dbo;
CREATE SCHEMA dim AUTHORIZATION dbo;
CREATE SCHEMA bridge AUTHORIZATION dbo;
CREATE SCHEMA staging AUTHORIZATION dbo;
GO
```

### Step 2: Create Dimension Tables

```sql
-- DimTime: Time-phased dimension with 15-minute granularity
CREATE TABLE dim.DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName NVARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    FiscalPeriod INT NOT NULL,
    FiscalQuarter TINYINT NOT NULL,
    FiscalYear INT NOT NULL
);

CREATE NONCLUSTERED INDEX IX_DimTime_DateTime ON dim.DimTime(FullDateTime);
CREATE NONCLUSTERED INDEX IX_DimTime_Date ON dim.DimTime(DateKey);
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE dim.DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationCode NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, Distribution Center, Route Node
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100) NOT NULL,
    Region NVARCHAR(100) NOT NULL,
    Continent NVARCHAR(50) NOT NULL,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE
);

CREATE NONCLUSTERED INDEX IX_DimGeography_Location ON dim.DimGeography(LocationCode, IsActive);
GO

-- DimProduct: Product hierarchy with gravity scoring
CREATE TABLE dim.DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    ProductCategory NVARCHAR(100) NOT NULL,
    ProductSubcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2), -- kg
    UnitVolume DECIMAL(10,2), -- cubic meters
    IsFragile BIT NOT NULL DEFAULT 0,
    RequiresColdStorage BIT NOT NULL DEFAULT 0,
    ShelfLifeDays INT,
    UnitValue DECIMAL(12,2), -- currency
    GravityScore DECIMAL(5,2), -- Calculated: velocity × value × fragility
    CurrentGravityZone NVARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_DimProduct_SKU ON dim.DimProduct(SKU, IsActive);
CREATE NONCLUSTERED INDEX IX_DimProduct_Gravity ON dim.DimProduct(GravityScore DESC) INCLUDE (CurrentGravityZone);
GO

-- DimVehicle: Fleet dimension
CREATE TABLE dim.DimVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID NVARCHAR(50) NOT NULL UNIQUE,
    VehicleType NVARCHAR(50) NOT NULL, -- Truck, Van, Trailer
    Make NVARCHAR(50),
    Model NVARCHAR(50),
    Year INT,
    Capacity DECIMAL(10,2), -- cubic meters
    MaxPayload DECIMAL(10,2), -- kg
    FuelType NVARCHAR(50),
    HasRefrigeration BIT NOT NULL DEFAULT 0,
    HomeLocationKey INT FOREIGN KEY REFERENCES dim.DimGeography(GeographyKey),
    AcquisitionDate DATE,
    IsActive BIT NOT NULL DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_DimVehicle_ID ON dim.DimVehicle(VehicleID, IsActive);
GO
```

### Step 3: Create Fact Tables

```sql
-- FactWarehouseOperations: Warehouse activity grain
CREATE TABLE fact.FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES dim.DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES dim.DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES dim.DimProduct(ProductKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID NVARCHAR(50),
    QuantityHandled INT NOT NULL,
    CycleTimeSeconds INT, -- Time to complete operation
    DwellTimeHours DECIMAL(10,2), -- Time in storage before movement
    StorageZone NVARCHAR(50),
    OperatorID NVARCHAR(50),
    ErrorFlag BIT NOT NULL DEFAULT 0,
    ErrorReason NVARCHAR(500)
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON fact.FactWarehouseOperations(TimeKey) INCLUDE (WarehouseKey, ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON fact.FactWarehouseOperations(ProductKey, TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Type ON fact.FactWarehouseOperations(OperationType, TimeKey);
GO

-- FactFleetTrips: Fleet telemetry and trip segments
CREATE TABLE fact.FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKeyStart INT NOT NULL FOREIGN KEY REFERENCES dim.DimTime(TimeKey),
    TimeKeyEnd INT FOREIGN KEY REFERENCES dim.DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES dim.DimVehicle(VehicleKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES dim.DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES dim.DimGeography(GeographyKey),
    TripID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    AverageSpeed DECIMAL(5,2),
    MaxSpeed DECIMAL(5,2),
    LoadWeight DECIMAL(10,2),
    LoadVolume DECIMAL(10,2),
    DelayMinutes INT DEFAULT 0,
    DelayReason NVARCHAR(200),
    OnTimeFlag BIT
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON fact.FactFleetTrips(TimeKeyStart) INCLUDE (VehicleKey, OriginKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON fact.FactFleetTrips(VehicleKey, TimeKeyStart);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON fact.FactFleetTrips(OriginKey, DestinationKey, TimeKeyStart);
GO

-- FactCrossDock: Cross-dock transfers
CREATE TABLE fact.FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKeyReceived INT NOT NULL FOREIGN KEY REFERENCES dim.DimTime(TimeKey),
    TimeKeyShipped INT FOREIGN KEY REFERENCES dim.DimTime(TimeKey),
    FacilityKey INT NOT NULL FOREIGN KEY REFERENCES dim.DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES dim.DimProduct(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES fact.FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES fact.FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    DwellTimeMinutes INT, -- Time between inbound and outbound
    TargetDwellMinutes INT,
    ExceededTargetFlag BIT
);

CREATE NONCLUSTERED INDEX IX_FactCrossDock_Time ON fact.FactCrossDock(TimeKeyReceived) INCLUDE (ProductKey, FacilityKey);
GO
```

### Step 4: Create Views for Cross-Fact Analysis

```sql
-- View: Unified logistics performance
CREATE VIEW fact.vw_UnifiedLogisticsPerformance AS
SELECT 
    dt.FullDateTime,
    dt.DateKey,
    dt.FiscalQuarter,
    dg.LocationName AS Warehouse,
    dg.Region,
    dp.ProductCategory,
    dp.GravityScore,
    
    -- Warehouse metrics
    SUM(CASE WHEN fwo.OperationType = 'Picking' THEN 1 ELSE 0 END) AS PickCount,
    AVG(CASE WHEN fwo.OperationType = 'Picking' THEN fwo.CycleTimeSeconds ELSE NULL END) AS AvgPickTime,
    AVG(fwo.DwellTimeHours) AS AvgDwellHours,
    
    -- Fleet metrics (joined via cross-dock or direct shipment)
    COUNT(DISTINCT fft.TripKey) AS TripCount,
    SUM(fft.DistanceKm) AS TotalDistanceKm,
    AVG(fft.IdleTimeMinutes) AS AvgIdleMinutes,
    SUM(fft.FuelConsumedLiters) AS TotalFuelLiters,
    
    -- Cross-dock efficiency
    AVG(fcd.DwellTimeMinutes) AS AvgCrossDockDwellMinutes,
    SUM(CAST(fcd.ExceededTargetFlag AS INT)) AS CrossDockDelays

FROM fact.FactWarehouseOperations fwo
INNER JOIN dim.DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN dim.DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
INNER JOIN dim.DimProduct dp ON fwo.ProductKey = dp.ProductKey
LEFT JOIN fact.FactCrossDock fcd ON fwo.ProductKey = fcd.ProductKey 
    AND ABS(DATEDIFF(MINUTE, dt.FullDateTime, (SELECT FullDateTime FROM dim.DimTime WHERE TimeKey = fcd.TimeKeyReceived))) < 60
LEFT JOIN fact.FactFleetTrips fft ON fcd.OutboundTripKey = fft.TripKey

GROUP BY 
    dt.FullDateTime,
    dt.DateKey,
    dt.FiscalQuarter,
    dg.LocationName,
    dg.Region,
    dp.ProductCategory,
    dp.GravityScore;
GO
```

### Step 5: Create Stored Procedures for Data Loading

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE staging.usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assumes staging.WarehouseOperations_Staging table exists with source data
    INSERT INTO fact.FactWarehouseOperations (
        TimeKey,
        WarehouseKey,
        ProductKey,
        OperationType,
        OrderID,
        QuantityHandled,
        CycleTimeSeconds,
        DwellTimeHours,
        StorageZone,
        OperatorID,
        ErrorFlag,
        ErrorReason
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        s.OperationType,
        s.OrderID,
        s.Quantity,
        s.CycleTimeSeconds,
        s.DwellTimeHours,
        s.StorageZone,
        s.OperatorID,
        s.ErrorFlag,
        s.ErrorReason
    FROM staging.WarehouseOperations_Staging s
    INNER JOIN dim.DimTime dt ON dt.FullDateTime = DATEADD(MINUTE, 
        (DATEPART(MINUTE, s.OperationDateTime) / 15) * 15 - DATEPART(MINUTE, s.OperationDateTime),
        s.OperationDateTime)
    INNER JOIN dim.DimGeography dg ON dg.LocationCode = s.WarehouseCode AND dg.IsActive = 1
    INNER JOIN dim.DimProduct dp ON dp.SKU = s.SKU AND dp.IsActive = 1
    WHERE s.OperationDateTime > @LastLoadDateTime
        AND s.LoadedFlag = 0;
    
    UPDATE staging.WarehouseOperations_Staging
    SET LoadedFlag = 1
    WHERE OperationDateTime > @LastLoadDateTime;
    
    RETURN @@ROWCOUNT;
END;
GO

-- Gravity score calculation procedure
CREATE PROCEDURE dim.usp_CalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(CycleTimeSeconds) AS AvgCycleTime
        FROM fact.FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT TOP 1 TimeKey FROM dim.DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY TimeKey)
        GROUP BY ProductKey
    )
    UPDATE dp
    SET GravityScore = (
        (ISNULL(pv.PickFrequency, 0) * 0.5) + -- Velocity weight
        ((dp.UnitValue / NULLIF((SELECT MAX(UnitValue) FROM dim.DimProduct), 0)) * 100 * 0.3) + -- Value weight
        (CASE WHEN dp.IsFragile = 1 THEN 20 ELSE 0 END) -- Fragility weight
    ),
    CurrentGravityZone = CASE 
        WHEN (
            (ISNULL(pv.PickFrequency, 0) * 0.5) + 
            ((dp.UnitValue / NULLIF((SELECT MAX(UnitValue) FROM dim.DimProduct), 0)) * 100 * 0.3) + 
            (CASE WHEN dp.IsFragile = 1 THEN 20 ELSE 0 END)
        ) >= 70 THEN 'High-Gravity'
        WHEN (
            (ISNULL(pv.PickFrequency, 0) * 0.5) + 
            ((dp.UnitValue / NULLIF((SELECT MAX(UnitValue) FROM dim.DimProduct), 0)) * 100 * 0.3) + 
            (CASE WHEN dp.IsFragile = 1 THEN 20 ELSE 0 END)
        ) >= 40 THEN 'Medium-Gravity'
        ELSE 'Low-Gravity'
    END
    FROM dim.DimProduct dp
    LEFT JOIN ProductVelocity pv ON dp.ProductKey = pv.ProductKey
    WHERE dp.IsActive = 1;
END;
GO
```

## Configuration

### Connection String Setup

Store connection strings in environment variables or secure configuration:

```sql
-- For Azure SQL Database
Server=${LOGIFLEET_SQL_SERVER};Database=LogiFleetPulse;Authentication=Active Directory Integrated;Encrypt=True;

-- For on-premises SQL Server with Windows Auth
Server=${LOGIFLEET_SQL_SERVER};Database=LogiFleetPulse;Integrated Security=True;

-- For SQL authentication (not recommended for production)
Server=${LOGIFLEET_SQL_SERVER};Database=LogiFleetPulse;User ID=${LOGIFLEET_SQL_USER};Password=${LOGIFLEET_SQL_PASSWORD};
```

### Power BI Configuration

1. Open `LogiFleet_Pulse_Master.pbit` template
2. Enter SQL Server connection parameters:
   - Server: `${LOGIFLEET_SQL_SERVER}`
   - Database: `LogiFleetPulse`
3. Configure data refresh schedule (15-minute recommended):
   - Power BI Service → Dataset Settings → Scheduled Refresh
   - Gateway: Use on-premises data gateway for local SQL Server

### Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE security.UserAccess (
    UserEmail NVARCHAR(255) PRIMARY KEY,
    Region NVARCHAR(100),
    AccessLevel NVARCHAR(50) -- 'User', 'Supervisor', 'Executive'
);

-- RLS filter in Power BI (DAX):
-- Create Role "RegionalAccess"
-- Filter on dim.DimGeography table:
[Region] = LOOKUPVALUE(
    security[UserAccess][Region],
    security[UserAccess][UserEmail],
    USERPRINCIPALNAME()
)
```

## Key API Patterns

### Querying Cross-Fact KPIs

```sql
-- Correlation: Dwell time vs. fleet idle time
SELECT 
    dp.ProductCategory,
    dp.GravityScore,
    AVG(fwo.DwellTimeHours) AS AvgWarehouseDwellHours,
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    CORR(fwo.DwellTimeHours, fft.IdleTimeMinutes) OVER (PARTITION BY dp.ProductCategory) AS CorrelationCoefficient
FROM fact.FactWarehouseOperations fwo
INNER JOIN dim.DimProduct dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN fact.FactCrossDock fcd ON fwo.ProductKey = fcd.ProductKey
INNER JOIN fact.FactFleetTrips fft ON fcd.OutboundTripKey = fft.TripKey
WHERE fwo.TimeKey >= (SELECT TOP 1 TimeKey FROM dim.DimTime WHERE FullDateTime >= DATEADD(DAY, -90, GETDATE()) ORDER BY TimeKey)
GROUP BY dp.ProductCategory, dp.GravityScore;
```

### Warehouse Gravity Zone Analysis

```sql
-- Products recommended for zone reassignment
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.CurrentGravityZone,
    CASE 
        WHEN dp.GravityScore >= 70 THEN 'High-Gravity'
        WHEN dp.GravityScore >= 40 THEN 'Medium-Gravity'
        ELSE 'Low-Gravity'
    END AS RecommendedZone,
    dp.GravityScore,
    COUNT(fwo.OperationKey) AS Recent30DayPicks
FROM dim.DimProduct dp
LEFT JOIN fact.FactWarehouseOperations fwo ON dp.ProductKey = fwo.ProductKey
    AND fwo.OperationType = 'Picking'
    AND fwo.TimeKey >= (SELECT TOP 1 TimeKey FROM dim.DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY TimeKey)
WHERE dp.IsActive = 1
    AND dp.CurrentGravityZone != CASE 
        WHEN dp.GravityScore >= 70 THEN 'High-Gravity'
        WHEN dp.GravityScore >= 40 THEN 'Medium-Gravity'
        ELSE 'Low-Gravity'
    END
GROUP BY dp.SKU, dp.ProductName, dp.CurrentGravityZone, dp.GravityScore
HAVING COUNT(fwo.OperationKey) > 10 -- Minimum activity threshold
ORDER BY dp.GravityScore DESC;
```

### Fleet Triage Scoring

```sql
-- Proactive maintenance priority queue
WITH FleetHealth AS (
    SELECT 
        dv.VehicleID,
        dv.VehicleType,
        dv.Make,
        dv.Model,
        AVG(fft.FuelConsumedLiters / NULLIF(fft.DistanceKm, 0)) AS AvgFuelEfficiency,
        SUM(fft.IdleTimeMinutes) AS TotalIdleMinutes,
        SUM(fft.DelayMinutes) AS TotalDelayMinutes,
        COUNT(fft.TripKey) AS TripCount
    FROM dim.DimVehicle dv
    INNER JOIN fact.FactFleetTrips fft ON dv.VehicleKey = fft.VehicleKey
    WHERE fft.TimeKeyStart >= (SELECT TOP 1 TimeKey FROM dim.DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY TimeKey)
    GROUP BY dv.VehicleID, dv.VehicleType, dv.Make, dv.Model
),
FleetRevenue AS (
    SELECT 
        dv.VehicleKey,
        SUM(dp.UnitValue * fwo.QuantityHandled) AS AssociatedRevenue
    FROM dim.DimVehicle dv
    INNER JOIN fact.FactFleetTrips fft ON dv.VehicleKey = fft.VehicleKey
    INNER JOIN fact.FactCrossDock fcd ON fft.TripKey = fcd.OutboundTripKey
    INNER JOIN fact.FactWarehouseOperations fwo ON fcd.ProductKey = fwo.ProductKey
    INNER JOIN dim.DimProduct dp ON fwo.ProductKey = dp.ProductKey
    WHERE fft.TimeKeyStart >= (SELECT TOP 1 TimeKey FROM dim.DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY TimeKey)
    GROUP BY dv.VehicleKey
)
SELECT 
    fh.VehicleID,
    fh.VehicleType,
    fh.TripCount,
    fh.TotalIdleMinutes,
    fh.TotalDelayMinutes,
    fh.AvgFuelEfficiency,
    ISNULL(fr.AssociatedRevenue, 0) AS RevenueAtRisk,
    -- Triage Score: Higher = More urgent
    (
        (fh.TotalIdleMinutes / NULLIF(fh.TripCount, 0) * 0.2) + -- Idle time per trip
        (fh.TotalDelayMinutes / NULLIF(fh.TripCount, 0) * 0.3) + -- Delay frequency
        ((fh.AvgFuelEfficiency - (SELECT AVG(AvgFuelEfficiency) FROM FleetHealth)) * 100 * 0.2) + -- Fuel inefficiency deviation
        (ISNULL(fr.AssociatedRevenue, 0) / 1000 * 0.3) -- Revenue impact
    ) AS TriageScore
FROM FleetHealth fh
LEFT JOIN FleetRevenue fr ON fh.VehicleID = (SELECT VehicleID FROM dim.DimVehicle WHERE VehicleKey = fr.VehicleKey)
ORDER BY TriageScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential congestion points
WITH HourlyThroughput AS (
    SELECT 
        dt.DateKey,
        dt.Hour,
        dg.LocationName,
        COUNT(fwo.OperationKey) AS OperationCount,
        AVG(fwo.CycleTimeSeconds) AS AvgCycleTime,
        STDEV(fwo.CycleTimeSeconds) AS StdDevCycleTime
    FROM fact.FactWarehouseOperations fwo
    INNER JOIN dim.DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN dim.DimGeography dg ON fwo.WarehouseKey = dg.GeographyKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY dt.DateKey, dt.Hour, dg.LocationName
)
SELECT 
    LocationName,
    Hour,
    AVG(OperationCount) AS AvgOperationsPerHour,
    AVG(AvgCycleTime) AS AvgCycleTime,
    AVG(StdDevCycleTime) AS AvgStdDev,
    -- Bottleneck Index: High variance + high volume = congestion risk
    (AVG(OperationCount) * AVG(StdDevCycleTime) / NULLIF(AVG(AvgCycleTime), 0)) AS BottleneckIndex
FROM HourlyThroughput
GROUP BY LocationName, Hour
HAVING AVG(StdDevCycleTime) > (SELECT AVG(StdDevCycleTime) * 1.5 FROM HourlyThroughput) -- 50% above average variance
ORDER BY BottleneckIndex DESC;
```

## Power BI DAX Measures

### Core KPI Measures

```dax
// Total Operations
Total Operations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time
Avg Dwell Time (Hours) = 
AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Dwell Time Trend (vs. previous period)
Dwell Time Trend = 
VAR CurrentPeriod = [Avg Dwell Time (Hours)]
VAR PreviousPeriod = 
    CALCULATE(
        [Avg Dwell Time (Hours)],
        DATEADD(DimTime[FullDateTime], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentPeriod - PreviousPeriod, PreviousPeriod, 0)

// Fleet Utilization Rate
Fleet Utilization % = 
VAR TotalTime = SUM(FactFleetTrips[DurationMinutes])
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(TotalTime - IdleTime, TotalTime, 0)

// Cross-Dock Efficiency
Cross-Dock Efficiency % = 
VAR TargetMet = COUNTROWS(FILTER(FactCrossDock, FactCrossDock[ExceededTargetFlag] = 0))
VAR TotalTransfers = COUNTROWS(FactCrossDock)
RETURN
    DIVIDE(TargetMet, TotalTransfers, 0)

// Gravity Score Weighted Average
Weighted Gravity Score = 
SUMX(
    DimProduct,
    DimProduct[GravityScore] * RELATED(FactWarehouseOperations[QuantityHandled])
) / SUM(FactWarehouseOperations[QuantityHandled])

// Revenue at Risk (Fleet)
Revenue at Risk = 
SUMX(
    FactFleetTrips,
    VAR VehicleKey = FactFleetTrips[VehicleKey]
    VAR DelayMinutes = FactFleetTrips[DelayMinutes]
    RETURN
        DelayMinutes * 
        CALCULATE(
            SUM(DimProduct[UnitValue]),
            RELATEDTABLE(FactCrossDock),
            RELATEDTABLE(FactWarehouseOperations)
        ) * 0.01 // Cost per minute delay factor
)
```

### Time Intelligence Patterns

```dax
// Year-over-Year Growth
YoY Operations Growth = 
VAR CurrentYear = [Total Operations]
VAR PreviousYear = 
    CALCULATE(
        [Total Operations],
        SAMEPERIODLASTYEAR(DimTime[FullDateTime])
    )
RETURN
    DIVIDE(CurrentYear - PreviousYear, PreviousYear, 0)

// Moving Average (30 days)
30-Day MA Cycle Time = 
AVERAGEX(
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -30, DAY),
    [Avg Dwell Time (Hours)]
)

// Fiscal Period Comparison
Fiscal QTD vs Previous QTD = 
VAR CurrentQTD = 
    CALCULATE(
        [Total Operations],
        DATESQTD(DimTime[FullDateTime])
    )
VAR PreviousQTD = 
    CALCULATE(
        [Total Operations],
        DATESQTD(DATEADD(DimTime[FullDateTime], -1, QUARTER))
    )
RETURN
    CurrentQTD - PreviousQTD
```

## Common Patterns
