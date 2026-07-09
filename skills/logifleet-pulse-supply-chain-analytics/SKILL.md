---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics analytics with multi-fact star schema, fleet telemetry, and warehouse operations intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement multi-fact star schema for warehouse data"
  - "create fleet telemetry data model"
  - "build supply chain KPI dashboard"
  - "integrate warehouse management system with SQL Server"
  - "deploy logistics intelligence platform"
  - "optimize warehouse gravity zones in Power BI"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced data warehousing and analytics platform for logistics and supply chain management. It implements a multi-fact star schema in MS SQL Server that harmonizes warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer visualized through Power BI.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time dashboard refreshes (15-minute granularity)
- Warehouse gravity zone optimization
- Predictive bottleneck detection
- Fleet maintenance triage engine
- Role-based access control with row-level security

## Architecture Components

### Core Fact Tables
- `FactWarehouseOperations` - putaway cycles, pick rates, packing times, dwell spans
- `FactFleetTrips` - route segments, fuel consumption, idle time, loading events
- `FactCrossDock` - transfers between inbound/outbound without storage

### Dimension Tables
- `DimTime` - 15-minute buckets, hour-of-day, day-of-week, fiscal periods
- `DimGeography` - hierarchical (continent → country → region → warehouse/route node)
- `DimProductGravity` - velocity, value, and fragility-based scoring
- `DimSupplierReliability` - lead time variance, defect rates, compliance scores

## Installation & Setup

### Prerequisites
- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)

### 1. Deploy SQL Schema

```sql
-- Create database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create fact table for warehouse operations
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    QuantityHandled INT,
    OperationCost DECIMAL(10,2),
    EmployeeID INT,
    EquipmentID INT,
    RecordedAt DATETIME2 DEFAULT GETUTCDATE()
)
GO

-- Create clustered columnstore index for analytics
CREATE CLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOperations 
ON FactWarehouseOperations
GO

-- Create fact table for fleet trips
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    TripCost DECIMAL(10,2),
    OnTimeDelivery BIT,
    RecordedAt DATETIME2 DEFAULT GETUTCDATE()
)
GO

CREATE CLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips 
ON FactFleetTrips
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    Day INT,
    Hour INT,
    Minute15Block INT, -- 0, 15, 30, 45
    DayOfWeek INT,
    DayName VARCHAR(20),
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalYear INT,
    FiscalQuarter INT
)
GO

-- Create geography dimension with hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'WAREHOUSE', 'DISTRIBUTION_CENTER', 'ROUTE_NODE'
    Address VARCHAR(500),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1
)
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductID VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    SKU VARCHAR(100),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(10,2),
    FragilityScore INT, -- 1-10 scale
    VelocityScore INT, -- 1-10 scale (picks per day)
    GravityScore AS (VelocityScore * UnitValue / NULLIF(FragilityScore, 0)), -- Computed
    RecommendedZone VARCHAR(50), -- 'HIGH_GRAVITY', 'MEDIUM_GRAVITY', 'LOW_GRAVITY'
    IsActive BIT DEFAULT 1
)
GO

-- Create indexes for dimension lookups
CREATE NONCLUSTERED INDEX IX_DimGeography_LocationID 
ON DimGeography(LocationID) INCLUDE (GeographyKey)
GO

CREATE NONCLUSTERED INDEX IX_DimProductGravity_ProductID 
ON DimProductGravity(ProductID) INCLUDE (ProductKey)
GO
```

### 2. Populate Time Dimension

```sql
-- Stored procedure to populate time dimension
CREATE OR ALTER PROCEDURE PopulateTimeDimension
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        -- Round to nearest 15-minute block
        DECLARE @Rounded DATETIME2 = DATEADD(MINUTE, 
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15, 
            DATEADD(HOUR, DATEDIFF(HOUR, 0, @CurrentDateTime), 0));
        
        DECLARE @TimeKey INT = 
            YEAR(@Rounded) * 100000000 +
            MONTH(@Rounded) * 1000000 +
            DAY(@Rounded) * 10000 +
            DATEPART(HOUR, @Rounded) * 100 +
            (DATEPART(MINUTE, @Rounded) / 15) * 15;
        
        IF NOT EXISTS (SELECT 1 FROM DimTime WHERE TimeKey = @TimeKey)
        BEGIN
            INSERT INTO DimTime (
                TimeKey, DateTime, Year, Quarter, Month, Week, Day, 
                Hour, Minute15Block, DayOfWeek, DayName, IsWeekend, IsHoliday
            )
            VALUES (
                @TimeKey,
                @Rounded,
                YEAR(@Rounded),
                DATEPART(QUARTER, @Rounded),
                MONTH(@Rounded),
                DATEPART(WEEK, @Rounded),
                DAY(@Rounded),
                DATEPART(HOUR, @Rounded),
                (DATEPART(MINUTE, @Rounded) / 15) * 15,
                DATEPART(WEEKDAY, @Rounded),
                DATENAME(WEEKDAY, @Rounded),
                CASE WHEN DATEPART(WEEKDAY, @Rounded) IN (1, 7) THEN 1 ELSE 0 END,
                0 -- Holiday flag needs external calendar
            );
        END
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

-- Execute to populate for 2 years
EXEC PopulateTimeDimension '2025-01-01', '2026-12-31';
GO
```

