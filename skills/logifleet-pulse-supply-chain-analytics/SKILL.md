---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI logistics intelligence platform for warehouse operations, fleet management, and supply chain optimization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy warehouse and fleet data model"
  - "create Power BI logistics dashboard"
  - "configure multi-fact star schema for supply chain"
  - "implement warehouse gravity zones analytics"
  - "build fleet optimization data warehouse"
  - "integrate WMS and telematics data streams"
  - "create cross-modal logistics KPI dashboard"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that combines:
- **Warehouse operations analytics** (receiving, putaway, picking, packing, shipping)
- **Fleet telemetry and GPS tracking** (routes, fuel consumption, driver behavior)
- **Inventory management** (aging curves, SKU velocity, storage optimization)
- **Cross-fact KPI harmonization** using a custom multi-fact star schema

Built on MS SQL Server for data warehousing and Power BI for visualization, it provides real-time logistics decision support through:
- Unified semantic layer across warehouse, fleet, and external data
- Time-phased dimensions (15-minute granularity)
- Predictive bottleneck detection
- Warehouse Gravity Zones™ (spatial optimization based on pick frequency, value, fragility)
- Adaptive fleet maintenance triage

## Installation and Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to source systems: WMS, TMS, telematics APIs, ERP
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Execute in SSMS or Azure Data Studio
-- Connect to your target database instance

-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Run schema creation scripts in order:
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create bridge tables
-- 4. Create views
-- 5. Create stored procedures
-- 6. Configure indexing
```

3. **Configure data source connections:**
```json
{
  "connections": {
    "wms_api": {
      "connection_string": "${WMS_CONNECTION_STRING}",
      "refresh_interval_minutes": 15
    },
    "fleet_telemetry": {
      "connection_string": "${FLEET_API_CONNECTION_STRING}",
      "refresh_interval_minutes": 5
    },
    "erp_system": {
      "connection_string": "${ERP_CONNECTION_STRING}",
      "refresh_interval_minutes": 30
    }
  }
}
```

## Core Data Model Components

### Dimension Tables

#### DimTime - Time-phased dimension with 15-minute granularity
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeStamp DATETIME2 NOT NULL,
    TimeInterval15Min VARCHAR(5), -- e.g., '09:15'
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    MonthName VARCHAR(10),
    Quarter TINYINT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT,
    ShiftType VARCHAR(20) -- 'Morning', 'Afternoon', 'Night'
)

-- Create clustered index
CREATE CLUSTERED INDEX IX_DimTime_DateTime 
ON DimTime(DateTimeStamp)

-- Populate time dimension (example for one day)
DECLARE @StartDate DATETIME2 = '2026-01-01 00:00:00'
DECLARE @EndDate DATETIME2 = '2026-12-31 23:59:59'
DECLARE @CurrentDate DATETIME2 = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        DateTimeStamp, 
        TimeInterval15Min, 
        HourOfDay, 
        DayOfWeek,
        DayName,
        WeekOfYear,
        MonthNumber,
        MonthName,
        Quarter,
        FiscalYear,
        IsWeekend,
        IsHoliday,
        ShiftType
    )
    VALUES (
        @CurrentDate,
        FORMAT(@CurrentDate, 'HH:mm'),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATENAME(WEEKDAY, @CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        MONTH(@CurrentDate),
        DATENAME(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        YEAR(@CurrentDate),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        0, -- Populate with holiday calendar logic
        CASE 
            WHEN DATEPART(HOUR, @CurrentDate) BETWEEN 6 AND 13 THEN 'Morning'
            WHEN DATEPART(HOUR, @CurrentDate) BETWEEN 14 AND 21 THEN 'Afternoon'
            ELSE 'Night'
        END
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
```

#### DimProductGravity - Product hierarchy with gravity scoring
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    
    -- Gravity components
    PickFrequencyScore DECIMAL(5,2), -- 0-100, higher = more frequent
    ItemValue DECIMAL(10,2),
    ValueScore DECIMAL(5,2), -- 0-100, normalized value
    FragilityIndex DECIMAL(5,2), -- 0-100, higher = more fragile
    
    -- Composite gravity score
    GravityScore AS (
        (PickFrequencyScore * 0.5) + 
        (ValueScore * 0.3) + 
        (FragilityIndex * 0.2)
    ) PERSISTED,
    
    -- Storage recommendations
    RecommendedZoneType VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    OptimalDistanceFromDock INT, -- in meters
    
    -- Metadata
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    LastUpdated DATETIME2 DEFAULT GETDATE()
)

