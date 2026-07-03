---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain analytics with multi-fact star schema modeling
triggers:
  - "set up LogiFleet Pulse analytics"
  - "configure supply chain data warehouse"
  - "create logistics Power BI dashboard"
  - "implement multi-fact star schema for logistics"
  - "build fleet analytics warehouse"
  - "design warehouse operations reporting"
  - "set up logistics KPI tracking"
  - "configure cross-modal supply chain analytics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics solution for logistics and supply chain management. It provides:

- **Multi-fact star schema** combining warehouse operations, fleet telemetry, cross-dock activities, and supplier data
- **Time-phased dimensional modeling** with 15-minute granularity
- **Power BI dashboards** for real-time operational visibility
- **Cross-fact KPI harmonization** linking inventory turnover with fleet performance
- **Predictive analytics** for bottleneck detection and maintenance prioritization
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and value

Built on MS SQL Server with Power BI visualization layer, designed for 3PLs, retail chains, food distributors, and any organization managing complex logistics operations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Access to data sources: WMS, telematics/GPS, ERP, procurement systems

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema creation script (order matters)

-- 1. Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- 2. Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfWeek INT NOT NULL,
    HourOfDay INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    IsBusinessHour BIT NOT NULL,
    FiscalPeriod VARCHAR(20)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Supplier', 'Customer'
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    Timezone VARCHAR(50)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitValue DECIMAL(12,2),
    Fragility VARCHAR(20), -- 'High', 'Medium', 'Low'
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    GravityScore DECIMAL(5,2), -- 0-100, higher = closer to shipping dock
    RequiresColdStorage BIT,
    RequiresHazmatHandling BIT,
    WeightKg DECIMAL(10,2),
    VolumeCubicMeters DECIMAL(10,4)
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(100) UNIQUE NOT NULL,
    SupplierName VARCHAR(300),
    LeadTimeDaysAvg DECIMAL(6,2),
    LeadTimeDaysStdDev DECIMAL(6,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastAuditDate DATE,
    TierLevel VARCHAR(20) -- 'Platinum', 'Gold', 'Silver', 'Bronze'
);

CREATE TABLE DimFleet (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(100) UNIQUE NOT NULL,
    VehicleType VARCHAR(50), -- 'Truck', 'Van', 'Refrigerated', etc.
    Capacity DECIMAL(10,2),
    CapacityUnit VARCHAR(20), -- 'kg', 'cubic meters'
    FuelType VARCHAR(50),
    YearManufactured INT,
    MaintenanceStatus VARCHAR(50),
    LastServiceDate DATE,
    NextServiceDue DATE
);

-- 3. Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    ZoneID VARCHAR(50),
    Quantity INT,
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    OperatorID VARCHAR(50),
    BatchID VARCHAR(100),
    ErrorsLogged INT DEFAULT 0
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimFleet(VehicleKey),
    OriginKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DriverID VARCHAR(50),
    RouteID VARCHAR(100),
    DistanceKm DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    AverageSpeedKmh DECIMAL(6,2),
    WeatherCondition VARCHAR(50),
    OnTimeDelivery BIT
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundVehicleKey INT FOREIGN KEY REFERENCES DimFleet(VehicleKey),
    OutboundVehicleKey INT FOREIGN KEY REFERENCES DimFleet(VehicleKey),
    Quantity INT,
    DwellTimeMinutes INT, -- Time in cross-dock area
    TransferType VARCHAR(50) -- 'Direct', 'Consolidated', 'Breakbulk'
);

-- 4. Create indexes for performance
CREATE INDEX IX_FactWarehouseOps_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouseOps_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactFleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleetTrips_Vehicle ON FactFleetTrips(VehicleKey);
CREATE INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey);

-- 5. Create stored procedure for incremental loading
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartTime DATETIME,
    @EndTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS table
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        ZoneID, Quantity, DwellTimeMinutes, CycleTimeMinutes,
        OperatorID, BatchID, ErrorsLogged
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        wms.operation_type,
        wms.zone_id,
        wms.quantity,
        DATEDIFF(MINUTE, wms.start_time, wms.end_time) AS DwellTimeMinutes,
        wms.cycle_time_minutes,
        wms.operator_id,
        wms.batch_id,
        wms.error_count
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimTime dt ON DATEADD(MINUTE, (DATEPART(MINUTE, wms.timestamp) / 15) * 15, 
                                      DATEADD(HOUR, DATEDIFF(HOUR, 0, wms.timestamp), 0)) = dt.DateTime
    INNER JOIN DimGeography dg ON wms.warehouse_id = dg.LocationID
    INNER JOIN DimProductGravity dp ON wms.sku = dp.SKU
    WHERE wms.timestamp BETWEEN @StartTime AND @EndTime;