### 3. Create Cross-Fact KPI Views

```sql
-- View: Warehouse efficiency vs Fleet utilization
CREATE OR ALTER VIEW vw_WarehouseFleetEfficiency AS
SELECT 
    t.Year,
    t.Quarter,
    t.Month,
    g.Country,
    g.StateProvince,
    
    -- Warehouse metrics
    COUNT(DISTINCT w.OperationID) AS TotalWarehouseOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(w.QuantityHandled) AS TotalQuantityHandled,
    SUM(w.OperationCost) AS TotalWarehouseCost,
    
    -- Fleet metrics
    COUNT(DISTINCT f.TripID) AS TotalFleetTrips,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    SUM(f.FuelConsumedGallons) AS TotalFuelConsumed,
    SUM(f.TripCost) AS TotalFleetCost,
    
    -- Cross-fact KPI
    CASE 
        WHEN AVG(w.DwellTimeMinutes) > 0 
        THEN AVG(f.IdleTimeMinutes) / AVG(w.DwellTimeMinutes)
        ELSE NULL 
    END AS IdleDwellRatio,
    
    SUM(w.OperationCost) + SUM(f.TripCost) AS TotalLogisticsCost

FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
LEFT JOIN FactFleetTrips f ON t.TimeKey = f.TimeKey 
    AND g.GeographyKey = f.OriginGeographyKey

GROUP BY t.Year, t.Quarter, t.Month, g.Country, g.StateProvince
GO
```

### 4. Implement Incremental Data Loading

```sql
-- Stored procedure for incremental warehouse operations load
CREATE OR ALTER PROCEDURE LoadWarehouseOperationsIncremental
    @SourceConnectionString VARCHAR(500) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last loaded timestamp
    DECLARE @LastLoadTime DATETIME2;
    SELECT @LastLoadTime = ISNULL(MAX(RecordedAt), '1900-01-01')
    FROM FactWarehouseOperations;
    
    -- Staging table for new data
    CREATE TABLE #StagingWarehouseOps (
        OperationType VARCHAR(20),
        ProductID VARCHAR(50),
        WarehouseID VARCHAR(50),
        DwellTimeMinutes INT,
        QuantityHandled INT,
        OperationCost DECIMAL(10,2),
        OperationDateTime DATETIME2
    );
    
    -- Example: Insert from external source or linked server
    -- BULK INSERT or OPENROWSET would be used in production
    -- INSERT INTO #StagingWarehouseOps SELECT * FROM OPENROWSET(...)
    
    -- Transform and load into fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        ProductKey,
        WarehouseKey,
        OperationType,
        DwellTimeMinutes,
        QuantityHandled,
        OperationCost,
        RecordedAt
    )
    SELECT 
        YEAR(s.OperationDateTime) * 100000000 +
        MONTH(s.OperationDateTime) * 1000000 +
        DAY(s.OperationDateTime) * 10000 +
        DATEPART(HOUR, s.OperationDateTime) * 100 +
        (DATEPART(MINUTE, s.OperationDateTime) / 15) * 15 AS TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        s.DwellTimeMinutes,
        s.QuantityHandled,
        s.OperationCost,
        s.OperationDateTime
    FROM #StagingWarehouseOps s
    INNER JOIN DimProductGravity p ON s.ProductID = p.ProductID
    INNER JOIN DimGeography g ON s.WarehouseID = g.LocationID
    WHERE s.OperationDateTime > @LastLoadTime;
    
    DROP TABLE #StagingWarehouseOps;
    
    SELECT @@ROWCOUNT AS RowsInserted;
END
GO
```

## Power BI Configuration

### 1. Connect to SQL Server

