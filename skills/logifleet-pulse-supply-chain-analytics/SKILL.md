---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehousing engine with multi-fact star schema for fleet and warehouse intelligence
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - implement multi-fact star schema for logistics
  - create Power BI logistics dashboard
  - configure warehouse and fleet data model
  - deploy SQL Server logistics data warehouse
  - build cross-modal supply chain analytics
  - integrate telemetry with warehouse operations
  - implement predictive logistics bottleneck detection
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server and Power BI-based logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a multi-fact star schema. It enables cross-fact KPI analysis, predictive bottleneck detection, and real-time operational dashboards for logistics optimization.

**Core Capabilities:**
- Multi-fact star schema (FactWarehouseOperations, FactFleetTrips, FactCrossDock)
- Time-phased dimensions with 15-minute granularity
- Warehouse Gravity Zones for spatial optimization
- Fleet triage scoring based on revenue impact
- Power BI semantic layer with role-based security
- Predictive analytics for congestion and maintenance

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telematics APIs)

### Deploy SQL Schema

```sql
-- 1. Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Deploy dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    Hour TINYINT,
    Minute TINYINT,
    DayOfWeek VARCHAR(10),
    FiscalPeriod VARCHAR(10),
    IsBusinessHour BIT
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Route Node, Cross-Dock
    ParentLocationID VARCHAR(50),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Weighted score: velocity + value + fragility
    IsPerishable BIT,
    WeightKg DECIMAL(10,2),
    VolumeM3 DECIMAL(10,4)
);

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeAvgDays INT,
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2)
);

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL,
    VehicleType VARCHAR(50), -- Truck, Van, Container
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(50),
    LastMaintenanceDate DATE,
    MaintenanceScore DECIMAL(5,2) -- Proactive triage score
);

-- 3. Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity DECIMAL(10,2),
    DwellTimeMinutes INT,
    StorageZone VARCHAR(50),
    GravityZoneAssignment VARCHAR(50),
    EmployeeID VARCHAR(50),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (SupplierKey) REFERENCES DimSupplier(SupplierKey)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    FleetKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripDistanceKm DECIMAL(10,2),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    AverageSpeedKmH DECIMAL(5,2),
    WeatherCondition VARCHAR(50),
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (FleetKey) REFERENCES DimFleet(FleetKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundFleetKey INT,
    OutboundFleetKey INT,
    TransferDurationMinutes INT,
    TemporaryStorageMinutes INT,
    Quantity DECIMAL(10,2),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (InboundFleetKey) REFERENCES DimFleet(FleetKey),
    FOREIGN KEY (OutboundFleetKey) REFERENCES DimFleet(FleetKey)
);

-- 4. Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
CREATE NONCLUSTERED INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey);
```

### Create Time Dimension Population Procedure

```sql
CREATE PROCEDURE PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @EndDateTime DATETIME2 = DATEADD(DAY, 1, @EndDate);
    
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        INSERT INTO DimTime (TimeKey, DateTime, Date, Hour, Minute, DayOfWeek, FiscalPeriod, IsBusinessHour)
        VALUES (
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            CONCAT('FY', YEAR(DATEADD(MONTH, 6, @CurrentDateTime)), 'Q', DATEPART(QUARTER, DATEADD(MONTH, 6, @CurrentDateTime))),
            CASE WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
                 AND DATEPART(WEEKDAY, @CurrentDateTime) NOT IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Populate time dimension for 2 years
EXEC PopulateDimTime '2025-01-01', '2026-12-31';
```

## Data Loading Patterns

### Incremental Warehouse Operations Load

```sql
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from staging or external source
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, 
        StorageZone, GravityZoneAssignment, EmployeeID
    )
    SELECT 
        CAST(FORMAT(o.OperationDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        o.OperationType,
        o.Quantity,
        DATEDIFF(MINUTE, o.StartTime, o.EndTime) AS DwellTimeMinutes,
        o.StorageZone,
        -- Assign gravity zone based on product score
        CASE 
            WHEN p.GravityScore > 8.0 THEN 'High-Gravity'
            WHEN p.GravityScore > 5.0 THEN 'Medium-Gravity'
            ELSE 'Low-Gravity'
        END AS GravityZoneAssignment,
        o.EmployeeID
    FROM StagingWarehouseOperations o
    INNER JOIN DimGeography g ON o.LocationID = g.LocationID
    INNER JOIN DimProduct p ON o.SKU = p.SKU
    LEFT JOIN DimSupplier s ON o.SupplierID = s.SupplierID
    WHERE o.OperationDateTime > @LastLoadDateTime;
END;
GO
```

### Fleet Telemetry Integration

