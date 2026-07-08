---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for warehouse operations, fleet telemetry, and supply chain KPI harmonization
triggers:
  - "set up LogiFleet Pulse analytics platform"
  - "deploy the supply chain data warehouse schema"
  - "configure Power BI logistics dashboard"
  - "create warehouse gravity zone analysis"
  - "implement cross-fact KPI queries for fleet and inventory"
  - "build real-time logistics intelligence dashboards"
  - "optimize warehouse operations with dimensional modeling"
  - "analyze fleet telemetry and dwell time correlations"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers implement LogiFleet Pulse, a comprehensive logistics intelligence platform that combines warehouse operations, fleet telemetry, and inventory analytics using MS SQL Server data warehousing and Power BI visualization. The platform uses a multi-fact star schema architecture to harmonize cross-modal supply chain KPIs.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that:

- Integrates warehouse operations (receiving, putaway, picking, packing, shipping) with fleet telemetry (GPS, fuel consumption, driver behavior)
- Uses a custom multi-fact star schema with time-phased dimensions for cross-KPI analysis
- Provides real-time dashboards (15-minute refresh intervals) for operational awareness
- Implements "Warehouse Gravity Zones" — spatial optimization based on pick frequency, item value, and fragility
- Generates predictive bottleneck detection and proactive fleet maintenance prioritization
- Supports role-based access control with row-level security

**Core Architecture:**
- **Database**: MS SQL Server 2019+ (transactional integrity, partitioning, incremental loading)
- **Visualization**: Power BI (adaptive dashboards, natural language queries)
- **Data Model**: Multi-fact star schema with shared time/geography/product dimensions

## Installation & Deployment

### Prerequisites

```bash
# Required software
- MS SQL Server 2019 or later
- SQL Server Management Studio (SSMS) 18+
- Power BI Desktop (latest version)
- Windows Server 2019+ or Azure SQL Database
```

### Step 1: Deploy SQL Server Schema

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
ON PRIMARY 
(
    NAME = LogiFleetPulse_Data,
    FILENAME = 'C:\SQLData\LogiFleetPulse.mdf',
    SIZE = 10GB,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = LogiFleetPulse_Log,
    FILENAME = 'C:\SQLData\LogiFleetPulse.ldf',
    SIZE = 2GB,
    FILEGROWTH = 512MB
);
GO

USE LogiFleetPulse;
GO

-- Create schemas for organization
CREATE SCHEMA Warehouse;
GO
CREATE SCHEMA Fleet;
GO
CREATE SCHEMA Dimension;
GO
CREATE SCHEMA Bridge;
GO
```

### Step 2: Create Core Dimension Tables

```sql
-- DimTime: 15-minute granularity time dimension
CREATE TABLE Dimension.DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName NVARCHAR(20) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    MonthName NVARCHAR(20) NOT NULL,
    Quarter TINYINT NOT NULL,
    Year SMALLINT NOT NULL,
    FiscalPeriod NVARCHAR(10) NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0,
    CONSTRAINT UQ_DimTime_FullDateTime UNIQUE (FullDateTime)
);
GO

CREATE NONCLUSTERED INDEX IX_DimTime_DateKey ON Dimension.DimTime(DateKey);
CREATE NONCLUSTERED INDEX IX_DimTime_FiscalPeriod ON Dimension.DimTime(FiscalPeriod);
GO

-- DimGeography: Hierarchical location dimension
CREATE TABLE Dimension.DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, RouteNode, CustomerSite
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Country NVARCHAR(100) NOT NULL,
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    ValidFrom DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    ValidTo DATETIME2 NOT NULL DEFAULT '9999-12-31',
    IsCurrent BIT NOT NULL DEFAULT 1
);
GO

CREATE NONCLUSTERED INDEX IX_DimGeography_LocationID ON Dimension.DimGeography(LocationID);
CREATE NONCLUSTERED INDEX IX_DimGeography_LocationType ON Dimension.DimGeography(LocationType);
GO