```powerquery
// Power Query M code for connecting to SQL Server
let
    Source = Sql.Database(
        #"SQL_SERVER_HOST", 
        "LogiFleetPulse",
        [
            CreateNavigationProperties = false,
            CommandTimeout = #duration(0, 0, 30, 0)
        ]
    ),
    
    // Load fact tables
    FactWarehouse = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleet = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    
    // Load dimension tables
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data],
    DimProduct = Source{[Schema="dbo",Item="DimProductGravity"]}[Data]
in
    Source
```

### 2. Create DAX Measures

```dax
// Total Logistics Cost (Cross-Fact)
Total Logistics Cost = 
    SUM(FactWarehouseOperations[OperationCost]) + 
    SUM(FactFleetTrips[TripCost])

// Warehouse Efficiency Score
Warehouse Efficiency = 
DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SUM(FactWarehouseOperations[DwellTimeMinutes]),
    0
) * 100

// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DistanceMiles]) - SUM(FactFleetTrips[IdleTimeMinutes]) / 60 * 45,
    SUM(FactFleetTrips[DistanceMiles]),
    0
) * 100

// Gravity Zone Compliance
Gravity Zone Compliance % = 
DIVIDE(
    COUNTROWS(
        FILTER(
            FactWarehouseOperations,
            RELATED(DimProductGravity[GravityScore]) >= 50 && 
            FactWarehouseOperations[DwellTimeMinutes] < 48
        )
    ),
    COUNTROWS(FactWarehouseOperations),
    0
) * 100

// Predictive Bottleneck Index
Bottleneck Risk Index = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR StdDevDwell = STDEV.P(FactWarehouseOperations[DwellTimeMinutes])
RETURN
    (AvgDwell + AvgIdle) / IFERROR(StdDevDwell, 1)

// Time Intelligence - Month Over Month Change
MoM Logistics Cost = 
VAR CurrentMonth = [Total Logistics Cost]
VAR PreviousMonth = 
    CALCULATE(
        [Total Logistics Cost],
        DATEADD(DimTime[DateTime], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth, 0)
```

### 3. Row-Level Security

```dax
// RLS for regional managers (filter geography dimension)
[Country] = USERNAME() || [StateProvince] = LOOKUPVALUE(
    UserRegions[Region],
    UserRegions[Email],
    USERNAME()
)

// RLS for warehouse supervisors (filter by warehouse)
[LocationID] IN VALUES(
    FILTER(
        UserWarehouses,
        UserWarehouses[Email] = USERNAME()
    )[WarehouseID]
)
```

## Common Patterns

### Pattern 1: Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong gravity zones
WITH ProductVelocity AS (
    SELECT 
        w.ProductKey,
        p.ProductName,
        p.GravityScore,
        p.RecommendedZone,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS PickFrequency
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'PICK'
        AND w.RecordedAt >= DATEADD(DAY, -30, GETUTCDATE())
    GROUP BY w.ProductKey, p.ProductName, p.GravityScore, p.RecommendedZone
)
SELECT 
    ProductName,
    GravityScore,
    RecommendedZone,
    AvgDwellTime,
    PickFrequency,
    CASE 
        WHEN GravityScore >= 70 AND AvgDwellTime > 48 THEN 'Move to HIGH_GRAVITY'
        WHEN GravityScore BETWEEN 40 AND 69 AND AvgDwellTime > 72 THEN 'Move to MEDIUM_GRAVITY'
        WHEN GravityScore < 40 AND AvgDwellTime < 24 THEN 'Move to LOW_GRAVITY'
        ELSE 'Correctly Zoned'
    END AS Recommendation
FROM ProductVelocity
WHERE AvgDwellTime IS NOT NULL
ORDER BY GravityScore DESC;
```

### Pattern 2: Fleet Maintenance Triage

```sql
-- Predictive maintenance prioritization
WITH FleetHealth AS (
    SELECT 
        f.VehicleKey,
        v.VehicleID,
        v.VehicleType,
        AVG(f.FuelConsumedGallons / NULLIF(f.DistanceMiles, 0)) AS AvgFuelEfficiency,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(CASE WHEN f.OnTimeDelivery = 0 THEN 1 END) AS LateDeliveries,
        SUM(f.TripCost) AS TotalRevenue
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE f.RecordedAt >= DATEADD(DAY, -60, GETUTCDATE())
    GROUP BY f.VehicleKey, v.VehicleID, v.VehicleType
),
MaintenancePriority AS (
    SELECT 
        *,
        (AvgFuelEfficiency * 100) + 
        (AvgIdleTime * 2) + 
        (LateDeliveries * 50) -
        (TotalRevenue / 1000) AS PriorityScore
    FROM FleetHealth
)
SELECT 
    VehicleID,
    VehicleType,
    ROUND(AvgFuelEfficiency, 2) AS FuelEfficiency_GPM,
    ROUND(AvgIdleTime, 0) AS AvgIdleMinutes,
    LateDeliveries,
    ROUND(TotalRevenue, 2) AS Revenue,
    ROUND(PriorityScore, 0) AS PriorityScore,
    RANK() OVER (ORDER BY PriorityScore DESC) AS MaintenanceRank