-- Index on gravity score for zone assignment queries
CREATE INDEX IX_ProductGravity_Score 
ON DimProductGravity(GravityScore DESC)
```

#### DimGeography - Hierarchical location dimension
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Distribution Center', 'Route Node', 'Customer'
    
    -- Hierarchy
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    PostalCode VARCHAR(20),
    
    -- Coordinates
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    
    -- Warehouse-specific
    StorageCapacityCubicMeters DECIMAL(12,2),
    DockCount TINYINT,
    ZoneCount TINYINT,
    
    -- Metadata
    ActiveFlag BIT DEFAULT 1,
    OpenDate DATE,
    CloseDate DATE NULL
)

CREATE INDEX IX_Geography_Type_Country 
ON DimGeography(LocationType, Country)
```

### Fact Tables

#### FactWarehouseOperations - Core warehouse activity fact
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    
    -- Dimension foreign keys
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    
    -- Operation details
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderNumber VARCHAR(50),
    BatchID VARCHAR(50),
    
    -- Metrics
    QuantityUnits INT,
    DwellTimeMinutes INT, -- Time item spent in current operation
    CycleTimeSeconds INT, -- Time to complete the operation
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50), -- Forklift, pallet jack, etc.
    
    -- Quality indicators
    AccuracyFlag BIT, -- 1 = no errors, 0 = correction needed
    DamageFlag BIT,
    
    -- Storage location
    ZoneID VARCHAR(20),
    AisleID VARCHAR(10),
    RackID VARCHAR(10),
    BinID VARCHAR(10),
    DistanceFromDockMeters INT,
    
    -- Timestamps
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    LoadedTimestamp DATETIME2 DEFAULT GETDATE()
)

-- Partitioning by month for performance
CREATE PARTITION FUNCTION PF_WarehouseOps_Month (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01',
    '2026-05-01', '2026-06-01', '2026-07-01', '2026-08-01',
    '2026-09-01', '2026-10-01', '2026-11-01', '2026-12-01'
)

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_CS
ON FactWarehouseOperations (
    TimeKey, ProductKey, WarehouseKey, OperationType,
    QuantityUnits, DwellTimeMinutes, CycleTimeSeconds,
    DistanceFromDockMeters
)
```

#### FactFleetTrips - Fleet telemetry and route fact
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    
    -- Dimension foreign keys
    TripStartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    TripEndTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    
    -- Trip identifiers
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    RouteID VARCHAR(50),
    ShipmentID VARCHAR(50),
    
    -- Metrics
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(8,2),
    AverageSpeedKph DECIMAL(6,2),
    MaxSpeedKph DECIMAL(6,2),
    
    -- Loading/unloading
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    StopCount TINYINT,
    
    -- Costs
    FuelCost DECIMAL(10,2),
    TollCost DECIMAL(10,2),
    MaintenanceCostAllocated DECIMAL(10,2),
    
    -- Cargo details
    CargoWeightKg DECIMAL(10,2),
    CargoValueUSD DECIMAL(12,2),
    PalletCount INT,
    
    -- External factors
    WeatherCondition VARCHAR(50), -- 'Clear', 'Rain', 'Snow', 'Fog'
    TrafficDelayMinutes INT,
    
    -- Quality
    OnTimeFlag BIT,
    DamageReportedFlag BIT,
    
    -- Timestamps
    ActualStartTime DATETIME2,
    ActualEndTime DATETIME2,
    LoadedTimestamp DATETIME2 DEFAULT GETDATE()
)

-- Columnstore for cross-fact queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_CS
ON FactFleetTrips (
    TripStartTimeKey, OriginKey, DestinationKey,
    DistanceKm, DurationMinutes, IdleTimeMinutes,
    FuelConsumedLiters, OnTimeFlag
)
```

## Key Analytical Views and Stored Procedures