-- DimProduct: Product hierarchy with gravity scoring
CREATE TABLE Dimension.DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    Brand NVARCHAR(100),
    UnitCost DECIMAL(18,2),
    UnitPrice DECIMAL(18,2),
    Weight DECIMAL(10,3), -- kg
    Volume DECIMAL(10,3), -- cubic meters
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility factor
    OptimalZone NVARCHAR(50), -- Fast-Moving, Medium, Slow, Cold-Storage
    ValidFrom DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    ValidTo DATETIME2 NOT NULL DEFAULT '9999-12-31',
    IsCurrent BIT NOT NULL DEFAULT 1
);
GO

CREATE UNIQUE NONCLUSTERED INDEX IX_DimProduct_SKU_Current 
ON Dimension.DimProduct(SKU) WHERE IsCurrent = 1;
GO
```

### Step 3: Create Fact Tables

```sql
-- FactWarehouseOperations: Core warehouse activity tracking
CREATE TABLE Warehouse.FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    OperationID NVARCHAR(50) NOT NULL,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimProduct(ProductKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimGeography(GeographyKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    ZoneID NVARCHAR(50),
    QuantityHandled INT NOT NULL,
    DurationMinutes DECIMAL(10,2),
    LaborHours DECIMAL(10,2),
    EquipmentUsedID NVARCHAR(50),
    DwellTimeHours DECIMAL(10,2), -- Time between operations
    DistanceTraveled DECIMAL(10,2), -- meters within warehouse
    ErrorCount INT DEFAULT 0,
    ReworkRequired BIT DEFAULT 0,
    BatchID NVARCHAR(50),
    CreatedAt DATETIME2 DEFAULT SYSDATETIME()
);
GO

-- Partitioning by month for performance
CREATE PARTITION FUNCTION PF_WarehouseOps_Month (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401, 20260501, 20260601);
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeProduct 
ON Warehouse.FactWarehouseOperations(TimeKey, ProductKey);
GO

-- FactFleetTrips: Fleet telemetry and route tracking
CREATE TABLE Fleet.FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID NVARCHAR(50) NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DepartureTimeKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimTime(TimeKey),
    ArrivalTimeKey INT FOREIGN KEY REFERENCES Dimension.DimTime(TimeKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimGeography(GeographyKey),
    PlannedDistance DECIMAL(10,2), -- km
    ActualDistance DECIMAL(10,2),
    FuelConsumed DECIMAL(10,2), -- liters
    FuelCost DECIMAL(18,2),
    TripDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    PayloadWeight DECIMAL(10,2), -- kg
    DeliveryOnTime BIT,
    DelayReasonCode NVARCHAR(50),
    WeatherCondition NVARCHAR(50),
    TrafficSeverity NVARCHAR(20),
    MaintenanceFlags NVARCHAR(200), -- JSON array of alerts
    CreatedAtDATETIME2 DEFAULT SYSDATETIME()
);
GO

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Vehicle_Time 
ON Fleet.FactFleetTrips(VehicleID, DepartureTimeKey);
GO

-- FactCrossDock: Transfer operations without long-term storage
CREATE TABLE Warehouse.FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TransferID NVARCHAR(50) NOT NULL,
    InboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimTime(TimeKey),
    OutboundTimeKey INT FOREIGN KEY REFERENCES Dimension.DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimProduct(ProductKey),
    SourceWarehouseKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimGeography(GeographyKey),
    DestinationWarehouseKey INT NOT NULL FOREIGN KEY REFERENCES Dimension.DimGeography(GeographyKey),
    QuantityTransferred INT NOT NULL,
    DockDwellMinutes DECIMAL(10,2),
    TransferCost DECIMAL(18,2),
    CreatedAt DATETIME2 DEFAULT SYSDATETIME()
);
GO
```

### Step 4: Create Bridge Tables for Many-to-Many Relationships

```sql
-- Bridge table: Route to Storage Zone
CREATE TABLE Bridge.RouteZoneAssignment (
    RouteID NVARCHAR(50) NOT NULL,
    ZoneID NVARCHAR(50) NOT NULL,
    AllocationPercentage DECIMAL(5,2),
    EffectiveDate DATE NOT NULL,
    PRIMARY KEY (RouteID, ZoneID, EffectiveDate)
);
GO
```

## Key SQL Queries & Patterns

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Correlate warehouse dwell time with fleet idle time for cost optimization
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        w.WarehouseKey,
        AVG(w.DwellTimeHours) AS AvgDwellHours,
        SUM(w.QuantityHandled) AS TotalQuantity
    FROM Warehouse.FactWarehouseOperations w
    INNER JOIN Dimension.DimProduct p ON w.ProductKey = p.ProductKey
    INNER JOIN Dimension.DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
        AND w.OperationType = 'Picking'
    GROUP BY p.SKU, p.ProductName, w.WarehouseKey
),
FleetIdle AS (
    SELECT 
        f.OriginKey AS WarehouseKey,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(f.FuelCost * (f.IdleTimeMinutes / NULLIF(f.TripDurationMinutes, 0))) AS AvgIdleCost
    FROM Fleet.FactFleetTrips f
    INNER JOIN Dimension.DimTime t ON f.DepartureTimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY f.OriginKey
)
SELECT 
    wd.SKU,
    wd.ProductName,
    g.LocationName AS Warehouse,
    wd.AvgDwellHours,
    fi.AvgIdleMinutes,
    fi.AvgIdleCost,
    (wd.AvgDwellHours * 60 / NULLIF(fi.AvgIdleMinutes, 0)) AS DwellToIdleRatio,
    CASE 
        WHEN wd.AvgDwellHours > 72 AND fi.AvgIdleMinutes > 30 THEN 'High Priority'
        WHEN wd.AvgDwellHours > 48 THEN 'Medium Priority'
        ELSE 'Low Priority'
    END AS OptimizationPriority
FROM WarehouseDwell wd
INNER JOIN Dimension.DimGeography g ON wd.WarehouseKey = g.GeographyKey
LEFT JOIN FleetIdle fi ON wd.WarehouseKey = fi.WarehouseKey
ORDER BY DwellToIdleRatio DESC;
```

