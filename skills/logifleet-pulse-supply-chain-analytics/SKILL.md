---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing engine for fleet logistics and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse schema"
  - "configure Power BI fleet dashboard"
  - "create warehouse operations fact table"
  - "implement cross-modal supply chain KPIs"
  - "build logistics intelligence dashboard"
  - "query fleet telemetry with SQL"
  - "optimize warehouse gravity zones"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that unifies warehouse operations, fleet telemetry, inventory management, and external data signals into a cohesive MS SQL Server data warehouse with Power BI visualization layer. It provides:

- **Multi-fact star schema** for warehouse operations, fleet trips, and cross-dock transfers
- **Time-phased dimensions** with 15-minute granularity for real-time analytics
- **Cross-fact KPI harmonization** linking inventory turnover with fleet performance
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and item value
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Power BI dashboards** with role-based access and natural language queries

The platform integrates data from WMS, telematics/GPS feeds, supplier portals, weather APIs, and customer order history into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, ERP) via SQL, REST API, or flat files

### Step 1: Deploy SQL Schema

Clone or download the repository and locate the SQL deployment script:

```sql
-- Execute on your SQL Server instance
-- File: schema/01_create_database.sql

CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    FifteenMinuteBucket SMALLINT,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    Month TINYINT,
    Quarter TINYINT,
    FiscalYear SMALLINT,
    FiscalPeriod TINYINT,
    IsWeekend BIT,
    IsHoliday BIT
)
GO

CREATE CLUSTERED INDEX IX_DimTime_DateTime ON DimTime(DateTime)
GO

CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    Continent NVARCHAR(50),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    City NVARCHAR(100),
    PostalCode NVARCHAR(20),
    WarehouseID NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50)
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitValue DECIMAL(18,2),
    WeightKG DECIMAL(10,3),
    VolumeM3 DECIMAL(10,4),
    Fragility TINYINT, -- 1-5 scale
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value * fragility
    IsPerishable BIT,
    ShelfLifeDays INT,
    OptimalStorageTemp DECIMAL(5,2)
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    Country NVARCHAR(100),
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,3),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100
    LastAuditDate DATE
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    Quantity INT,
    DwellTimeMinutes INT,
    StorageZone NVARCHAR(50),
    StorageAisle NVARCHAR(20),
    HandlingEquipmentID NVARCHAR(50),
    OperatorID NVARCHAR(50),
    CycleTimeSeconds INT,
    ErrorsCount TINYINT,
    BatchNumber NVARCHAR(50),
    OrderID NVARCHAR(50)
)
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
INCLUDE (OperationType, Quantity, DwellTimeMinutes)
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey, TimeKey)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    AvgSpeedKPH DECIMAL(5,1),
    LoadWeightKG DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,2),
    StopsCount TINYINT,
    WeatherCondition NVARCHAR(50),
    TrafficDelay NVARCHAR(50),
    OnTimeDelivery BIT,
    DelayMinutes INT,
    TollCost DECIMAL(10,2),
    MaintenanceFlag BIT
)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey)
INCLUDE (VehicleID, DistanceKM, FuelConsumedLiters)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID, StartTimeKey)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OutboundTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT,
    DwellTimeMinutes INT,
    TransferType NVARCHAR(50), -- Direct, Staged, Consolidated
    SortingDurationSeconds INT
)
GO
```

### Step 2: Load Time Dimension

```sql
-- File: schema/02_populate_time_dimension.sql
-- Generate time dimension with 15-minute buckets for current year + 2 years

DECLARE @StartDate DATETIME2 = DATEFROMPARTS(YEAR(GETDATE()), 1, 1)
DECLARE @EndDate DATETIME2 = DATEADD(YEAR, 3, @StartDate)
DECLARE @CurrentDate DATETIME2 = @StartDate

WHILE @CurrentDate < @EndDate
BEGIN
    INSERT INTO DimTime (
        TimeKey,
        DateTime,
        FifteenMinuteBucket,
        HourOfDay,
        DayOfWeek,
        DayOfMonth,
        Month,
        Quarter,
        FiscalYear,
        FiscalPeriod,
        IsWeekend,
        IsHoliday
    )
    VALUES (
        CAST(FORMAT(@CurrentDate, 'yyyyMMddHHmm') AS INT),
        @CurrentDate,
        DATEPART(MINUTE, @CurrentDate) / 15,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(DAY, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        YEAR(@CurrentDate),
        ((MONTH(@CurrentDate) - 1) / 3) + 1,
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        0 -- Update with actual holiday logic
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
GO
```