```sql
CREATE PROCEDURE LoadFleetTrips
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, FleetKey, OriginGeographyKey, DestinationGeographyKey,
        TripDistanceKm, TripDurationMinutes, IdleTimeMinutes,
        FuelConsumedLiters, LoadWeightKg, AverageSpeedKmH,
        WeatherCondition, DelayMinutes, DelayReason
    )
    SELECT 
        CAST(FORMAT(t.TripStartDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        f.FleetKey,
        og.GeographyKey AS OriginGeographyKey,
        dg.GeographyKey AS DestinationGeographyKey,
        t.DistanceKm,
        DATEDIFF(MINUTE, t.TripStartDateTime, t.TripEndDateTime),
        t.IdleMinutes,
        t.FuelLiters,
        t.LoadKg,
        CASE WHEN DATEDIFF(MINUTE, t.TripStartDateTime, t.TripEndDateTime) > 0
             THEN t.DistanceKm / (DATEDIFF(MINUTE, t.TripStartDateTime, t.TripEndDateTime) / 60.0)
             ELSE 0 END AS AvgSpeed,
        t.WeatherCondition,
        CASE WHEN t.PlannedArrival IS NOT NULL 
             THEN DATEDIFF(MINUTE, t.PlannedArrival, t.TripEndDateTime)
             ELSE 0 END AS DelayMinutes,
        t.DelayReason
    FROM StagingFleetTrips t
    INNER JOIN DimFleet f ON t.VehicleID = f.VehicleID
    INNER JOIN DimGeography og ON t.OriginLocationID = og.LocationID
    INNER JOIN DimGeography dg ON t.DestinationLocationID = dg.LocationID
    WHERE t.TripStartDateTime > @LastLoadDateTime;
END;
GO
```

## Key Analytics Queries

### Cross-Fact Analysis: Dwell Time vs Fleet Idle Time

```sql
-- Correlate warehouse dwell time with fleet idle time by product category
WITH WarehouseDwell AS (
    SELECT 
        p.Category,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.Category
),
FleetIdle AS (
    SELECT 
        p.Category,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(ft.FuelConsumedLiters) AS TotalFuel
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON ft.OriginGeographyKey = wo.GeographyKey
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.Category
)
SELECT 
    wd.Category,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    fi.TotalFuel,
    -- Correlation index
    (wd.AvgDwellMinutes * fi.AvgIdleMinutes) / 100.0 AS FrictionIndex
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.Category = fi.Category
ORDER BY FrictionIndex DESC;
```

### Predictive Bottleneck Detection

```sql
CREATE VIEW vw_BottleneckPrediction AS
WITH CurrentLoad AS (
    SELECT 
        g.LocationName,
        g.LocationType,
        COUNT(*) AS ActiveOperations,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        MAX(wo.DwellTimeMinutes) AS MaxDwell
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -2, GETDATE())
    GROUP BY g.LocationName, g.LocationType
),
HistoricalBaseline AS (
    SELECT 
        g.LocationName,
        AVG(wo.DwellTimeMinutes) AS BaselineDwell,
        STDEV(wo.DwellTimeMinutes) AS DwellStdDev
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
      AND t.Date < DATEADD(DAY, -1, GETDATE())
    GROUP BY g.LocationName
)
SELECT 
    cl.LocationName,
    cl.LocationType,
    cl.ActiveOperations,
    cl.AvgDwell AS CurrentAvgDwell,
    hb.BaselineDwell,
    -- Bottleneck score (standard deviations above baseline)
    CASE WHEN hb.DwellStdDev > 0 
         THEN (cl.AvgDwell - hb.BaselineDwell) / hb.DwellStdDev 
         ELSE 0 END AS BottleneckScore,
    CASE 
        WHEN (cl.AvgDwell - hb.BaselineDwell) / NULLIF(hb.DwellStdDev, 0) > 2.0 THEN 'Critical'
        WHEN (cl.AvgDwell - hb.BaselineDwell) / NULLIF(hb.DwellStdDev, 0) > 1.0 THEN 'Warning'
        ELSE 'Normal'
    END AS AlertLevel
FROM CurrentLoad cl
INNER JOIN HistoricalBaseline hb ON cl.LocationName = hb.LocationName;
GO
```

### Fleet Maintenance Triage Scoring