### Warehouse Gravity Zone Analysis

```sql
-- Calculate optimal gravity zones based on pick frequency and product value
WITH ProductVelocity AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.UnitPrice,
        p.IsFragile,
        COUNT(*) AS PickCount,
        SUM(w.QuantityHandled) AS TotalPicked,
        AVG(w.DurationMinutes) AS AvgPickTime
    FROM Warehouse.FactWarehouseOperations w
    INNER JOIN Dimension.DimProduct p ON w.ProductKey = p.ProductKey
    INNER JOIN Dimension.DimTime t ON w.TimeKey = t.TimeKey
    WHERE w.OperationType = 'Picking'
        AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.UnitPrice, p.IsFragile
)
SELECT 
    SKU,
    ProductName,
    TotalPicked,
    UnitPrice,
    IsFragile,
    -- Gravity Score = (Velocity * Value * Fragility Factor)
    (TotalPicked * UnitPrice * (CASE WHEN IsFragile = 1 THEN 1.5 ELSE 1.0 END)) AS CalculatedGravityScore,
    CASE 
        WHEN (TotalPicked * UnitPrice) > 100000 THEN 'Zone-A-HighGravity'
        WHEN (TotalPicked * UnitPrice) > 50000 THEN 'Zone-B-MediumGravity'
        ELSE 'Zone-C-LowGravity'
    END AS RecommendedZone,
    AvgPickTime
FROM ProductVelocity
ORDER BY CalculatedGravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks using moving average and standard deviation
WITH HourlyMetrics AS (
    SELECT 
        t.DateKey,
        t.HourOfDay,
        w.ZoneID,
        COUNT(*) AS OperationCount,
        AVG(w.DurationMinutes) AS AvgDuration,
        STDEV(w.DurationMinutes) AS StdDevDuration
    FROM Warehouse.FactWarehouseOperations w
    INNER JOIN Dimension.DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -14, GETDATE())
    GROUP BY t.DateKey, t.HourOfDay, w.ZoneID
),
ThresholdCalculation AS (
    SELECT 
        ZoneID,
        AVG(AvgDuration) AS BaselineAvg,
        AVG(StdDevDuration) AS BaselineStdDev
    FROM HourlyMetrics
    GROUP BY ZoneID
)
SELECT 
    hm.DateKey,
    hm.HourOfDay,
    hm.ZoneID,
    hm.OperationCount,
    hm.AvgDuration,
    tc.BaselineAvg,
    tc.BaselineStdDev,
    (hm.AvgDuration - tc.BaselineAvg) / NULLIF(tc.BaselineStdDev, 0) AS ZScore,
    CASE 
        WHEN (hm.AvgDuration - tc.BaselineAvg) / NULLIF(tc.BaselineStdDev, 0) > 2 THEN 'CRITICAL'
        WHEN (hm.AvgDuration - tc.BaselineAvg) / NULLIF(tc.BaselineStdDev, 0) > 1 THEN 'WARNING'
        ELSE 'NORMAL'
    END AS BottleneckAlert
FROM HourlyMetrics hm
INNER JOIN ThresholdCalculation tc ON hm.ZoneID = tc.ZoneID
WHERE (hm.AvgDuration - tc.BaselineAvg) / NULLIF(tc.BaselineStdDev, 0) > 1
ORDER BY ZScore DESC;
```