### Step 3: Create ETL Stored Procedures

```sql
-- File: schema/03_etl_procedures.sql

CREATE PROCEDURE usp_LoadWarehouseOperations
    @SourceConnectionString NVARCHAR(500),
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS via linked server or OPENROWSET
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, StorageZone,
        CycleTimeSeconds, OrderID
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        ds.SupplierKey,
        src.operation_type,
        src.quantity,
        DATEDIFF(MINUTE, src.start_time, src.end_time) AS DwellTimeMinutes,
        src.storage_zone,
        DATEDIFF(SECOND, src.start_time, src.end_time) AS CycleTimeSeconds,
        src.order_id
    FROM 
        -- Replace with your actual source: linked server, OPENROWSET, or staging table
        OPENROWSET('SQLNCLI', @SourceConnectionString,
            'SELECT * FROM wms.operations WHERE timestamp > ?', @LastLoadTimestamp) AS src
    INNER JOIN DimTime dt ON CAST(FORMAT(src.start_time, 'yyyyMMddHHmm') AS INT) = dt.TimeKey
    INNER JOIN DimGeography dg ON src.warehouse_id = dg.WarehouseID
    INNER JOIN DimProductGravity dp ON src.sku = dp.SKU
    LEFT JOIN DimSupplierReliability ds ON src.supplier_id = ds.SupplierID
    WHERE src.timestamp > @LastLoadTimestamp
    
    -- Update last load timestamp in control table
    UPDATE ETLControl SET LastLoadTimestamp = GETDATE() WHERE TableName = 'FactWarehouseOperations'
END
GO

CREATE PROCEDURE usp_CalculateProductGravityScore
AS
BEGIN
    -- Recalculate gravity scores based on recent velocity
    UPDATE dp
    SET GravityScore = (
        (velocity.MovementFrequency * 0.4) +
        ((dp.UnitValue / NULLIF(maxval.MaxValue, 0)) * 100 * 0.3) +
        (dp.Fragility * 20 * 0.3)
    )
    FROM DimProductGravity dp
    CROSS APPLY (
        SELECT TOP 1 MAX(UnitValue) AS MaxValue FROM DimProductGravity
    ) maxval
    OUTER APPLY (
        SELECT 
            COUNT(DISTINCT TimeKey) AS MovementFrequency
        FROM FactWarehouseOperations
        WHERE ProductKey = dp.ProductKey
            AND TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY DateTime LIMIT 1)
    ) velocity
END
GO
```

### Step 4: Configure Power BI Connection

1. Open Power BI Desktop
2. Load the template file: `LogiFleet_Pulse_Master.pbit`
3. When prompted, enter connection parameters:

```
Server: ${SQL_SERVER_HOST}
Database: LogiFleetPulse
Authentication: SQL Server / Windows / Azure AD
Username: ${SQL_USERNAME}
Password: ${SQL_PASSWORD}
```

4. The model will auto-detect fact and dimension tables and build relationships

## Key SQL Queries & Patterns

### Query: Cross-Fact KPI - Dwell Time vs Fleet Idling

```sql
-- Find SKUs with high warehouse dwell time that also correlate with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        wo.ProductKey,
        dp.SKU,
        dp.ProductName,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY DateTime OFFSET 0 ROWS FETCH NEXT 1 ROW ONLY)
    GROUP BY wo.ProductKey, dp.SKU, dp.ProductName
    HAVING AVG(wo.DwellTimeMinutes) > 72 * 60 -- More than 72 hours
),
FleetIdle AS (
    SELECT 
        ft.VehicleID,
        AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(ft.DurationMinutes, 0)) * 100 AS IdlePercentage
    FROM FactFleetTrips ft
    WHERE ft.StartTimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY DateTime OFFSET 0 ROWS FETCH NEXT 1 ROW ONLY)
    GROUP BY ft.VehicleID
    HAVING AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(ft.DurationMinutes, 0)) > 0.15 -- >15% idle
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.AvgDwellTimeMinutes / 60.0 AS AvgDwellTimeHours,
    wd.OperationCount,
    fi.VehicleID,
    fi.IdlePercentage
FROM WarehouseDwell wd
CROSS JOIN FleetIdle fi
ORDER BY wd.AvgDwellTimeMinutes DESC, fi.IdlePercentage DESC
```