### Cross-Fact KPI: Dwell Time vs Fleet Idle Time Correlation
```sql
CREATE VIEW vw_DwellTime_Fleet_Correlation AS
SELECT 
    t.DateTimeStamp,
    t.DayName,
    t.HourOfDay,
    pg.Category AS ProductCategory,
    
    -- Warehouse metrics
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(wo.QuantityUnits) AS TotalUnitsProcessed,
    
    -- Fleet metrics (for shipments from same warehouse)
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(ft.DurationMinutes) AS AvgTripDuration,
    
    -- Cross-fact KPI
    (AVG(wo.DwellTimeMinutes) / NULLIF(AVG(ft.DurationMinutes), 0)) * 100 
        AS DwellToTripRatioPct,
    
    -- Cost impact
    SUM(ft.FuelCost) AS TotalFuelCost,
    SUM(wo.DwellTimeMinutes * 0.5) AS EstimatedWarehousingCost -- $0.50 per minute
    
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
INNER JOIN FactFleetTrips ft ON wo.OrderNumber = ft.ShipmentID
    AND wo.WarehouseKey = ft.OriginKey

WHERE wo.OperationType = 'Shipping'
    AND t.DateTimeStamp >= DATEADD(DAY, -30, GETDATE())

GROUP BY 
    t.DateTimeStamp,
    t.DayName,
    t.HourOfDay,
    pg.Category
```

### Warehouse Gravity Zone Assignment Procedure
```sql
CREATE PROCEDURE usp_AssignGravityZones
    @WarehouseKey INT,
    @RecalculateScores BIT = 0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Recalculate gravity scores based on recent activity
    IF @RecalculateScores = 1
    BEGIN
        ;WITH ProductActivity AS (
            SELECT 
                ProductKey,
                COUNT(*) AS PickCount,
                AVG(CycleTimeSeconds) AS AvgCycleTime,
                SUM(QuantityUnits) AS TotalUnits
            FROM FactWarehouseOperations
            WHERE WarehouseKey = @WarehouseKey
                AND OperationType = 'Picking'
                AND TimeKey IN (
                    SELECT TimeKey FROM DimTime 
                    WHERE DateTimeStamp >= DATEADD(DAY, -90, GETDATE())
                )
            GROUP BY ProductKey
        )
        UPDATE pg
        SET 
            PickFrequencyScore = (pa.PickCount * 1.0 / MAX(pa.PickCount) OVER()) * 100,
            LastUpdated = GETDATE()
        FROM DimProductGravity pg
        INNER JOIN ProductActivity pa ON pg.ProductKey = pa.ProductKey
    END
    
    -- Assign zones based on gravity scores
    ;WITH ZoneAssignment AS (
        SELECT 
            ProductKey,
            SKU,
            GravityScore,
            CASE 
                WHEN GravityScore >= 70 THEN 'High-Gravity'
                WHEN GravityScore >= 40 THEN 'Medium-Gravity'
                ELSE 'Low-Gravity'
            END AS ZoneType,
            CASE 
                WHEN GravityScore >= 70 THEN 10  -- 10 meters from dock
                WHEN GravityScore >= 40 THEN 25
                ELSE 50
            END AS DistanceFromDock
        FROM DimProductGravity
    )
    UPDATE pg
    SET 
        RecommendedZoneType = za.ZoneType,
        OptimalDistanceFromDock = za.DistanceFromDock,
        LastUpdated = GETDATE()
    FROM DimProductGravity pg
    INNER JOIN ZoneAssignment za ON pg.ProductKey = za.ProductKey
    
    -- Return assignment summary
    SELECT 
        RecommendedZoneType,
        COUNT(*) AS ProductCount,
        AVG(GravityScore) AS AvgGravityScore,
        AVG(OptimalDistanceFromDock) AS AvgDistanceFromDock
    FROM DimProductGravity
    GROUP BY RecommendedZoneType
    ORDER BY AvgGravityScore DESC
END
GO
```

