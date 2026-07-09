---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server data warehouse and Power BI analytics engine for logistics, fleet management, and supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "create logistics data warehouse with Power BI"
  - "implement fleet and warehouse tracking dashboard"
  - "build multi-fact star schema for logistics"
  - "configure supply chain analytics with SQL Server"
  - "deploy LogiFleet warehouse optimization system"
  - "integrate fleet telemetry with inventory data"
  - "create cross-modal logistics intelligence dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:
- **Multi-fact star schema data warehouse** for warehouse operations, fleet trips, and cross-dock transfers
- **Power BI dashboards** with real-time (15-minute) refresh cycles
- **Cross-fact KPI harmonization** linking inventory, fleet, and supplier metrics
- **Predictive analytics** for bottleneck detection and maintenance triage
- **Warehouse Gravity Zones™** optimization using spatial and velocity analysis

The platform integrates data from WMS, telematics, GPS, supplier portals, and external APIs into a unified semantic layer.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to source systems: WMS, TMS, telematics APIs
- Optional: Azure Synapse Analytics for big data enrichment

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the main schema script
:r schema/01_create_dimensions.sql
:r schema/02_create_facts.sql
:r schema/03_create_views.sql
:r schema/04_create_procedures.sql
:r schema/05_create_indexes.sql
```

3. **Configure data source connections:**
```json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "telematics_feed": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "erp_database": {
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "username": "${ERP_DB_USER}",
      "password": "${ERP_DB_PASSWORD}"
    }
  }
}
```

## Core Data Model Structure

### Fact Tables

**FactWarehouseOperations** - Warehouse transaction-level data:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeID INT FOREIGN KEY REFERENCES DimTime(TimeID),
    ProductID INT FOREIGN KEY REFERENCES DimProduct(ProductID),
    WarehouseZoneID INT FOREIGN KEY REFERENCES DimWarehouseZone(ZoneID),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity DECIMAL(18,2),
    DwellTimeMinutes INT,
    OperationDurationSeconds INT,
    OperatorID INT,
    ProcessedDate DATETIME2,
    CONSTRAINT CK_OpType CHECK (OperationType IN ('Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'))
);

-- Clustered index on TimeID for partition alignment
CREATE CLUSTERED INDEX IX_FactWH_TimeID ON FactWarehouseOperations(TimeID, ProcessedDate);
-- Non-clustered index for product-based queries
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductID) INCLUDE (Quantity, DwellTimeMinutes);
```

**FactFleetTrips** - Fleet telemetry and route data:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeID INT FOREIGN KEY REFERENCES DimTime(TimeID),
    VehicleID INT FOREIGN KEY REFERENCES DimVehicle(VehicleID),
    RouteID INT FOREIGN KEY REFERENCES DimRoute(RouteID),
    DriverID INT FOREIGN KEY REFERENCES DimDriver(DriverID),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    StopCount INT,
    AverageSpeedKmh DECIMAL(5,2),
    OnTimeDelivery BIT
);

CREATE CLUSTERED INDEX IX_FactFleet_TimeID ON FactFleetTrips(TimeID, TripStartTime);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID, TripStartTime) INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

**FactCrossDock** - Cross-dock transfer operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeID INT FOREIGN KEY REFERENCES DimTime(TimeID),
    ProductID INT FOREIGN KEY REFERENCES DimProduct(ProductID),
    InboundTripID BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripID),
    OutboundTripID BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripID),
    Quantity DECIMAL(18,2),
    DockDwellMinutes INT,
    TransferTime DATETIME2
);