END;
GO
```

### Step 2: Populate Dimension Tables

```sql
-- Example: Populate DimTime with 15-minute intervals
DECLARE @StartDate DATE = '2024-01-01';
DECLARE @EndDate DATE = '2026-12-31';
DECLARE @CurrentDateTime DATETIME = @StartDate;

WHILE @CurrentDateTime <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, DateTime, Date, Year, Quarter, Month, Week, 
                         DayOfWeek, HourOfDay, MinuteBucket, IsBusinessHour, FiscalPeriod)
    VALUES (
        CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
        @CurrentDateTime,
        CAST(@CurrentDateTime AS DATE),
        YEAR(@CurrentDateTime),
        DATEPART(QUARTER, @CurrentDateTime),
        MONTH(@CurrentDateTime),
        DATEPART(WEEK, @CurrentDateTime),
        DATEPART(WEEKDAY, @CurrentDateTime),
        DATEPART(HOUR, @CurrentDateTime),
        DATEPART(MINUTE, @CurrentDateTime),
        CASE WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
             AND DATEPART(WEEKDAY, @CurrentDateTime) BETWEEN 2 AND 6 
             THEN 1 ELSE 0 END,
        'FY' + CAST(CASE WHEN MONTH(@CurrentDateTime) >= 7 
                         THEN YEAR(@CurrentDateTime) + 1 
                         ELSE YEAR(@CurrentDateTime) END AS VARCHAR(4))
    );
    
    SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
END;

-- Example: Insert sample geography data
INSERT INTO DimGeography (LocationID, LocationName, LocationType, City, Region, Country, Continent)
VALUES 
    ('WH-001', 'Chicago Distribution Center', 'Warehouse', 'Chicago', 'Illinois', 'USA', 'North America'),
    ('WH-002', 'Los Angeles Hub', 'Warehouse', 'Los Angeles', 'California', 'USA', 'North America'),
    ('RN-101', 'Route Junction A', 'Route Node', 'Denver', 'Colorado', 'USA', 'North America');

-- Example: Calculate and update gravity scores
UPDATE DimProductGravity
SET GravityScore = 
    (CASE VelocityClass 
        WHEN 'Fast' THEN 50.0 
        WHEN 'Medium' THEN 30.0 
        WHEN 'Slow' THEN 10.0 
        ELSE 5.0 
    END) +
    (UnitValue / 1000.0 * 20) + -- Value contribution
    (CASE Fragility 
        WHEN 'High' THEN 15.0 
        WHEN 'Medium' THEN 5.0 
        ELSE 0.0 
    END);
```

### Step 3: Configure Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Connect to your server with credentials from environment variables:

```powerquery
// In Power BI Power Query (M language)
let
    Server = Environment.GetEnvironmentVariable("LOGIFLEET_SQL_SERVER"),
    Database = "LogiFleetPulse",
    Source = Sql.Database(Server, Database)
in
    Source
```

### Step 4: Create Power BI Data Model

```powerquery
// Load fact tables
FactWarehouseOps = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
FactFleet = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
FactCrossDock = Source{[Schema="dbo",Item="FactCrossDock"]}[Data],

// Load dimensions
DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data],
DimProduct = Source{[Schema="dbo",Item="DimProductGravity"]}[Data],
DimSupplier = Source{[Schema="dbo",Item="DimSupplierReliability"]}[Data],
DimFleet = Source{[Schema="dbo",Item="DimFleet"]}[Data]

// Create relationships in Power BI model:
// FactWarehouseOps[TimeKey] → DimTime[TimeKey] (many-to-one)
// FactWarehouseOps[ProductKey] → DimProduct[ProductKey] (many-to-one)
// FactFleetTrips[VehicleKey] → DimFleet[VehicleKey] (many-to-one)
// etc.
```

## Key DAX Measures for Analytics

```dax
// Total Dwell Time (hours)
Total Dwell Time = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