### Predictive Bottleneck Detection
```sql
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHours INT = 4,
    @ThresholdPercentile DECIMAL(5,2) = 90.0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Analyze historical patterns to predict congestion
    ;WITH HistoricalPeaks AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            wo.OperationType,
            wo.WarehouseKey,
            PERCENTILE_CONT(@ThresholdPercentile/100.0) 
                WITHIN GROUP (ORDER BY wo.CycleTimeSeconds) 
                OVER (PARTITION BY t.HourOfDay, t.DayOfWeek, wo.OperationType) 
                AS CycleTimeP90,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateTimeStamp >= DATEADD(DAY, -90, GETDATE())
        GROUP BY t.HourOfDay, t.DayOfWeek, wo.OperationType, wo.WarehouseKey
    ),
    CurrentLoad AS (
        SELECT 
            t.HourOfDay,
            wo.OperationType,
            wo.WarehouseKey,
            COUNT(*) AS CurrentOperationCount,
            AVG(wo.CycleTimeSeconds) AS CurrentAvgCycleTime
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateTimeStamp >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY t.HourOfDay, wo.OperationType, wo.WarehouseKey
    ),
    ForecastWindow AS (
        SELECT DISTINCT
            HourOfDay,
            DayOfWeek
        FROM DimTime
        WHERE DateTimeStamp BETWEEN GETDATE() 
            AND DATEADD(HOUR, @ForecastHours, GETDATE())
    )
    SELECT 
        fw.HourOfDay AS ForecastHour,
        hp.OperationType,
        g.LocationName AS Warehouse,
        hp.CycleTimeP90 AS HistoricalP90CycleTime,
        cl.CurrentAvgCycleTime,
        cl.CurrentOperationCount,
        
        -- Bottleneck probability score (0-100)
        CASE 
            WHEN cl.CurrentAvgCycleTime > hp.CycleTimeP90 * 0.8 THEN 85
            WHEN cl.CurrentAvgCycleTime > hp.CycleTimeP90 * 0.6 THEN 60
            WHEN cl.CurrentOperationCount > hp.OperationCount * 1.2 THEN 45
            ELSE 20
        END AS BottleneckProbabilityScore,
        
        -- Recommended action
        CASE 
            WHEN cl.CurrentAvgCycleTime > hp.CycleTimeP90 * 0.8 THEN 'CRITICAL: Add staff immediately'
            WHEN cl.CurrentAvgCycleTime > hp.CycleTimeP90 * 0.6 THEN 'WARNING: Monitor closely, prepare backup'
            WHEN cl.CurrentOperationCount > hp.OperationCount * 1.2 THEN 'ALERT: Elevated load detected'
            ELSE 'NORMAL: Within expected parameters'
        END AS RecommendedAction
        
    FROM ForecastWindow fw
    INNER JOIN HistoricalPeaks hp ON fw.HourOfDay = hp.HourOfDay AND fw.DayOfWeek = hp.DayOfWeek
    LEFT JOIN CurrentLoad cl ON hp.OperationType = cl.OperationType 
        AND hp.WarehouseKey = cl.WarehouseKey
    INNER JOIN DimGeography g ON hp.WarehouseKey = g.GeographyKey
    
    WHERE hp.OperationCount > 10 -- Ignore low-volume operations
    ORDER BY BottleneckProbabilityScore DESC, fw.HourOfDay
END
GO
```

## Power BI Configuration

### Connect to SQL Server Data Model

1. **Open Power BI Desktop**
2. **Get Data → SQL Server**
```
Server: ${SQL_SERVER_INSTANCE}
Database: LogiFleetPulse
Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)
```

3. **Select tables:**
- All Dim* tables (dimensions)
- All Fact* tables (facts)
- Key views: `vw_DwellTime_Fleet_Correlation`

### DAX Measures for Cross-Fact KPIs

#### Warehouse Efficiency Score
```dax
Warehouse Efficiency Score = 
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
VAR TargetCycleTime = 45 -- seconds
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR TargetDwellTime = 120 -- minutes

VAR CycleEfficiency = DIVIDE(TargetCycleTime, AvgCycleTime, 1) * 100
VAR DwellEfficiency = DIVIDE(TargetDwellTime, AvgDwellTime, 1) * 100

RETURN 
    (CycleEfficiency * 0.6) + (DwellEfficiency * 0.4)
```

#### Fleet Cost Per Km
```dax
Fleet Cost Per Km = 
DIVIDE(
    SUM(FactFleetTrips[FuelCost]) + 
    SUM(FactFleetTrips[MaintenanceCostAllocated]) + 
    SUM(FactFleetTrips[TollCost]),
    SUM(FactFleetTrips[DistanceKm]),
    0
)
```

#### Cross-Fact: Revenue Impact of Dwell Time
```dax
Dwell Time Revenue Impact = 
VAR TotalDwellMinutes = SUM(FactWarehouseOperations[DwellTimeMinutes])
VAR OpportunityCostPerMinute = 2.5 -- USD
VAR DelayedShipments = 
    CALCULATE(
        COUNTROWS(FactFleetTrips),
        FactFleetTrips[OnTimeFlag] = 0
    )
VAR AvgShipmentValue = AVERAGE(FactFleetTrips[CargoValueUSD])

RETURN 
    (TotalDwellMinutes * OpportunityCostPerMinute) + 
    (DelayedShipments * AvgShipmentValue * 0.05) -- 5% penalty for delays
```