CREATE CLUSTERED INDEX IX_FactCD_TimeID ON FactCrossDock(TimeID, TransferTime);
```

### Key Dimension Tables

**DimTime** - 15-minute granularity time dimension:
```sql
CREATE TABLE DimTime (
    TimeID INT PRIMARY KEY,
    DateKey INT,
    FullDate DATE,
    TimeOfDay TIME,
    HourOfDay TINYINT,
    QuarterHour TINYINT, -- 0, 15, 30, 45
    DayOfWeek TINYINT,
    DayName VARCHAR(20),
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    MonthName VARCHAR(20),
    Quarter TINYINT,
    Year SMALLINT,
    FiscalYear SMALLINT,
    FiscalQuarter TINYINT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- Populate time dimension
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDate DATETIME2 = @StartDate;
    DECLARE @QuarterHours TABLE (Minutes INT);
    INSERT INTO @QuarterHours VALUES (0), (15), (30), (45);
    
    WHILE @CurrentDate <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeID, DateKey, FullDate, TimeOfDay, HourOfDay, QuarterHour, DayOfWeek, DayName, WeekOfYear, MonthNumber, MonthName, Quarter, Year, FiscalYear, FiscalQuarter, IsWeekend, IsHoliday)
        SELECT 
            CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT) AS TimeID,
            CAST(FORMAT(@CurrentDate, 'yyyyMMdd') AS INT) AS DateKey,
            CAST(@CurrentDate AS DATE) AS FullDate,
            CAST(@CurrentDate AS TIME) AS TimeOfDay,
            DATEPART(HOUR, @CurrentDate) AS HourOfDay,
            qh.Minutes AS QuarterHour,
            DATEPART(WEEKDAY, @CurrentDate) AS DayOfWeek,
            DATENAME(WEEKDAY, @CurrentDate) AS DayName,
            DATEPART(WEEK, @CurrentDate) AS WeekOfYear,
            MONTH(@CurrentDate) AS MonthNumber,
            DATENAME(MONTH, @CurrentDate) AS MonthName,
            DATEPART(QUARTER, @CurrentDate) AS Quarter,
            YEAR(@CurrentDate) AS Year,
            CASE WHEN MONTH(@CurrentDate) >= 4 THEN YEAR(@CurrentDate) ELSE YEAR(@CurrentDate) - 1 END AS FiscalYear,
            CASE WHEN MONTH(@CurrentDate) >= 4 THEN DATEPART(QUARTER, @CurrentDate) ELSE DATEPART(QUARTER, @CurrentDate) + 1 END AS FiscalQuarter,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend,
            0 AS IsHoliday -- Populate separately with holiday calendar
        FROM @QuarterHours qh
        WHERE DATEADD(MINUTE, qh.Minutes, @CurrentDate) = @CurrentDate;
        
        SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
    END
END;
```

**DimProductGravity** - Product classification with gravity score:
```sql
CREATE TABLE DimProduct (
    ProductID INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(255),
    CategoryLevel1 VARCHAR(100),
    CategoryLevel2 VARCHAR(100),
    CategoryLevel3 VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLifeDays INT,
    UnitCost DECIMAL(18,2),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility factor
    RecommendedZoneType VARCHAR(50) -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity', 'Bulk'
);

-- Stored procedure to calculate and update gravity scores
CREATE PROCEDURE UpdateProductGravityScores
AS
BEGIN
    WITH VelocityCalc AS (
        SELECT 
            ProductID,
            COUNT(*) AS PickCount,
            SUM(Quantity) AS TotalQuantity,
            AVG(DwellTimeMinutes) AS AvgDwellTime
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
          AND ProcessedDate >= DATEADD(DAY, -90, GETDATE())
        GROUP BY ProductID
    )
    UPDATE p
    SET 
        GravityScore = 
            (v.PickCount / 90.0) * -- Daily pick frequency
            (p.UnitCost / 100.0) * -- Value factor
            (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END) * -- Fragility multiplier
            (CASE WHEN p.IsPerishable = 1 THEN 2.0 ELSE 1.0 END), -- Perishability multiplier
        RecommendedZoneType = 
            CASE 
                WHEN (v.PickCount / 90.0) * (p.UnitCost / 100.0) > 50 THEN 'High-Gravity'
                WHEN (v.PickCount / 90.0) * (p.UnitCost / 100.0) > 20 THEN 'Medium-Gravity'
                WHEN (v.PickCount / 90.0) * (p.UnitCost / 100.0) > 5 THEN 'Low-Gravity'
                ELSE 'Bulk'
            END
    FROM DimProduct p
    INNER JOIN VelocityCalc v ON p.ProductID = v.ProductID;