### Fleet Maintenance Prioritization

```sql
-- Adaptive fleet triage using weighted scoring
WITH FleetHealthScore AS (
    SELECT 
        VehicleID,
        COUNT(*) AS TripCount,
        AVG(FuelConsumed / NULLIF(ActualDistance, 0)) AS AvgFuelEfficiency,
        SUM(CASE WHEN DelayReasonCode IS NOT NULL THEN 1 ELSE 0 END) AS DelayCount,
        MAX(TripDurationMinutes) AS MaxTripDuration,
        SUM(IdleTimeMinutes) / NULLIF(SUM(TripDurationMinutes), 0) AS IdleRatio,
        -- Parse maintenance flags (assuming JSON format)
        COUNT(CASE WHEN MaintenanceFlags LIKE '%tire_pressure%' THEN 1 END) AS TirePressureAlerts,
        COUNT(CASE WHEN MaintenanceFlags LIKE '%engine_diagnostic%' THEN 1 END) AS EngineAlerts
    FROM Fleet.FactFleetTrips
    WHERE DepartureTimeKey IN (
        SELECT TimeKey FROM Dimension.DimTime 
        WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE())
    )
    GROUP BY VehicleID
)
SELECT 
    VehicleID,
    TripCount,
    AvgFuelEfficiency,
    DelayCount,
    IdleRatio,
    TirePressureAlerts,
    EngineAlerts,
    -- Weighted health score (lower is worse)
    (
        (AvgFuelEfficiency * 10) + 
        (DelayCount * -5) + 
        (IdleRatio * -20) + 
        (TirePressureAlerts * -3) + 
        (EngineAlerts * -10)
    ) AS HealthScore,
    CASE 
        WHEN EngineAlerts > 2 THEN 'URGENT - Engine Issues'
        WHEN TirePressureAlerts > 3 THEN 'HIGH - Tire Maintenance'
        WHEN DelayCount > 5 THEN 'MEDIUM - Performance Review'
        ELSE 'LOW - Routine Check'
    END AS MaintenancePriority
FROM FleetHealthScore
ORDER BY HealthScore ASC;
```

## Power BI Configuration

### Connection Setup

```powerquery
// Power Query M connection string
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse",
        [
            Query = null,
            CreateNavigationProperties = false,
            CommandTimeout = #duration(0, 0, 5, 0)
        ]
    )
in
    Source
```