#### Gravity Zone Compliance
```dax
Gravity Zone Compliance % = 
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
VAR CompliantOperations = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DistanceFromDockMeters] <= 
            RELATED(DimProductGravity[OptimalDistanceFromDock]) * 1.2 -- 20% tolerance
    )

RETURN 
    DIVIDE(CompliantOperations, TotalOperations, 0) * 100
```

### Create Relationships in Power BI Model

```dax
// In Power BI Desktop, Model view, create relationships:

FactWarehouseOperations[TimeKey] → DimTime[TimeKey] (Many-to-One)
FactWarehouseOperations[ProductKey] → DimProductGravity[ProductKey] (Many-to-One)
FactWarehouseOperations[WarehouseKey] → DimGeography[GeographyKey] (Many-to-One)

FactFleetTrips[TripStartTimeKey] → DimTime[TimeKey] (Many-to-One, role: TripStart)
FactFleetTrips[TripEndTimeKey] → DimTime[TimeKey] (Many-to-One, inactive, role: TripEnd)
FactFleetTrips[OriginKey] → DimGeography[GeographyKey] (Many-to-One, role: Origin)
FactFleetTrips[DestinationKey] → DimGeography[GeographyKey] (Many-to-One, inactive, role: Destination)

// For inactive relationships, use USERELATIONSHIP() in DAX:
Trip End Time Metric = CALCULATE(SUM(FactFleetTrips[DurationMinutes]), USERELATIONSHIP(FactFleetTrips[TripEndTimeKey], DimTime[TimeKey]))
```

## Data Refresh and ETL Patterns

### Incremental Load Stored Procedure
```sql
CREATE PROCEDURE usp_IncrementalLoad_WarehouseOps
    @LastLoadTimestamp DATETIME2 = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last loaded timestamp if not provided
    IF @LastLoadTimestamp IS NULL
    BEGIN
        SELECT @LastLoadTimestamp = MAX(LoadedTimestamp) 
        FROM FactWarehouseOperations
    END
    
    -- Insert new records from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey,
        OperationType, OrderNumber, BatchID,
        QuantityUnits, DwellTimeMinutes, CycleTimeSeconds,
        OperatorID, EquipmentID, AccuracyFlag, DamageFlag,
        ZoneID, AisleID, RackID, BinID, DistanceFromDockMeters,
        OperationStartTime, OperationEndTime
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        stg.OperationType,
        stg.OrderNumber,
        stg.BatchID,
        stg.QuantityUnits,
        stg.DwellTimeMinutes,
        stg.CycleTimeSeconds,
        stg.OperatorID,
        stg.EquipmentID,
        stg.AccuracyFlag,
        stg.DamageFlag,
        stg.ZoneID,
        stg.AisleID,
        stg.RackID,
        stg.BinID,
        stg.DistanceFromDockMeters,
        stg.OperationStartTime,
        stg.OperationEndTime
    FROM Staging_WarehouseOperations stg
    INNER JOIN DimTime t ON stg.OperationStartTime >= t.DateTimeStamp 
        AND stg.OperationStartTime < DATEADD(MINUTE, 15, t.DateTimeStamp)
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    INNER JOIN DimGeography g ON stg.WarehouseID = g.LocationID
    WHERE stg.LoadedToStaging > @LastLoadTimestamp
    
    -- Return row count
    SELECT @@ROWCOUNT AS RowsInserted
END
GO
```

### Schedule in SQL Server Agent
```sql
-- Create job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleetPulse_IncrementalRefresh',
    @enabled = 1

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleetPulse_IncrementalRefresh',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @command = N'EXEC usp_IncrementalLoad_WarehouseOps',
    @database_name = N'LogiFleetPulse'

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleetPulse_IncrementalRefresh',
    @schedule_name = N'Every15Minutes'

EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'LogiFleetPulse_IncrementalRefresh'
```

## Common Usage Patterns

### Query: Top 10 Products by Gravity Score
```sql
SELECT TOP 10
    SKU,
    ProductName,
    Category,
    GravityScore,
    RecommendedZoneType,
    OptimalDistanceFromDock
FROM DimProductGravity
ORDER BY GravityScore DESC
```

### Query: Daily Warehouse Performance Summary
```sql
SELECT 
    CAST(t.DateTimeStamp AS DATE) AS OperationDate,
    g.LocationName AS Warehouse,
    wo.OperationType,
    COUNT(*) AS OperationCount,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTimeSeconds,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(CASE WHEN wo.AccuracyFlag = 1 THEN 1 ELSE 0 END) *