### Query: Warehouse Gravity Zone Optimization

```sql
-- Recommend product relocations based on gravity score vs current zone
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    wo.StorageZone,
    COUNT(*) AS PickCount,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTimeSeconds,
    CASE 
        WHEN dp.GravityScore > 80 AND wo.StorageZone NOT LIKE 'A-%' THEN 'MOVE TO ZONE A (High Gravity)'
        WHEN dp.GravityScore BETWEEN 50 AND 80 AND wo.StorageZone NOT LIKE 'B-%' THEN 'MOVE TO ZONE B (Medium Gravity)'
        WHEN dp.GravityScore < 50 AND wo.StorageZone LIKE 'A-%' THEN 'MOVE TO ZONE C/D (Low Gravity)'
        ELSE 'Optimally Placed'
    END AS Recommendation
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
WHERE wo.OperationType = 'Picking'
    AND wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(DAY, -90, GETDATE()) ORDER BY DateTime OFFSET 0 ROWS FETCH NEXT 1 ROW ONLY)
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, wo.StorageZone
HAVING COUNT(*) > 10
ORDER BY dp.GravityScore DESC
```

### Query: Predictive Bottleneck Detection

```sql
-- Identify time slots with elevated risk of congestion
WITH HourlyMetrics AS (
    SELECT 
        dt.HourOfDay,
        dt.DayOfWeek,
        COUNT(*) AS OperationVolume,
        AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
        STDEV(wo.CycleTimeSeconds) AS StdDevCycleTime,
        SUM(wo.ErrorsCount) AS TotalErrors
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE wo.TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(DAY, -30, GETDATE()) ORDER BY DateTime OFFSET 0 ROWS FETCH NEXT 1 ROW ONLY)
    GROUP BY dt.HourOfDay, dt.DayOfWeek
)
SELECT 
    HourOfDay,
    DayOfWeek,
    OperationVolume,
    AvgCycleTime,
    StdDevCycleTime,
    TotalErrors,
    -- Bottleneck Index: high volume + high variance + errors
    (OperationVolume / 100.0) * 0.4 +
    (StdDevCycleTime / NULLIF(AvgCycleTime, 0)) * 100 * 0.4 +
    (TotalErrors * 10) * 0.2 AS BottleneckIndex
FROM HourlyMetrics
ORDER BY BottleneckIndex DESC
```

### Query: Fleet Fuel Efficiency by Route Segment

```sql
-- Calculate fuel consumption per km for each route segment
SELECT 
    og.City AS OriginCity,
    dg.City AS DestinationCity,
    COUNT(*) AS TripCount,
    AVG(ft.DistanceKM) AS AvgDistanceKM,
    AVG(ft.FuelConsumedLiters) AS AvgFuelLiters,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelPerKM,
    AVG(ft.AvgSpeedKPH) AS AvgSpeed,
    AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(ft.DurationMinutes, 0)) * 100 AS AvgIdlePercent
FROM FactFleetTrips ft
INNER JOIN DimGeography og ON ft.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON ft.DestinationGeographyKey = dg.GeographyKey
WHERE ft.StartTimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(DAY, -60, GETDATE()) ORDER BY DateTime OFFSET 0 ROWS FETCH NEXT 1 ROW ONLY)
GROUP BY og.City, dg.City
HAVING COUNT(*) >= 5
ORDER BY AvgFuelPerKM DESC
```

## Configuration

### Environment Variables

Set these in your deployment environment or `.env` file:

```bash
# SQL Server Connection
SQL_SERVER_HOST=your-sql-server.database.windows.net
SQL_DATABASE=LogiFleetPulse
SQL_USERNAME=${DB_USERNAME}
SQL_PASSWORD=${DB_PASSWORD}

# External Data Sources
WMS_API_ENDPOINT=${WMS_API_URL}
WMS_API_KEY=${WMS_API_KEY}
TELEMATICS_API_ENDPOINT=${TELEMATICS_API_URL}
TELEMATICS_API_KEY=${TELEMATICS_API_KEY}
WEATHER_API_KEY=${WEATHER_API_KEY}

# Power BI Service (for scheduled refresh)
POWERBI_WORKSPACE_ID=${POWERBI_WORKSPACE_ID}
POWERBI_CLIENT_ID=${AZURE_CLIENT_ID}
POWERBI_CLIENT_SECRET=${AZURE_CLIENT_SECRET}
POWERBI_TENANT_ID=${AZURE_TENANT_ID}
```