### Create Calculated Measures

```dax
// Total Warehouse Operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Average Dwell Time
AvgDwellTime = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    FactWarehouseOperations[OperationType] = "Picking"
)

// Fleet Utilization Rate
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
)

// Cross-Fact KPI: Cost per Unit Moved
CostPerUnit = 
DIVIDE(
    SUM(FactFleetTrips[FuelCost]) + SUM(FactWarehouseOperations[LaborHours]) * 25,
    SUM(FactWarehouseOperations[QuantityHandled]),
    0
)

// Gravity Zone Performance
GravityZoneEfficiency = 
VAR HighGravityPicks = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        DimProduct[OptimalZone] = "Zone-A-HighGravity"
    )
VAR TotalPicks = SUM(FactWarehouseOperations[QuantityHandled])
RETURN DIVIDE(HighGravityPicks, TotalPicks, 0)

// Time Intelligence: YoY Comparison
OperationsYoY = 
VAR CurrentPeriod = [TotalOperations]
VAR PriorPeriod = 
    CALCULATE(
        [TotalOperations],
        SAMEPERIODLASTYEAR(DimTime[FullDateTime])
    )
RETURN DIVIDE(CurrentPeriod - PriorPeriod, PriorPeriod, 0)
```

### Row-Level Security Setup

```dax
// RLS for warehouse managers (see only their warehouse)
[WarehouseKey] = LOOKUPVALUE(
    Users[AssignedWarehouseKey],
    Users[EmailAddress],
    USERPRINCIPALNAME()
)

// RLS for regional managers (see all warehouses in their region)
[Region] IN 
CALCULATETABLE(
    VALUES(DimGeography[Region]),
    Users[EmailAddress] = USERPRINCIPALNAME()
)
```

## Stored Procedures for Automation

### Incremental Data Load

```sql
CREATE PROCEDURE Warehouse.usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO Warehouse.FactWarehouseOperations (
        OperationID, TimeKey, ProductKey, WarehouseKey,
        OperationType, ZoneID, QuantityHandled, DurationMinutes,
        LaborHours, EquipmentUsedID, DwellTimeHours, DistanceTraveled,
        ErrorCount, ReworkRequired, BatchID
    )
    SELECT 
        s.OperationID,
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        s.ZoneID,
        s.QuantityHandled,
        s.DurationMinutes,
        s.LaborHours,
        s.EquipmentUsedID,
        s.DwellTimeHours,
        s.DistanceTraveled,
        s.ErrorCount,
        s.ReworkRequired,
        s.BatchID
    FROM Staging.WarehouseOperations s
    INNER JOIN Dimension.DimTime t ON s.OperationDateTime = t.FullDateTime
    INNER JOIN Dimension.DimProduct p ON s.SKU = p.SKU AND p.IsCurrent = 1
    INNER JOIN Dimension.DimGeography g ON s.LocationID = g.LocationID AND g.IsCurrent = 1
    WHERE s.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM Warehouse.FactWarehouseOperations w 
            WHERE w.OperationID = s.OperationID
        );
        
    -- Update last load timestamp
    UPDATE ETL.LoadControl
    SET LastLoadDateTime = SYSDATETIME()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Gravity Score Recalculation

```sql
CREATE PROCEDURE Dimension.usp_RecalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            p.UnitPrice,
            p.IsFragile,
            COUNT(*) AS PickFrequency,
            SUM(w.QuantityHandled) AS TotalVolume,
            AVG(w.DwellTimeHours) AS AvgDwell
        FROM Dimension.DimProduct p
        INNER JOIN Warehouse.FactWarehouseOperations w ON p.ProductKey = w.ProductKey
        INNER JOIN Dimension.DimTime t ON w.TimeKey = t.TimeKey
        WHERE p.IsCurrent = 1
            AND w.OperationType = 'Picking'
            AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY p.ProductKey, p.SKU, p.UnitPrice, p.IsFragile
    )
    UPDATE p
    SET 
        p.GravityScore = (
            pm.TotalVolume * 
            p.UnitPrice * 
            (CASE WHEN p.IsFragile = 1 THEN 1.5 ELSE 1.0 END) *
            (1 / NULLIF(pm.AvgDwell, 0))
        ),
        p.OptimalZone = CASE 
            WHEN (pm.TotalVolume * p.UnitPrice) > 100000 THEN 'Zone-A-HighGravity'
            WHEN (pm.TotalVolume * p.UnitPrice) > 50000 THEN 'Zone-B-MediumGravity'
            ELSE 'Zone-C-LowGravity'
        END
    FROM Dimension.DimProduct p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey
    WHERE p.IsCurrent = 1;