// Average Fleet Utilization (%)
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Cross-Fact KPI: Fuel Cost per Product Unit Shipped
Fuel Cost per Unit = 
VAR TotalFuel = SUM(FactFleetTrips[FuelConsumedLiters])
VAR FuelPrice = 1.5 -- From parameter table or external source
VAR TotalUnits = SUM(FactWarehouseOperations[Quantity])
RETURN
DIVIDE(TotalFuel * FuelPrice, TotalUnits, 0)

// Warehouse Gravity Score Efficiency
Gravity Efficiency = 
AVERAGEX(
    FactWarehouseOperations,
    RELATED(DimProductGravity[GravityScore]) / 
    (FactWarehouseOperations[DwellTimeMinutes] + 1)
)

// Predictive Bottleneck Index (simplified)
Bottleneck Risk = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR CurrentDwell = CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]), 
                             DimTime[Date] = TODAY())
VAR CurrentIdle = CALCULATE(AVERAGE(FactFleetTrips[IdleTimeMinutes]), 
                           DimTime[Date] = TODAY())
RETURN
((CurrentDwell / AvgDwell) + (CurrentIdle / AvgIdle)) / 2

// Temporal Elasticity: What-If Capacity Scenario
Capacity Impact = 
VAR BaseCapacity = 80
VAR WhatIfCapacity = [Capacity Scenario Parameter]
VAR CapacityRatio = WhatIfCapacity / BaseCapacity
RETURN
[Fleet Utilization %] * CapacityRatio
```

## Common Analytical Patterns

### Pattern 1: Cross-Modal Correlation Analysis

```sql
-- SQL query to identify correlation between dwell time and fleet delays
WITH WarehouseDwell AS (
    SELECT 
        wo.TimeKey,
        wo.WarehouseKey,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    WHERE wo.OperationType = 'Shipping'
    GROUP BY wo.TimeKey, wo.WarehouseKey
),
FleetDelays AS (
    SELECT 
        ft.TimeKey,
        ft.OriginKey,
        AVG(CASE WHEN ft.OnTimeDelivery = 0 THEN 1 ELSE 0 END) AS DelayRate,
        COUNT(*) AS TripCount
    FROM FactFleetTrips ft
    GROUP BY ft.TimeKey, ft.OriginKey
)
SELECT 
    dt.Date,
    dg.LocationName,
    wd.AvgDwell,
    fd.DelayRate,
    -- Correlation coefficient can be calculated in Power BI or Python
    wd.OperationCount,
    fd.TripCount
FROM WarehouseDwell wd
INNER JOIN FleetDelays fd ON wd.TimeKey = fd.TimeKey AND wd.WarehouseKey = fd.OriginKey
INNER JOIN DimTime dt ON wd.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON wd.WarehouseKey = dg.GeographyKey
WHERE dt.Date >= DATEADD(DAY, -30, GETDATE())
ORDER BY dt.Date DESC;
```

### Pattern 2: Gravity Zone Optimization

```sql
-- Identify products that should be moved to different gravity zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore AS CurrentGravity,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS PickFrequency,
    SUM(wo.Quantity * dp.UnitValue) AS TotalValue,
    -- Recommended new gravity score
    CASE 
        WHEN COUNT(*) > 100 AND AVG(wo.DwellTimeMinutes) > 30 THEN dp.GravityScore + 20
        WHEN COUNT(*) < 10 AND dp.GravityScore > 50 THEN dp.GravityScore - 15
        ELSE dp.GravityScore
    END AS RecommendedGravity
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
WHERE wo.OperationType IN ('Picking', 'Packing')
    AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore
HAVING AVG(wo.DwellTimeMinutes) <> 0
ORDER BY TotalValue DESC;
```

### Pattern 3: Fleet Maintenance Prioritization

```sql
-- Adaptive fleet triage scoring
WITH TripMetrics AS (
    SELECT 
        ft.VehicleKey,
        AVG(ft.IdleTimeMinutes * 1.0 / NULLIF(ft.DurationMinutes, 0)) AS IdleRatio,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS FuelEfficiency,
        COUNT(*) AS TripCount,
        SUM(CASE WHEN ft.OnTimeDelivery = 0 THEN 1 ELSE 0 END) AS DelayCount
    FROM FactFleetTrips ft
    WHERE ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY ft.VehicleKey
),
RevenueImpact AS (
    SELECT 
        ft.VehicleKey,
        SUM(wo.Quantity * dp.UnitValue) AS WeeklyRevenue
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo 
        ON ft.TimeKey = wo.TimeKey 
        AND ft.OriginKey = wo.WarehouseKey
    INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    WHERE ft.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
    GROUP BY ft.VehicleKey
)
SELECT 
    df.VehicleID,
    df.VehicleType,
    df.MaintenanceStatus,
    DATEDIFF(DAY, df.LastServiceDate, GETDATE()) AS DaysSinceService,
    tm.IdleRatio,
    tm.FuelEfficiency,
    tm.DelayCount,
    ri.WeeklyRevenue,
    -- Triage Score (higher = more urgent)
    (
        (tm.IdleRatio * 20) +
        (1.0 / NULLIF(tm.FuelEfficiency, 0) * 15) +
        (tm.DelayCount * 10) +
        (ri.WeeklyRevenue / 10000 * 25) +
        (DATEDIFF(DAY, df.LastServiceDate, GETDATE()) / 30.0 * 30)
    ) AS TriageScore
FROM DimFleet df
INNER JOIN TripMetrics tm ON df.VehicleKey = tm.VehicleKey
LEFT JOIN RevenueImpact ri ON df.VehicleKey = ri.VehicleKey
WHERE df.MaintenanceStatus <> 'In Service'
ORDER BY TriageScore DESC;
```

## Configuration

### Environment Variables

Set these before connecting Power BI or running ETL scripts:

```bash
# SQL Server connection
export LOGIFLEET_SQL_SERVER="your-server.database.windows.net"
export LOGIFLEET_SQL_DATABASE="LogiFleetPulse"
export LOGIFLEET_SQL_USERNAME="logifleet_user"
export LOGIFLEET_SQL_PASSWORD="your-secure-password"

# External data sources
export WMS_API_ENDPOINT="https://your-wms-api.com/v1"
export WMS_API_KEY="your-wms-api-key"
export TELEMATICS_API_ENDPOINT="https://fleet-tracking.com/api"
export TELEMATICS_API_KEY="your-telematics-key"

# Power BI service (for publishing)
export POWERBI_WORKSPACE_ID="your-workspace-guid"
```

### Scheduled Refresh Configuration

```sql
-- Create SQL Server Agent job for incremental ETL
USE msdb;
GO

EXEC sp_add_job 
    @job_name = N'LogiFleet_IncrementalLoad',
    @enabled = 1,
    @description = N'Load warehouse operations every 15 minutes';

EXEC sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load Warehouse Ops',
    @subsystem = N'TSQL',
    @command = N'
        DECLARE @StartTime DATETIME = DATEADD(MINUTE, -30, GETDATE());
        DECLARE @EndTime DATETIME = GETDATE();
        EXEC sp_LoadWarehouseOperations @StartTime, @EndTime;
    ',
    @database_name = N'LogiFleetPulse';

EXEC sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalLoad',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalLoad';
```

### Row-Level Security

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    UserEmail VARCHAR(255) PRIMARY KEY,
    RoleLevel VARCHAR(50), -- 'Executive', 'Manager', 'Operator'
    AllowedWarehouses VARCHAR(MAX), -- Comma-separated LocationIDs
    AllowedRegions VARCHAR(MAX)
);

-- Create RLS predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS result
    WHERE 
        EXISTS (
            SELECT 1 FROM dbo.SecurityRoles sr
            INNER JOIN dbo.DimGeography dg ON dg.GeographyKey = @WarehouseKey
            WHERE sr.UserEmail = USER_NAME()
                AND (
                    sr.RoleLevel = 'Executive'
                    OR ',' + sr.AllowedWarehouses + ',' LIKE '%,' + dg.LocationID + ',%'
                    OR ',' + sr.AllowedRegions + ',' LIKE '%,' + dg.Region + ',%'
                )
        );
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

## Troubleshooting

### Issue: Slow Query Performance on Large Fact Tables

**Solution**: Partition fact tables by time dimension

```sql
-- Create partition function (monthly partitions)
CREATE PARTITION FUNCTION pf_TimeKey (INT)
AS RANGE RIGHT FOR VALUES (
    202401010000, 202402010000, 202403010000, 202404010000,
    202405010000, 202406010000, 202407010000, 202408010000,
    202409010000, 202410010000, 202411010000, 202412010000
);