FROM MaintenancePriority
ORDER BY PriorityScore DESC;
```

### Pattern 3: Cross-Dock Efficiency Analysis

```sql
-- Analyze cross-dock operations for direct transfers
CREATE OR ALTER VIEW vw_CrossDockEfficiency AS
SELECT 
    t.Year,
    t.Month,
    g.LocationName AS CrossDockFacility,
    COUNT(*) AS TotalTransfers,
    AVG(DATEDIFF(MINUTE, w_in.RecordedAt, w_out.RecordedAt)) AS AvgTransferTimeMinutes,
    SUM(w_in.QuantityHandled) AS TotalUnitsTransferred,
    SUM(w_in.OperationCost + w_out.OperationCost) AS TotalCrossDockCost,
    CAST(SUM(CASE WHEN DATEDIFF(MINUTE, w_in.RecordedAt, w_out.RecordedAt) <= 120 THEN 1 ELSE 0 END) AS FLOAT) 
        / COUNT(*) * 100 AS UnderTwoHourRate
FROM FactWarehouseOperations w_in
INNER JOIN FactWarehouseOperations w_out 
    ON w_in.ProductKey = w_out.ProductKey 
    AND w_in.WarehouseKey = w_out.WarehouseKey
    AND w_out.OperationType = 'SHIP'
INNER JOIN DimTime t ON w_in.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w_in.WarehouseKey = g.GeographyKey
WHERE w_in.OperationType = 'RECEIVE'
    AND w_out.RecordedAt > w_in.RecordedAt
    AND DATEDIFF(HOUR, w_in.RecordedAt, w_out.RecordedAt) <= 48
GROUP BY t.Year, t.Month, g.LocationName;
GO
```

## Alerting & Automation

### Automated Threshold Alerts

```sql
-- Stored procedure for automated KPI alerts
CREATE OR ALTER PROCEDURE AlertKPIBreaches
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check for excessive dwell time
    DECLARE @ExcessiveDwell TABLE (
        WarehouseName VARCHAR(200),
        ProductName VARCHAR(200),
        AvgDwellHours DECIMAL(10,2)
    );
    
    INSERT INTO @ExcessiveDwell
    SELECT TOP 10
        g.LocationName,
        p.ProductName,
        AVG(w.DwellTimeMinutes) / 60.0 AS AvgDwellHours
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.RecordedAt >= DATEADD(HOUR, -24, GETUTCDATE())
    GROUP BY g.LocationName, p.ProductName
    HAVING AVG(w.DwellTimeMinutes) > 72
    ORDER BY AvgDwellHours DESC;
    
    -- Check for fleet idle time
    DECLARE @ExcessiveIdle TABLE (
        VehicleID VARCHAR(50),
        AvgIdlePercent DECIMAL(5,2)
    );
    
    INSERT INTO @ExcessiveIdle
    SELECT TOP 10
        v.VehicleID,
        AVG(CAST(f.IdleTimeMinutes AS FLOAT) / 
            NULLIF(f.IdleTimeMinutes + DATEDIFF(MINUTE, 0, f.RecordedAt), 0)) * 100
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    WHERE f.RecordedAt >= DATEADD(HOUR, -24, GETUTCDATE())
    GROUP BY v.VehicleID
    HAVING AVG(CAST(f.IdleTimeMinutes AS FLOAT) / 
            NULLIF(f.IdleTimeMinutes + DATEDIFF(MINUTE, 0, f.RecordedAt), 0)) > 0.15
    ORDER BY AvgIdlePercent DESC;
    
    -- Generate alert report
    IF EXISTS (SELECT 1 FROM @ExcessiveDwell) OR EXISTS (SELECT 1 FROM @ExcessiveIdle)
    BEGIN
        -- In production, integrate with email/Teams API
        SELECT 'ALERT: Excessive Dwell Time' AS AlertType, * FROM @ExcessiveDwell
        UNION ALL
        SELECT 'ALERT: Excessive Fleet Idle' AS AlertType, VehicleID, AvgIdlePercent, NULL 
        FROM @ExcessiveIdle;
    END
END
GO