### ETL Configuration Table

```sql
-- File: schema/04_etl_control.sql

CREATE TABLE ETLControl (
    TableName NVARCHAR(100) PRIMARY KEY,
    LastLoadTimestamp DATETIME2,
    LoadFrequencyMinutes INT,
    IsActive BIT,
    ErrorLog NVARCHAR(MAX)
)
GO

INSERT INTO ETLControl VALUES 
    ('FactWarehouseOperations', '2026-01-01', 15, 1, NULL),
    ('FactFleetTrips', '2026-01-01', 15, 1, NULL),
    ('FactCrossDock', '2026-01-01', 30, 1, NULL),
    ('DimProductGravity', '2026-01-01', 1440, 1, NULL) -- Daily
GO
```

### Automated Alerts Configuration

```sql
-- File: schema/05_alerting.sql

CREATE TABLE AlertRules (
    AlertRuleID INT IDENTITY(1,1) PRIMARY KEY,
    RuleName NVARCHAR(200),
    Metric NVARCHAR(100),
    Threshold DECIMAL(18,2),
    ComparisonOperator NVARCHAR(10), -- '>', '<', '=', '>=', '<='
    AlertRecipients NVARCHAR(MAX), -- Comma-separated emails
    IsActive BIT
)
GO

INSERT INTO AlertRules VALUES
    ('Fleet Idle Time Exceeded', 'IdleTimePercentage', 15.0, '>', 'fleet-managers@company.com', 1),
    ('Warehouse Dwell Time Breach', 'DwellTimeHours', 72.0, '>', 'warehouse-ops@company.com', 1),
    ('Fuel Efficiency Anomaly', 'FuelPerKM', 0.25, '>', 'logistics-vp@company.com', 1)
GO

CREATE PROCEDURE usp_CheckAlerts
AS
BEGIN
    -- Check each alert rule and send notifications
    -- This is a simplified example; integrate with your email/SMS system
    
    DECLARE @RuleID INT, @RuleName NVARCHAR(200), @Metric NVARCHAR(100), 
            @Threshold DECIMAL(18,2), @ComparisonOperator NVARCHAR(10), 
            @Recipients NVARCHAR(MAX)
    
    DECLARE alert_cursor CURSOR FOR
        SELECT AlertRuleID, RuleName, Metric, Threshold, ComparisonOperator, AlertRecipients
        FROM AlertRules WHERE IsActive = 1
    
    OPEN alert_cursor
    FETCH NEXT FROM alert_cursor INTO @RuleID, @RuleName, @Metric, @Threshold, @ComparisonOperator, @Recipients
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Example for Fleet Idle Time
        IF @Metric = 'IdleTimePercentage'
        BEGIN
            IF EXISTS (
                SELECT 1 
                FROM FactFleetTrips 
                WHERE StartTimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTime >= DATEADD(HOUR, -1, GETDATE()) ORDER BY DateTime OFFSET 0 ROWS FETCH NEXT 1 ROW ONLY)
                    AND (CAST(IdleTimeMinutes AS FLOAT) / NULLIF(DurationMinutes, 0)) * 100 > @Threshold
            )
            BEGIN
                -- Log alert and send notification (integrate with sp_send_dbmail or external service)
                PRINT 'ALERT: ' + @RuleName + ' triggered for recipients: ' + @Recipients
            END
        END
        
        FETCH NEXT FROM alert_cursor INTO @RuleID, @RuleName, @Metric, @Threshold, @ComparisonOperator, @Recipients
    END
    
    CLOSE alert_cursor
    DEALLOCATE alert_cursor
END
GO
```

## Power BI Dashboard Configuration

### Key Measures (DAX)

Create these measures in Power BI for cross-fact calculations:

```dax
-- Measure: Average Warehouse Dwell Time (Hours)
Avg Dwell Time (Hours) = 
AVERAGEX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60
)

-- Measure: Fleet Idle Percentage
Fleet Idle % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

-- Measure: Cross-Fact Efficiency Score
Efficiency Score = 
VAR WarehouseScore = 100 - ([Avg Dwell Time (Hours)] / 72 * 100)
VAR FleetScore = 100 - [Fleet Idle %]
RETURN (WarehouseScore * 0.5) + (FleetScore * 0.5)

-- Measure: Gravity Zone Compliance
Gravity Compliance % = 
VAR HighGravityInZoneA = 
    CALCULATE(
        DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
        DimProductGravity[GravityScore] > 80,
        FactWarehouseOperations[StorageZone] IN {"A-1", "A-2", "A-3"}
    )
VAR TotalHighGravity = 
    CALCULATE(
        DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
        DimProductGravity[GravityScore] > 80
    )
RETURN DIVIDE(HighGravityInZoneA, TotalHighGravity, 0) * 100

-- Measure: Fuel Cost (using external price table)
Total Fuel Cost = 
SUMX(
    FactFleetTrips,
    FactFleetTrips[FuelConsumedLiters] * RELATED(DimFuelPrice[PricePerLiter])
)
```

### Row-Level Security (RLS)

Define security roles in Power BI based on user geography:

```dax
-- Role: Region Managers (see only their region)
[DimGeography[Region]] = USERNAME()

-- Role: Warehouse Supervisors (see only their warehouse)
[DimGeography[WarehouseID]] = LOOKUPVALUE(
    UserAccess[WarehouseID],
    UserAccess[Email],
    USERPRINCIPALNAME()
)
```

## Common Patterns

### Pattern 1: Incremental ETL Load

```sql
-- Schedule this procedure to run every 15 minutes via SQL Agent
CREATE PROCEDURE usp_IncrementalETL
AS
BEGIN
    DECLARE @LastLoad DATETIME2
    
    -- Load Warehouse Operations
    SELECT @LastLoad = LastLoadTimestamp FROM ETLControl WHERE TableName = 'FactWarehouseOperations'
    EXEC usp_LoadWarehouseOperations '${WMS_CONNECTION_STRING}', @LastLoad
    
    -- Load Fleet Trips
    SELECT @LastLoad = LastLoadTimestamp FROM ETLControl WHERE TableName = 'FactFleetTrips'
    EXEC usp_LoadFleetTrips '${TELEMATICS_CONNECTION_STRING}', @LastLoad
    
    -- Recalculate Gravity Scores (daily)
    IF DATEPART(HOUR, GETDATE()) = 2 AND DATEPART(MINUTE, GETDATE()) < 15
    BEGIN
        EXEC usp_CalculateProductGravityScore
    END
    
    -- Check Alerts
    EXEC usp_CheckAlerts
END
GO
```

### Pattern 2: External Data Integration (Weather API)

```sql
-- Use OPENROWSET with JSON for REST API calls (requires enabling external scripts)
-- Alternative: use Azure Data Factory or Python/PowerShell scripts

CREATE PROCEDURE usp_LoadWeatherData
    @ApiKey NVARCHAR(200)
AS
BEGIN
    -- Example using staging table populated by external script
    -- The actual API call would be done via Python/PowerShell/ADF
    
    MERGE FactFleetTrips AS target
    USING (
        SELECT 
            TripKey,
            WeatherCondition
        FROM StagingWeatherData
    ) AS source
    ON target.TripKey = source.TripKey
    WHEN MATCHED THEN
        UPDATE SET target.WeatherCondition = source.WeatherCondition;
END
GO
```

### Pattern 3: Temporal Analysis with Window Functions

```sql
-- Compare current hour performance vs same hour last week
WITH CurrentHour AS (
    SELECT 
        dt.HourOfDay,
        COUNT(*) AS Operations,
        AVG(wo.CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.DateTime >= DATEADD(HOUR, -1, GETDATE())
        AND dt.DateTime < GETDATE()
    GROUP BY dt.HourOfDay
),
LastWeekSameHour AS (
    SELECT 
        dt.HourOfDay,
        COUNT(*) AS Operations,
        AVG(wo.CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE dt.DateTime >= DATEADD(HOUR, -169, GETDATE()) -- 168 hours + 1
        AND dt.DateTime < DATEADD(HOUR, -168, GETDATE())
    GROUP BY dt.HourOfDay
)
SELECT 
    c.HourOfDay,
    c.Operations AS CurrentOperations,
    l.Operations AS LastWeekOperations,
    c.AvgCycleTime AS CurrentCycleTime,
    l.AvgCycleTime AS LastWeekCycleTime,
    ((c.Operations - l.Operations) / CAST(l.Operations AS FLOAT)) * 100 AS VolumeChangePercent,