END;
```

## Key Stored Procedures

### Incremental Data Loading

**Load warehouse operations from WMS:**
```sql
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table (populated by ETL)
    INSERT INTO FactWarehouseOperations (
        TimeID, ProductID, WarehouseZoneID, OperationType, 
        Quantity, DwellTimeMinutes, OperationDurationSeconds, 
        OperatorID, ProcessedDate
    )
    SELECT 
        dt.TimeID,
        dp.ProductID,
        dz.ZoneID,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, s.StartTime, s.EndTime) AS OperationDurationSeconds,
        s.OperatorID,
        s.ProcessedDate
    FROM StagingWarehouseOps s
    INNER JOIN DimTime dt ON dt.FullDate = CAST(s.ProcessedDate AS DATE)
        AND dt.HourOfDay = DATEPART(HOUR, s.ProcessedDate)
        AND dt.QuarterHour = (DATEPART(MINUTE, s.ProcessedDate) / 15) * 15
    INNER JOIN DimProduct dp ON dp.SKU = s.SKU
    INNER JOIN DimWarehouseZone dz ON dz.ZoneCode = s.ZoneCode
    WHERE s.ProcessedDate > @LastLoadTime;
    
    -- Update last load timestamp in control table
    UPDATE ETLControl SET LastLoadTime = GETDATE() WHERE TableName = 'FactWarehouseOperations';
END;
```

**Load fleet trips from telematics:**
```sql
CREATE PROCEDURE LoadFleetTrips
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeID, VehicleID, RouteID, DriverID,
        TripStartTime, TripEndTime, DistanceKm, FuelConsumedLiters,
        IdleTimeMinutes, LoadWeightKg, StopCount, AverageSpeedKmh, OnTimeDelivery
    )
    SELECT 
        dt.TimeID,
        dv.VehicleID,
        dr.RouteID,
        dd.DriverID,
        s.TripStartTime,
        s.TripEndTime,
        s.DistanceKm,
        s.FuelConsumedLiters,
        s.IdleTimeMinutes,
        s.LoadWeightKg,
        s.StopCount,
        CASE 
            WHEN DATEDIFF(HOUR, s.TripStartTime, s.TripEndTime) > 0 
            THEN s.DistanceKm / DATEDIFF(HOUR, s.TripStartTime, s.TripEndTime)
            ELSE 0 
        END AS AverageSpeedKmh,
        CASE 
            WHEN s.ActualArrivalTime <= s.ScheduledArrivalTime THEN 1 
            ELSE 0 
        END AS OnTimeDelivery
    FROM StagingFleetTrips s
    INNER JOIN DimTime dt ON dt.FullDate = CAST(s.TripStartTime AS DATE)
        AND dt.HourOfDay = DATEPART(HOUR, s.TripStartTime)
        AND dt.QuarterHour = (DATEPART(MINUTE, s.TripStartTime) / 15) * 15
    INNER JOIN DimVehicle dv ON dv.VehicleNumber = s.VehicleNumber
    INNER JOIN DimRoute dr ON dr.RouteCode = s.RouteCode
    INNER JOIN DimDriver dd ON dd.DriverLicense = s.DriverLicense
    WHERE s.TripStartTime > @LastLoadTime;
    
    UPDATE ETLControl SET LastLoadTime = GETDATE() WHERE TableName = 'FactFleetTrips';
END;
```

### Cross-Fact KPI Views

**Composite KPI: Dwell time vs. fleet idle cost per route:**
```sql
CREATE VIEW vw_DwellTimeVsIdleCost AS
SELECT 
    dt.FullDate,
    dt.WeekOfYear,
    dp.CategoryLevel1 AS ProductCategory,
    dr.RouteName,
    dr.RouteType,
    AVG(wh.DwellTimeMinutes) AS AvgWarehouseDwellMinutes,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    SUM(ft.FuelConsumedLiters * 1.5) AS EstimatedIdleFuelCost, -- $1.50 per liter
    COUNT(DISTINCT wh.OperationID) AS WarehouseOperations,
    COUNT(DISTINCT ft.TripID) AS FleetTrips