```sql
CREATE PROCEDURE CalculateFleetTriageScore
AS
BEGIN
    UPDATE f
    SET MaintenanceScore = 
        -- Weight by revenue impact
        (100.0 * 
         (ISNULL(RecentIssues.IssueCount, 0) * 2.0 +
          DATEDIFF(DAY, f.LastMaintenanceDate, GETDATE()) / 30.0 +
          ISNULL(RevenueRisk.HighValueTrips, 0) * 5.0) 
        ) / 10.0
    FROM DimFleet f
    LEFT JOIN (
        SELECT 
            FleetKey,
            COUNT(*) AS IssueCount
        FROM FactFleetTrips
        WHERE DelayMinutes > 30
          AND TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY FleetKey
    ) RecentIssues ON f.FleetKey = RecentIssues.FleetKey
    LEFT JOIN (
        SELECT 
            ft.FleetKey,
            COUNT(*) AS HighValueTrips
        FROM FactFleetTrips ft
        INNER JOIN FactWarehouseOperations wo ON ft.OriginGeographyKey = wo.GeographyKey
        INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
        WHERE p.IsPerishable = 1
          AND ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY ft.FleetKey
    ) RevenueRisk ON f.FleetKey = RevenueRisk.FleetKey;
END;
GO
```

## Power BI Integration

### Connection String Configuration

Store in environment or config file:

```json
{
  "server": "your-server.database.windows.net",
  "database": "LogiFleetPulse",
  "authentication": "sql",
  "username": "logifleet_reader",
  "encrypted": true
}
```

Reference via environment variables:

```powerquery
let
    Server = Environment.GetEnvironmentVariable("LOGIFLEET_SQL_SERVER"),
    Database = Environment.GetEnvironmentVariable("LOGIFLEET_SQL_DATABASE"),
    Source = Sql.Database(Server, Database)
in
    Source
```

### DAX Measures for Cross-Fact KPIs

```dax
// Total Dwell Cost (combining warehouse time and fleet idle cost)
Total Dwell Cost = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * 0.50  // $0.50 per minute
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] * 1.20  // $1.20 per minute (fuel + driver)
    )
RETURN WarehouseCost + FleetCost

// Gravity Zone Efficiency
Gravity Zone Efficiency = 
DIVIDE(
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[GravityZoneAssignment] = "High-Gravity" &&
            FactWarehouseOperations[DwellTimeMinutes] < 60
        )
    ),
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[GravityZoneAssignment] = "High-Gravity"
        )
    ),
    0
) * 100

// Fleet Utilization Rate
Fleet Utilization % = 
VAR TotalTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN 
DIVIDE(TotalTime - IdleTime, TotalTime, 0) * 100
```

### Row-Level Security Implementation

```dax
// In Power BI, create role "Regional Manager"
[DimGeography[Region]] = USERNAME()

// In Power BI, create role "Warehouse Supervisor"
[DimGeography[LocationName]] IN 
    LOOKUPVALUE(
        UserLocations[LocationName],
        UserLocations[Username], USERNAME()
    )
```

## Configuration

### Environment Variables

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="logifleet-prod.database.windows.net"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USERNAME="logifleet_app"
export LOGIFLEET_SQL_PASSWORD="<stored-in-key-vault>"

# Data refresh schedule (Power BI Service)
export LOGIFLEET_REFRESH_INTERVAL="15"  # minutes

# Alert thresholds
export LOGIFLEET_ALERT_DWELL_THRESHOLD="180"  # minutes
export LOGIFLEET_ALERT_IDLE_THRESHOLD="15"    # percent of trip
export LOGIFLEET_ALERT_EMAIL="ops-team@company.com"
```

### Scheduled Alert Job

```sql
CREATE PROCEDURE SendDwellAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    SELECT @AlertMessage = STRING_AGG(
        CONCAT(
            'Location: ', g.LocationName, 
            ' | SKU: ', p.SKU,
            ' | Dwell Time: ', wo.DwellTimeMinutes, ' min'
        ),
        CHAR(13) + CHAR(10)
    )
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE wo.DwellTimeMinutes > 180
      AND t.DateTime >= DATEADD(HOUR, -1, GETDATE());
    
    IF @AlertMessage IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'ops-team@company.com',
            @subject = 'ALERT: Extended Dwell Times Detected',
            @body = @AlertMessage;
    END
END;
GO

-- Schedule via SQL Server Agent (daily at business hours)
```

## Common Patterns

### Pattern 1: Gravity Zone Rebalancing

```sql
-- Identify products that need zone reassignment
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.GravityScore AS CurrentScore,
        COUNT(*) AS OperationCount,
        AVG(wo.DwellTimeMinutes) AS AvgDwell
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
      AND wo.OperationType IN ('Picking', 'Packing')
    GROUP BY p.ProductKey, p.SKU, p.GravityScore
)
SELECT 
    SKU,
    CurrentScore,
    OperationCount,
    AvgDwell,
    -- Recalculate gravity (velocity * urgency)
    (OperationCount / 30.0) * (1 / NULLIF(AvgDwell, 0)) * 1000 AS NewGravityScore,
    CASE 
        WHEN (OperationCount / 30.0) * (1 / NULLIF(AvgDwell, 0)) * 1000 > 8.0 THEN 'High-Gravity'
        WHEN (OperationCount / 30.0) * (1 / NULLIF(AvgDwell, 0)) * 1000 > 5.0 THEN 'Medium-Gravity'
        ELSE 'Low-Gravity'
    END AS RecommendedZone