-- Create partition scheme
CREATE PARTITION SCHEME ps_TimeKey
AS PARTITION pf_TimeKey
ALL TO ([PRIMARY]);

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- ... other columns
    CONSTRAINT PK_FactWarehouseOps_Part PRIMARY KEY (TimeKey, OperationKey)
) ON ps_TimeKey(TimeKey);
```

### Issue: Power BI Report Refresh Timeout

**Solution**: Use incremental refresh policy

1. In Power BI Desktop, add RangeStart and RangeEnd parameters
2. Filter fact table:

```powerquery
= Table.SelectRows(
    FactWarehouseOps, 
    each [DateTime] >= RangeStart and [DateTime] < RangeEnd
)
```

3. Configure incremental refresh in dataset settings:
   - Archive data starting 2 years before refresh date
   - Incrementally refresh data starting 7 days before refresh date
   - Detect data changes based on TimeKey column

### Issue: Gravity Score Not Reflecting Current Patterns

**Solution**: Implement auto-recalculation stored procedure

```sql
CREATE PROCEDURE sp_RecalculateGravityScores
AS
BEGIN
    -- Calculate velocity from last 90 days
    WITH ProductVelocity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            AVG(DwellTimeMinutes) AS AvgDwell
        FROM FactWarehouseOperations
        WHERE OperationType IN ('Picking', 'Packing')
            AND TimeKey >= CAST(FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY ProductKey
    )
    UPDATE dp
    SET 
        VelocityClass = CASE 
            WHEN pv.PickCount > 200 THEN 'Fast'
            WHEN pv.PickCount > 50 THEN 'Medium'
            ELSE 'Slow'
        END,
        GravityScore = 
            (CASE 
                WHEN pv.PickCount > 200 THEN 50.0
                WHEN pv.PickCount > 50 THEN 30.0
                ELSE 10.0
            END) +
            (dp.UnitValue / 1000.0 * 20) +
            (CASE dp.Fragility 
                WHEN 'High' THEN 15.0
                WHEN 'Medium' THEN 5.0
                ELSE 0.0
            END) -
            (pv.AvgDwell / 10.0) -- Penalty for high dwell time
    FROM DimProductGravity dp
    INNER JOIN ProductVelocity pv ON dp.ProductKey = pv.ProductKey;
END;
```

Schedule this to run weekly.

### Issue: Missing Data from External APIs

**Solution**: Implement error logging and retry logic

```sql
CREATE TABLE ETL_ErrorLog (
    ErrorID INT IDENTITY(1,1) PRIMARY KEY,
    ErrorDateTime DATETIME DEFAULT GETDATE(),
    SourceSystem VARCHAR(100),
    ErrorMessage VARCHAR(MAX),
    RecordDetails VARCHAR(MAX),
    Resolved BIT DEFAULT 0
);

-- Example: Safe external data load with error handling
CREATE PROCEDURE sp_LoadTelematicsData
    @RetryCount INT = 3
AS
BEGIN
    DECLARE @Attempt INT = 0;
    DECLARE @Success BIT = 0;
    
    WHILE @Attempt < @RetryCount AND @Success = 0
    BEGIN TRY
        -- Attempt to load from external table or API
        INSERT INTO FactFleetTrips (TimeKey, VehicleKey, OriginKey, DestinationKey, ...)
        SELECT ... FROM OPENROWSET(...);
        
        SET @Success = 1;
    END TRY
    BEGIN CATCH
        SET @Attempt = @Attempt + 1;
        
        INSERT INTO ETL_ErrorLog (SourceSystem, ErrorMessage, RecordDetails)
        VALUES (
            'Telematics API',
            ERROR_MESSAGE(),
            'Attempt ' + CAST(@Attempt AS VARCHAR(10))
        );
        
        IF @Attempt < @RetryCount
            WAITFOR DELAY '00:00:30'; -- Wait 30 seconds before retry
    END CATCH
END;
```

## Advanced Use Cases

### Predictive Demand Forecasting Integration

```dax
// DAX measure using built-in forecasting
Demand