FROM FactWarehouseOperations wh
INNER JOIN DimTime dt ON wh.TimeID = dt.TimeID
INNER JOIN DimProduct dp ON wh.ProductID = dp.ProductID
LEFT JOIN FactCrossDock cd ON wh.ProductID = cd.ProductID AND wh.TimeID = cd.TimeID
LEFT JOIN FactFleetTrips ft ON cd.OutboundTripID = ft.TripID
INNER JOIN DimRoute dr ON ft.RouteID = dr.RouteID
WHERE dt.FullDate >= DATEADD(DAY, -90, GETDATE())
GROUP BY dt.FullDate, dt.WeekOfYear, dp.CategoryLevel1, dr.RouteName, dr.RouteType;
```

**Predictive bottleneck detection:**
```sql
CREATE VIEW vw_BottleneckRiskIndex AS
WITH ZoneVelocity AS (
    SELECT 
        WarehouseZoneID,
        DATEPART(HOUR, ProcessedDate) AS HourOfDay,
        AVG(DwellTimeMinutes) AS AvgDwell,
        STDEV(DwellTimeMinutes) AS StdDevDwell,
        COUNT(*) AS OpCount
    FROM FactWarehouseOperations
    WHERE ProcessedDate >= DATEADD(DAY, -30, GETDATE())
    GROUP BY WarehouseZoneID, DATEPART(HOUR, ProcessedDate)
)
SELECT 
    dz.ZoneName,
    zv.HourOfDay,
    zv.AvgDwell,
    zv.StdDevDwell,
    zv.OpCount,
    dz.MaxCapacityUnits,
    CAST((zv.OpCount * 1.0 / dz.MaxCapacityUnits) * 100 AS DECIMAL(5,2)) AS UtilizationPercent,
    CASE 
        WHEN (zv.OpCount * 1.0 / dz.MaxCapacityUnits) > 0.85 THEN 'High Risk'
        WHEN (zv.OpCount * 1.0 / dz.MaxCapacityUnits) > 0.70 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM ZoneVelocity zv
INNER JOIN DimWarehouseZone dz ON zv.WarehouseZoneID = dz.ZoneID
WHERE (zv.OpCount * 1.0 / dz.MaxCapacityUnits) > 0.60;
```

## Power BI Integration

### Connecting to SQL Server

1. **Open the provided `.pbit` template** (`LogiFleet_Pulse_Master.pbit`)

2. **Configure data source parameters:**
   - Server: Your SQL Server instance
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use service account)

3. **The template includes pre-built relationships:**
```dax
// Relationship configuration in Power BI model
// All fact tables relate to DimTime via TimeID
// FactWarehouseOperations[TimeID] → DimTime[TimeID] (Many-to-One)
// FactFleetTrips[TimeID] → DimTime[TimeID] (Many-to-One)
// FactCrossDock[TimeID] → DimTime[TimeID] (Many-to-One)

// Cross-fact bridge via FactCrossDock
// FactFleetTrips[TripID] → FactCrossDock[OutboundTripID] (One-to-Many)
// FactFleetTrips[TripID] → FactCrossDock[InboundTripID] (One-to-Many)
```

### Key DAX Measures

**Fleet efficiency score:**
```dax
Fleet_Efficiency_Score = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKm])
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedLiters])
VAR TotalIdle = SUM(FactFleetTrips[IdleTimeMinutes])
VAR TotalDriving = SUMX(
    FactFleetTrips,
    DATEDIFF(FactFleetTrips[TripStartTime], FactFleetTrips[TripEndTime], MINUTE)
)
VAR FuelEfficiency = DIVIDE(TotalDistance, TotalFuel, 0)
VAR IdleRatio = DIVIDE(TotalIdle, TotalDriving, 0)
RETURN
    (FuelEfficiency * 10) * (1 - IdleRatio) -- Normalized 0-100 score