FROM ProductVelocity
WHERE ABS((OperationCount / 30.0) * (1 / NULLIF(AvgDwell, 0)) * 1000 - CurrentScore) > 2.0
ORDER BY OperationCount DESC;
```

### Pattern 2: Route Optimization Analysis

```sql
-- Find optimal delivery routes based on historical fuel efficiency
WITH RoutePerformance AS (
    SELECT 
        og.LocationName AS Origin,
        dg.LocationName AS Destination,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.TripDistanceKm, 0)) AS AvgFuelPerKm,
        AVG(ft.IdleTimeMinutes) AS AvgIdle,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    INNER JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
    INNER JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -90, GETDATE())
    GROUP BY og.LocationName, dg.LocationName
    HAVING COUNT(*) >= 5  -- Minimum sample size
)
SELECT 
    Origin,
    Destination,
    AvgFuelPerKm,
    AvgIdle,
    TripCount,
    -- Efficiency score (lower is better)
    (AvgFuelPerKm * 100) + (AvgIdle / 10.0) AS EfficiencyScore
FROM RoutePerformance
ORDER BY Origin, EfficiencyScore;
```

### Pattern 3: Temporal Elasticity Simulation

```sql
-- Simulate impact of capacity changes on dwell time
DECLARE @CapacityIncrease DECIMAL(5,2) = 0.20;  -- 20% increase

WITH CurrentState AS (
    SELECT 
        g.LocationName,
        COUNT(*) AS CurrentOperations,
        AVG(wo.DwellTimeMinutes) AS CurrentAvgDwell
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY g.LocationName
)
SELECT 
    LocationName,
    CurrentOperations,
    CurrentAvgDwell,
    -- Projected operations with capacity increase
    CurrentOperations * (1 + @CapacityIncrease) AS ProjectedOperations,
    -- Projected dwell (simplified linear model)
    CurrentAvgDwell * (1 + (@CapacityIncrease * 0.3)) AS ProjectedAvgDwell,
    -- Projected cost increase
    (CurrentAvgDwell * (1 + (@CapacityIncrease * 0.3)) * CurrentOperations * (1 + @CapacityIncrease) * 0.50) -
    (CurrentAvgDwell * CurrentOperations * 0.50) AS CostDelta
FROM CurrentState
ORDER BY CostDelta DESC;
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution:** Ensure proper indexing and consider columnstore indexes for fact tables:

```sql
-- Add columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Analytics 
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, DwellTimeMinutes);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Analytics 
ON FactFleetTrips (TimeKey, FleetKey, OriginGeographyKey, IdleTimeMinutes);
```

### Issue: Power BI Refresh Timeout

**Solution:** Implement incremental refresh in Power BI:

1. Add RangeStart and RangeEnd parameters in Power Query
2. Filter fact tables by date range:

```powerquery
let
    Source = Sql.Database(Server, Database),
    FilteredData = Table.SelectRows(Source, 
        each [DateTime] >= RangeStart and [DateTime] < RangeEnd)
in
    FilteredData
```

3. Configure incremental refresh policy in Power BI Desktop

### Issue: Duplicate Records in Fact Tables

**Solution:** Add unique constraints and implement upsert logic:

```sql
-- Add unique constraint
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT UQ_WarehouseOp UNIQUE (TimeKey, GeographyKey, ProductKey, EmployeeID, OperationType);

-- Upsert procedure
CREATE PROCEDURE UpsertWarehouseOperations
AS
BEGIN
    MERGE FactWarehouseOperations AS target
    USING StagingWarehouseOperations AS source
    ON (target.TimeKey = CAST(FORMAT(source.OperationDateTime, 'yyyyMMddHHmm') AS INT)
        AND target.GeographyKey = (SELECT GeographyKey FROM DimGeography WHERE LocationID = source.LocationID)
        AND target.ProductKey = (SELECT ProductKey FROM DimProduct WHERE SKU = source.SKU)
        AND target.EmployeeID = source.EmployeeID
        AND target.OperationType = source.OperationType)
    WHEN MATCHED THEN
        UPDATE SET 
            Quantity = source.Quantity,
            DwellTimeMinutes = DATEDIFF(MINUTE, source.StartTime, source.EndTime)
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, GeographyKey, ProductKey, OperationType, Quantity, DwellTimeMinutes, EmployeeID)
        VALUES (
            CAST(FORMAT(source.OperationDateTime, 'yyyyMMddHHmm') AS INT),
            (SELECT GeographyKey FROM DimGeography WHERE LocationID = source.LocationID