-- Schedule via SQL Agent job
-- EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_Hourly_Alerts'
-- EXEC msdb.dbo.sp_add_jobstep @step_name = 'Run Alert Check', @command = 'EXEC AlertKPIBreaches'
-- EXEC msdb.dbo.sp_add_schedule @schedule_name = 'Hourly', @freq_type = 4, @freq_interval = 1
```

## Environment Configuration

### Connection Configuration Template

```json
{
  "data_sources": {
    "sql_server": {
      "host": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "authentication": {
        "type": "windows",
        "username": "${SQL_USER}",
        "password": "${SQL_PASSWORD}"
      }
    },
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}",
      "refresh_interval_minutes": 15
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}",
      "refresh_interval_minutes": 5
    }
  },
  "power_bi": {
    "workspace_id": "${POWERBI_WORKSPACE_ID}",
    "dataset_refresh_schedule": "0 */15 * * *"
  },
  "alerting": {
    "email_smtp_server": "${SMTP_SERVER}",
    "notification_recipients": [
      "${ALERT_EMAIL_1}",
      "${ALERT_EMAIL_2}"
    ]
  }
}
```

### Environment Variables Reference

```bash
# Required environment variables
export SQL_SERVER_HOST="your-sql-server.database.windows.net"
export SQL_USER="logifleet_admin"
export SQL_PASSWORD="your-secure-password"
export WMS_API_ENDPOINT="https://your-wms-system.com/api/v1"
export WMS_API_KEY="your-wms-api-key"
export TELEMATICS_API_ENDPOINT="https://fleet-tracking.com/api"
export TELEMATICS_API_KEY="your-telematics-key"
export POWERBI_WORKSPACE_ID="your-workspace-guid"
export SMTP_SERVER="smtp.company.com"
export ALERT_EMAIL_1="logistics-manager@company.com"
export ALERT_EMAIL_2="supply-chain-director@company.com"
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom:** Queries joining warehouse and fleet facts take >30 seconds

**Solution:**
```sql
-- Create covering indexes on foreign keys
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, QuantityHandled, OperationCost);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_TimeVehicle 
ON FactFleetTrips(TimeKey, VehicleKey) 
INCLUDE (IdleTimeMinutes, FuelConsumedGallons, TripCost);

-- Enable query store for analysis
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;

-- Check for missing statistics
EXEC sp_updatestats;
```

### Issue: Power BI Refresh Timeout

**Symptom:** Dataset refresh fails with timeout error

**Solution:**
```powerquery
// Split large fact tables into yearly partitions
let
    Source = Sql.Database(
        #"SQL_SERVER_HOST", 
        "LogiFleetPulse",
        [Query="
            SELECT * FROM FactWarehouseOperations
            WHERE YEAR(RecordedAt) = YEAR(GETDATE())
        "]
    ),
    
    // Apply incremental refresh policy in Power BI Service
    #"Filtered Rows" = Table.SelectRows(
        Source, 
        each [RecordedAt] >= RangeStart and [RecordedAt] < RangeEnd
    )
in
    #"Filtered Rows"
```

### Issue: Incorrect Gravity Zone Assignments

**Symptom:** High-value products showing low gravity scores

**Solution:**
```sql
-- Recalculate gravity scores based on actual performance
UPDATE DimProductGravity
SET 
    VelocityScore = ISNULL(v.ActualVelocity, VelocityScore),
    GravityScore = ISNULL(v.ActualVelocity * UnitValue / NULLIF(FragilityScore, 0), GravityScore),
    RecommendedZone = CASE 
        WHEN v.ActualVelocity * UnitValue / NULLIF(FragilityScore, 0) >= 70 THEN 'HIGH_GRAVITY'
        WHEN v.ActualVelocity * UnitValue / NULLIF(FragilityScore, 0) BETWEEN 40 AND 69 THEN 'MEDIUM_GRAVITY'
        ELSE 'LOW_GRAVITY'
    END
FROM DimProductGravity p
LEFT JOIN (
    SELECT 
        ProductKey,
        COUNT(*) / 30.0 AS ActualVelocity -- picks per day over 30 days
    FROM FactWarehouseOperations
    WHERE OperationType = 'PICK'
        AND RecordedAt >= DATEADD(DAY, -30, GETUTCDATE())
    GROUP BY ProductKey
) v ON p.ProductKey = v.ProductKey;
```

### Issue: Row-Level Security Not Filtering

**Symptom:** Users see data outside their authorized region

**Solution:**
```sql
-- Verify RLS role membership
SELECT 
    dp.name AS RoleName,
    u.name AS UserName,
    u.type_desc AS UserType
FROM sys.database_role_members drm