```

**Warehouse gravity zone optimization score:**
```dax
Gravity_Zone_Alignment_Score = 
VAR ProductsInOptimalZone = 
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            RELATED(DimProduct[RecommendedZoneType]) = RELATED(DimWarehouseZone[ZoneType])
        )
    )
VAR TotalOperations = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(ProductsInOptimalZone, TotalOperations, 0) * 100
```

**Predictive lead time variance:**
```dax
Predicted_Lead_Time_Days = 
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDate], MAX(DimTime[FullDate]), -90, DAY)
    )
VAR HistoricalStdDev = 
    CALCULATE(
        STDEV.P(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDate], MAX(DimTime[FullDate]), -90, DAY)
    )
RETURN
    (HistoricalAvg + (1.5 * HistoricalStdDev)) / (60 * 24) -- Convert to days with 1.5 sigma buffer
```

### Scheduled Refresh Configuration

**Set up incremental refresh in Power BI:**
```powerquery
// Power Query M code for incremental refresh
let
    Source = Sql.Database("YOUR_SERVER", "LogiFleetPulse"),
    FactWarehouseOperations = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FilteredRows = Table.SelectRows(
        FactWarehouseOperations, 
        each [ProcessedDate] >= RangeStart and [ProcessedDate] < RangeEnd
    )
in
    FilteredRows
```

**Configure in Power BI Service:**
- Incremental refresh: Last 7 days (full refresh)
- Historical data: 2 years (partitioned by month)
- Refresh frequency: Every 15 minutes (Premium capacity required)

## Automated Alerting System

### Create alert stored procedure:
```sql
CREATE PROCEDURE GenerateLogisticsAlerts
AS
BEGIN
    DECLARE @Alerts TABLE (
        AlertType VARCHAR(100),
        Severity VARCHAR(20),
        Message NVARCHAR(500),
        AffectedEntity VARCHAR(255),
        MetricValue DECIMAL(18,2)
    );
    
    -- Alert: High warehouse dwell time
    INSERT INTO @Alerts
    SELECT 
        'High Dwell Time' AS AlertType,
        'Warning' AS Severity,
        'Product ' + dp.SKU + ' in zone ' + dz.ZoneName + ' has exceeded normal dwell time' AS Message,
        dp.SKU AS AffectedEntity,
        AVG(DwellTimeMinutes) AS MetricValue
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct dp ON wh.ProductID = dp.ProductID
    INNER JOIN DimWarehouseZone dz ON wh.WarehouseZoneID = dz.ZoneID
    WHERE ProcessedDate >= DATEADD(HOUR, -2, GETDATE())
    GROUP BY dp.SKU, dz.ZoneName
    HAVING AVG(DwellTimeMinutes) > 120;
    
    -- Alert: Fleet idle time excessive
    INSERT INTO @Alerts
    SELECT 
        'Excessive Idle Time' AS AlertType,
        'Critical' AS Severity,
        'Vehicle ' + dv.VehicleNumber + ' idle time is ' + CAST(AVG(IdleTimeMinutes) AS VARCHAR) + ' minutes' AS Message,
        dv.VehicleNumber AS AffectedEntity,
        AVG(IdleTimeMinutes) AS MetricValue
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle dv ON ft.VehicleID = dv.VehicleID
    WHERE TripStartTime >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY dv.VehicleNumber
    HAVING AVG(IdleTimeMinutes) > DATEDIFF(MINUTE, MIN(TripStartTime), MAX(TripEndTime)) * 0.15;
    
    -- Alert: On-time delivery below threshold
    INSERT INTO @Alerts
    SELECT 
        'Low On-Time Delivery' AS AlertType,
        'Warning' AS Severity,
        'Route ' + dr.RouteName + ' has only ' + CAST(AVG(CAST(OnTimeDelivery AS DECIMAL)) * 100 AS VARCHAR) + '% on-time delivery' AS Message,
        dr.RouteName AS AffectedEntity,
        AVG(CAST(OnTimeDelivery AS DECIMAL)) * 100 AS MetricValue
    FROM FactFleetTrips ft
    INNER JOIN DimRoute dr ON ft.RouteID = dr.RouteID
    WHERE TripStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY dr.RouteName
    HAVING AVG(CAST(OnTimeDelivery AS DECIMAL)) < 0.85;
    
    -- Send alerts (implement email/Teams notification)
    SELECT * FROM @Alerts ORDER BY Severity DESC, MetricValue DESC;