END;
GO
```

### Automated Alerting

```sql
CREATE PROCEDURE Alerts.usp_CheckBottleneckThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Detect zones exceeding threshold
    DECLARE @AlertResults TABLE (
        ZoneID NVARCHAR(50),
        CurrentAvgDuration DECIMAL(10,2),
        ThresholdDuration DECIMAL(10,2),
        Severity NVARCHAR(20)
    );
    
    INSERT INTO @AlertResults
    SELECT 
        w.ZoneID,
        AVG(w.DurationMinutes) AS CurrentAvgDuration,
        z.ThresholdMinutes,
        CASE 
            WHEN AVG(w.DurationMinutes) > z.ThresholdMinutes * 1.5 THEN 'CRITICAL'
            WHEN AVG(w.DurationMinutes) > z.ThresholdMinutes THEN 'WARNING'
        END AS Severity
    FROM Warehouse.FactWarehouseOperations w
    INNER JOIN Dimension.DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN Config.ZoneThresholds z ON w.ZoneID = z.ZoneID
    WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
    GROUP BY w.ZoneID, z.ThresholdMinutes
    HAVING AVG(w.DurationMinutes) > z.ThresholdMinutes;
    
    -- Send alerts (integrate with email/Teams webhook)
    IF EXISTS (SELECT 1 FROM @AlertResults WHERE Severity = 'CRITICAL')
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = SYSTEM_USER + '@ALERT_EMAIL_DOMAIN',
            @subject = 'CRITICAL: Warehouse Bottleneck Detected',
            @body = 'Check Power BI dashboard for details',
            @body_format = 'HTML';
    END;
END;
GO
```

## Configuration Files

### Environment Variables

```bash
# Database connection
SQL_SERVER_HOST=your-server.database.windows.net
SQL_SERVER_DATABASE=LogiFleetPulse
SQL_SERVER_USER=logifleet_app
SQL_SERVER_PASSWORD=env:DB_PASSWORD

# Power BI Service
POWERBI_WORKSPACE_ID=your-workspace-guid
POWERBI_DATASET_ID=your-dataset-guid

# External API integration
WEATHER_API_KEY=env:WEATHER_API_KEY
TRAFFIC_API_KEY=env:TRAFFIC_API_KEY

# Alert configuration
ALERT_EMAIL_DOMAIN=yourcompany.com
TEAMS_WEBHOOK_URL=env:TEAMS_WEBHOOK_URL
```

### Sample Data Population

```sql
-- Populate DimTime (15-minute granularity for one year)
DECLARE @StartDate DATETIME2 = '2026-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2026-12-31 23:45:00';

WHILE @StartDate <= @EndDate
BEGIN
    INSERT INTO Dimension.DimTime (
        FullDateTime, DateKey, TimeSlot, HourOfDay, DayOfWeek, DayName,
        WeekOfYear, MonthNumber, MonthName, Quarter, Year, FiscalPeriod, IsWeekend
    )
    VALUES (
        @StartDate,
        CONVERT(INT, FORMAT(@StartDate, 'yyyyMMdd')),
        CONVERT(TIME, @StartDate),
        DATEPART(HOUR,