END;
```

### Schedule alert job:
```sql
-- Create SQL Server Agent job (run every 15 minutes)
USE msdb;
GO

EXEC sp_add_job 
    @job_name = 'LogiFleet_Alerts_Monitor',
    @enabled = 1,
    @description = 'Monitors logistics KPIs and generates alerts';

EXEC sp_add_jobstep 
    @job_name = 'LogiFleet_Alerts_Monitor',
    @step_name = 'Execute Alert Procedure',
    @subsystem = 'TSQL',
    @database_name = 'LogiFleetPulse',
    @command = 'EXEC GenerateLogisticsAlerts;';

EXEC sp_add_schedule 
    @schedule_name = 'Every_15_Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule 
    @job_name = 'LogiFleet_Alerts_Monitor',
    @schedule_name = 'Every_15_Minutes';

EXEC sp_add_jobserver 
    @job_name = 'LogiFleet_Alerts_Monitor';
```

## Common Usage Patterns

### Pattern 1: Daily Operations Dashboard Query

**Show today's warehouse efficiency by zone:**
```sql
SELECT 
    dz.ZoneName,
    dz.ZoneType,
    COUNT(DISTINCT wh.OperationID) AS TotalOperations,
    AVG(wh.DwellTimeMinutes) AS AvgDwellMinutes,
    SUM(wh.Quantity) AS TotalUnitsProcessed,
    AVG(wh.OperationDurationSeconds) AS AvgOperationSeconds,
    COUNT(DISTINCT wh.OperatorID) AS ActiveOperators,
    CAST((COUNT(DISTINCT wh.OperationID) * 1.0 / dz.MaxCapacityUnits) * 100 AS DECIMAL(5,2)) AS UtilizationPercent
FROM FactWarehouseOperations wh
INNER JOIN DimTime dt ON wh.TimeID = dt.TimeID
INNER JOIN DimWarehouseZone dz ON wh.WarehouseZoneID = dz.ZoneID
WHERE dt.FullDate = CAST(GETDATE() AS DATE)
GROUP BY dz.ZoneName, dz.ZoneType, dz.MaxCapacityUnits
ORDER BY UtilizationPercent DESC;
```

### Pattern 2: Fleet Performance Analysis

**Weekly fleet performance with cost breakdown:**
```sql
SELECT 
    DATEPART(WEEK, dt.FullDate) AS WeekNumber,
    dv.VehicleNumber,
    dv.VehicleType,
    COUNT(ft.TripID) AS TotalTrips,
    SUM(ft.DistanceKm) AS TotalDistanceKm,
    SUM(ft.FuelConsumedLiters) AS TotalFuelLiters,
    AVG(ft.AverageSpeedKmh) AS AvgSpeedKmh,
    SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
    SUM(ft.FuelConsumedLiters * 1.5) AS EstimatedFuelCost,
    AVG(CAST(ft.OnTimeDelivery AS DECIMAL)) * 100 AS OnTimeDeliveryPercent,
    CASE 
        WHEN AVG(CAST(ft.OnTimeDelivery AS DECIMAL)) >= 0.95 THEN 'Excellent'
        WHEN AVG(CAST(ft.OnTimeDelivery AS DECIMAL)) >= 0.85 THEN 'Good'
        WHEN AVG(CAST(ft.OnTimeDelivery AS DECIMAL)) >= 0.75 THEN '
