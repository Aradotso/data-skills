---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and supply chain KPI harmonization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse tracking"
  - "implement cross-fact KPI queries for logistics"
  - "create warehouse gravity zone optimization reports"
  - "build real-time fleet telemetry dashboards"
  - "query multi-modal supply chain metrics"
  - "set up logistics predictive bottleneck alerts"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive MS SQL Server data warehouse and Power BI visualization template for logistics and supply chain analytics. It implements a multi-fact star schema that harmonizes:

- **Warehouse operations** (putaway, picking, packing, dwell time)
- **Fleet telemetry** (routes, fuel consumption, idle time, maintenance)
- **Cross-dock transfers** (inbound-to-outbound without storage)
- **Supplier reliability** metrics
- **Predictive bottleneck detection** using time-phased dimensions

The project provides SQL schema scripts, stored procedures for incremental loading, and Power BI templates (`.pbit`) with pre-built dashboards and KPI calculations.

## Installation & Setup

### Prerequisites

- **MS SQL Server 2019+** (Express, Standard, or Enterprise)
- **Power BI Desktop** (latest version)
- Database credentials with CREATE TABLE and CREATE PROCEDURE permissions
- Optional: Azure Synapse Analytics for external data enrichment

### Step 1: Deploy the SQL Schema

Clone or download the repository, then execute the schema creation script:

```sql
-- Connect to your SQL Server instance and run:
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the complete schema script (provided in repository)
-- This creates all dimension and fact tables with relationships
```

### Step 2: Create Core Tables

The project includes these primary tables:

```sql
-- Dimension: Time (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    FiscalPeriod VARCHAR(10),
    IsBusinessHour BIT NOT NULL,
    INDEX IX_DimTime_DateTime NONCLUSTERED (FullDateTime)
);

-- Dimension: Product with Gravity Score
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- Velocity + Value + Fragility composite
    IsActive BIT DEFAULT 1,
    INDEX IX_Product_SKU NONCLUSTERED (SKU)
);

-- Dimension: Geography (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(20), -- 'Warehouse', 'RouteNode', 'CrossDock'
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    INDEX IX_Geography_Type NONCLUSTERED (LocationType)
);

-- Fact: Warehouse Operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    Quantity INT,
    RecordedAt DATETIME DEFAULT GETDATE(),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_Warehouse_Time NONCLUSTERED (TimeKey),
    INDEX IX_Warehouse_Product NONCLUSTERED (ProductKey)
);

-- Fact: Fleet Trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightLbs INT,
    OnTimeDelivery BIT,
    RecordedAt DATETIME DEFAULT GETDATE(),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey),
    INDEX IX_Fleet_Time NONCLUSTERED (TimeKey),
    INDEX IX_Fleet_Vehicle NONCLUSTERED (VehicleID)
);
```

### Step 3: Populate Dimension Tables

```sql
-- Populate DimTime for the past 2 years with 15-minute intervals
DECLARE @StartDate DATETIME = DATEADD(YEAR, -2, GETDATE());
DECLARE @EndDate DATETIME = GETDATE();
DECLARE @CurrentDate DATETIME = @StartDate;

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (FullDateTime, DateKey, TimeOfDay, HourOfDay, DayOfWeek, IsBusinessHour)
    VALUES (
        @CurrentDate,
        CONVERT(INT, CONVERT(VARCHAR(8), @CurrentDate, 112)),
        CONVERT(TIME, @CurrentDate),
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        CASE WHEN DATEPART(HOUR, @CurrentDate) BETWEEN 8 AND 17 THEN 1 ELSE 0 END
    );
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate);
END;
```

## Key Commands & Operations

### Cross-Fact KPI Queries

**Query 1: Dwell Time vs. Fleet Idle Time Correlation**

```sql
-- Find products with high warehouse dwell time that also experience
-- fleet idle time during delivery
SELECT 
    p.SKU,
    p.ProductName,
    AVG(w.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdle,
    COUNT(DISTINCT f.TripKey) AS TotalTrips
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips f ON w.GeographyKey = f.OriginGeographyKey
    AND DATEDIFF(DAY, w.RecordedAt, f.RecordedAt) = 0
WHERE w.OperationType = 'Ship'
    AND w.DwellTimeMinutes > 72 * 60 -- More than 72 hours
GROUP BY p.SKU, p.ProductName
HAVING AVG(f.IdleTimeMinutes) > 30
ORDER BY AvgWarehouseDwell DESC;
```

**Query 2: Warehouse Gravity Zone Optimization**

```sql
-- Identify products that should be moved to higher-gravity zones
-- based on recent pick frequency
WITH RecentPickFrequency AS (
    SELECT 
        ProductKey,
        COUNT(*) AS PickCount,
        AVG(CycleTimeMinutes) AS AvgPickTime
    FROM FactWarehouseOperations
    WHERE OperationType = 'Pick'
        AND RecordedAt >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ProductKey
)
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore AS CurrentGravity,
    rpf.PickCount,
    rpf.AvgPickTime,
    CASE 
        WHEN rpf.PickCount > 100 AND p.GravityScore < 70 THEN 'Move to High-Gravity Zone'
        WHEN rpf.PickCount < 20 AND p.GravityScore > 50 THEN 'Move to Low-Gravity Zone'
        ELSE 'No Change Needed'
    END AS Recommendation
FROM DimProductGravity p
INNER JOIN RecentPickFrequency rpf ON p.ProductKey = rpf.ProductKey
WHERE p.IsActive = 1
ORDER BY rpf.PickCount DESC;
```

**Query 3: Predictive Bottleneck Detection**

```sql
-- Identify route segments with increasing idle time trends
-- (potential bottlenecks forming)
WITH WeeklyIdleTime AS (
    SELECT 
        f.OriginGeographyKey,
        f.DestinationGeographyKey,
        DATEPART(WEEK, t.FullDateTime) AS WeekNumber,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY f.OriginGeographyKey, f.DestinationGeographyKey, DATEPART(WEEK, t.FullDateTime)
),
TrendAnalysis AS (
    SELECT 
        OriginGeographyKey,
        DestinationGeographyKey,
        AVG(AvgIdleTime) AS OverallAvgIdle,
        MAX(AvgIdleTime) - MIN(AvgIdleTime) AS IdleTimeVariance
    FROM WeeklyIdleTime
    GROUP BY OriginGeographyKey, DestinationGeographyKey
)
SELECT 
    go.LocationName AS OriginWarehouse,
    gd.LocationName AS DestinationHub,
    ta.OverallAvgIdle,
    ta.IdleTimeVariance,
    CASE 
        WHEN ta.IdleTimeVariance > 20 THEN 'High Risk'
        WHEN ta.IdleTimeVariance > 10 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM TrendAnalysis ta
INNER JOIN DimGeography go ON ta.OriginGeographyKey = go.GeographyKey
INNER JOIN DimGeography gd ON ta.DestinationGeographyKey = gd.GeographyKey
WHERE ta.IdleTimeVariance > 10
ORDER BY ta.IdleTimeVariance DESC;
```

### Stored Procedures for Incremental Loading

```sql
-- Stored procedure: Load warehouse operations from external source
CREATE PROCEDURE sp_LoadWarehouseOperations
    @SourceTable NVARCHAR(200) -- External table or staging table name
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, OperationType, DwellTimeMinutes, CycleTimeMinutes, Quantity)
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        src.OperationType,
        src.DwellTimeMinutes,
        src.CycleTimeMinutes,
        src.Quantity
    FROM @SourceTable src
    INNER JOIN DimTime t ON CAST(src.OperationDateTime AS DATETIME) = t.FullDateTime
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    INNER JOIN DimGeography g ON src.WarehouseID = g.LocationID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations existing
        WHERE existing.TimeKey = t.TimeKey
            AND existing.ProductKey = p.ProductKey
            AND existing.GeographyKey = g.GeographyKey
            AND existing.OperationType = src.OperationType
    );
END;
GO

-- Stored procedure: Calculate and update product gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity based on 30-day pick velocity, average value, and fragility
    WITH ProductMetrics AS (
        SELECT 
            w.ProductKey,
            COUNT(*) AS PickFrequency,
            AVG(w.CycleTimeMinutes) AS AvgCycleTime,
            SUM(w.Quantity) AS TotalQuantity
        FROM FactWarehouseOperations w
        WHERE w.OperationType = 'Pick'
            AND w.RecordedAt >= DATEADD(DAY, -30, GETDATE())
        GROUP BY w.ProductKey
    )
    UPDATE p
    SET GravityScore = 
        (pm.PickFrequency * 0.5) + -- Velocity weight
        (100 / NULLIF(pm.AvgCycleTime, 0) * 0.3) + -- Efficiency weight
        (pm.TotalQuantity / 100.0 * 0.2) -- Volume weight
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO
```

## Configuration

### Connection String Setup

Store connection strings in environment variables:

```bash
# Set SQL Server connection
export LOGIFLEET_SQL_SERVER="Server=your-server.database.windows.net;Database=LogiFleetPulse;User Id=admin;Password=$SQL_PASSWORD;Encrypt=True;"

# Set Power BI service connection (if using automated refresh)
export POWERBI_WORKSPACE_ID="your-workspace-guid"
export POWERBI_CLIENT_ID="your-app-registration-id"
export POWERBI_CLIENT_SECRET="$POWERBI_SECRET"
```

### External Data Source Configuration

```sql
-- Configure external data source (e.g., for telemetry API)
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = HADOOP,
    LOCATION = 'https://telemetry-api.example.com/v1/data',
    CREDENTIAL = TelemetryAPICredential
);

-- Create external table for streaming fleet data
CREATE EXTERNAL TABLE ext_FleetTelemetry (
    VehicleID VARCHAR(50),
    Timestamp DATETIME,
    Latitude DECIMAL(10,6),
    Longitude DECIMAL(10,6),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = TelemetryAPI,
    LOCATION = '/fleet/realtime',
    FILE_FORMAT = JSONFormat
);
```

## Power BI Integration

### Loading the Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection string when prompted
3. Refresh the data model to validate connections

### Key DAX Measures

**Measure 1: Fleet Utilization Rate**

```dax
Fleet Utilization % = 
VAR TotalTripTime = SUM(FactFleetTrips[DistanceMiles]) / 55 * 60  -- Assume 55 mph avg
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN 
    DIVIDE(TotalTripTime, TotalTripTime + TotalIdleTime, 0) * 100
```

**Measure 2: Cross-Fact Efficiency Index**

```dax
Logistics Efficiency Index = 
VAR WarehouseCycleEfficiency = 
    DIVIDE(
        COUNTROWS(FactWarehouseOperations),
        SUM(FactWarehouseOperations[CycleTimeMinutes]),
        0
    ) * 100

VAR FleetFuelEfficiency = 
    DIVIDE(
        SUM(FactFleetTrips[DistanceMiles]),
        SUM(FactFleetTrips[FuelConsumedGallons]),
        0
    )

RETURN 
    (WarehouseCycleEfficiency * 0.4) + (FleetFuelEfficiency * 0.6)
```

**Measure 3: Time-Phased Dwell Alert**

```dax
High Dwell Alert = 
VAR CurrentDwellAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], TODAY(), -7, DAY)
    )
VAR PriorDwellAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATESINPERIOD(DimTime[FullDateTime], TODAY() - 14, -7, DAY)
    )
RETURN 
    IF(CurrentDwellAvg > PriorDwellAvg * 1.2, "⚠️ Alert", "✓ Normal")
```

## Common Patterns

### Pattern 1: Incremental Data Load with Change Detection

```sql
-- Create staging table for new records
CREATE TABLE staging_WarehouseOps (
    OperationID VARCHAR(50),
    OperationDateTime DATETIME,
    SKU VARCHAR(50),
    WarehouseID VARCHAR(50),
    OperationType VARCHAR(20),
    DwellTimeMinutes INT,
    CycleTimeMinutes INT,
    Quantity INT
);

-- Merge staging into fact table (upsert pattern)
MERGE INTO FactWarehouseOperations AS target
USING (
    SELECT 
        t.TimeKey,
        p.ProductKey,
        g.GeographyKey,
        s.OperationType,
        s.DwellTimeMinutes,
        s.CycleTimeMinutes,
        s.Quantity
    FROM staging_WarehouseOps s
    INNER JOIN DimTime t ON s.OperationDateTime = t.FullDateTime
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    INNER JOIN DimGeography g ON s.WarehouseID = g.LocationID
) AS source
ON target.TimeKey = source.TimeKey 
    AND target.ProductKey = source.ProductKey 
    AND target.GeographyKey = source.GeographyKey
WHEN MATCHED THEN 
    UPDATE SET 
        target.DwellTimeMinutes = source.DwellTimeMinutes,
        target.CycleTimeMinutes = source.CycleTimeMinutes,
        target.Quantity = source.Quantity
WHEN NOT MATCHED THEN
    INSERT (TimeKey, ProductKey, GeographyKey, OperationType, DwellTimeMinutes, CycleTimeMinutes, Quantity)
    VALUES (source.TimeKey, source.ProductKey, source.GeographyKey, source.OperationType, source.DwellTimeMinutes, source.CycleTimeMinutes, source.Quantity);

-- Clean staging table
TRUNCATE TABLE staging_WarehouseOps;
```

### Pattern 2: Role-Playing Dimensions (Time)

```sql
-- Create views for different time perspectives
CREATE VIEW vw_OperationTimeSnapshot AS
SELECT 
    w.*,
    t.FullDateTime AS OperationDateTime,
    t.HourOfDay,
    t.DayOfWeek,
    t.IsBusinessHour
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey;

CREATE VIEW vw_TripDepartureTime AS
SELECT 
    f.*,
    t.FullDateTime AS DepartureDateTime,
    t.HourOfDay AS DepartureHour
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey;
```

### Pattern 3: Automated Alert System

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY(1,1),
    MetricName VARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(10,2) NOT NULL,
    ComparisonOperator VARCHAR(2), -- '>', '<', '>=', '<=', '='
    AlertRecipients VARCHAR(500), -- Comma-separated emails
    IsActive BIT DEFAULT 1
);

-- Stored procedure to check and send alerts
CREATE PROCEDURE sp_CheckLogisticsAlerts
AS
BEGIN
    -- Check fleet idle time threshold
    DECLARE @AvgIdleTime DECIMAL(10,2);
    SELECT @AvgIdleTime = AVG(IdleTimeMinutes)
    FROM FactFleetTrips
    WHERE RecordedAt >= DATEADD(HOUR, -1, GETDATE());
    
    IF @AvgIdleTime > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'FleetIdleTimeMinutes' AND IsActive = 1)
    BEGIN
        -- Log alert (in production, trigger email via msdb.dbo.sp_send_dbmail)
        PRINT 'ALERT: Fleet idle time exceeded threshold: ' + CAST(@AvgIdleTime AS VARCHAR(10));
    END;
    
    -- Check warehouse dwell time threshold
    DECLARE @AvgDwellTime DECIMAL(10,2);
    SELECT @AvgDwellTime = AVG(DwellTimeMinutes)
    FROM FactWarehouseOperations
    WHERE OperationType = 'Ship'
        AND RecordedAt >= DATEADD(DAY, -1, GETDATE());
    
    IF @AvgDwellTime > (SELECT ThresholdValue FROM AlertThresholds WHERE MetricName = 'WarehouseDwellTimeMinutes' AND IsActive = 1)
    BEGIN
        PRINT 'ALERT: Warehouse dwell time exceeded threshold: ' + CAST(@AvgDwellTime AS VARCHAR(10));
    END;
END;
GO

-- Schedule this procedure to run every 15 minutes via SQL Agent
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom:** Queries joining FactWarehouseOperations and FactFleetTrips take >30 seconds

**Solution:** Create composite indexes on time and geography keys

```sql
-- Add composite indexes for common join patterns
CREATE NONCLUSTERED INDEX IX_Warehouse_TimeGeo 
ON FactWarehouseOperations (TimeKey, GeographyKey) 
INCLUDE (ProductKey, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_Fleet_TimeOrigin 
ON FactFleetTrips (TimeKey, OriginGeographyKey) 
INCLUDE (IdleTimeMinutes, FuelConsumedGallons);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Power BI Refresh Timeout

**Symptom:** Power BI refresh fails with "Query timeout expired"

**Solution:** Implement incremental refresh policy

1. Add a RangeStart and RangeEnd parameter to the Power BI query:

```powerquery
let
    Source = Sql.Database(ServerName, DatabaseName, [Query="
        SELECT * FROM FactWarehouseOperations
        WHERE RecordedAt >= #(lf)'" & DateTime.ToText(RangeStart, "yyyy-MM-dd HH:mm:ss") & "' AND 
              RecordedAt < #(lf)'" & DateTime.ToText(RangeEnd, "yyyy-MM-dd HH:mm:ss") & "'
    "])
in
    Source
```

2. Configure incremental refresh in Power BI Desktop (Model tab)
3. Set to refresh only last 7 days, store 2 years

### Issue: Duplicate Records in Fact Tables

**Symptom:** SUM aggregations show inflated values

**Solution:** Add unique constraint and deduplication logic

```sql
-- Create unique index to prevent duplicates
CREATE UNIQUE NONCLUSTERED INDEX UX_Warehouse_Operation
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey, OperationType, RecordedAt);

-- Deduplication query (run as part of ETL)
WITH DuplicateRecords AS (
    SELECT 
        OperationKey,
        ROW_NUMBER() OVER (
            PARTITION BY TimeKey, ProductKey, GeographyKey, OperationType 
            ORDER BY RecordedAt DESC
        ) AS RowNum
    FROM FactWarehouseOperations
)
DELETE FROM DuplicateRecords WHERE RowNum > 1;
```

### Issue: Gravity Score Not Updating

**Symptom:** DimProductGravity.GravityScore remains static despite operational changes

**Solution:** Schedule the gravity update procedure

```sql
-- Verify the update logic is running
EXEC sp_UpdateProductGravityScores;

-- Check if products have recent pick activity
SELECT 
    p.SKU,
    p.GravityScore,
    COUNT(*) AS RecentPicks
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
WHERE w.OperationType = 'Pick'
    AND w.RecordedAt >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.SKU, p.GravityScore
ORDER BY RecentPicks DESC;

-- Create SQL Agent job to run daily at 2 AM
USE msdb;
GO
EXEC sp_add_job @job_name = 'UpdateProductGravityScores';
EXEC sp_add_jobstep @job_name = 'UpdateProductGravityScores', 
    @step_name = 'Execute Update', 
    @command = 'EXEC LogiFleetPulse.dbo.sp_UpdateProductGravityScores;';
EXEC sp_add_schedule @schedule_name = 'DailyAt2AM', 
    @freq_type = 4, 
    @freq_interval = 1, 
    @active_start_time = 020000;
EXEC sp_attach_schedule @job_name = 'UpdateProductGravityScores', 
    @schedule_name = 'DailyAt2AM';
```

## Advanced Use Cases

### Scenario: Multi-Warehouse Network Optimization

```sql
-- Identify which products should be transferred between warehouses
-- to minimize overall dwell time
WITH WarehouseInventoryVelocity AS (
    SELECT 
        w.GeographyKey,
        w.ProductKey,
        AVG(w.DwellTimeMinutes) AS AvgDwell,
        COUNT(*) AS PickFrequency,
        SUM(w.Quantity) AS TotalQuantity
    FROM FactWarehouseOperations w
    WHERE w.RecordedAt >= DATEADD(DAY, -60, GETDATE())
    GROUP BY w.GeographyKey, w.ProductKey
),
TransferCandidates AS (
    SELECT 
        slow.GeographyKey AS SourceWarehouse,
        fast.GeographyKey AS TargetWarehouse,
        slow.ProductKey,
        slow.AvgDwell AS SourceDwell,
        fast.AvgDwell AS TargetDwell,
        slow.TotalQuantity AS AvailableQuantity,
        fast.PickFrequency AS TargetDemand
    FROM WarehouseInventoryVelocity slow
    INNER JOIN WarehouseInventoryVelocity fast 
        ON slow.ProductKey = fast.ProductKey 
        AND slow.GeographyKey <> fast.GeographyKey
    WHERE slow.AvgDwell > 72 * 60 -- High dwell at source
        AND fast.PickFrequency > slow.PickFrequency * 2 -- Much higher demand at target
        AND slow.TotalQuantity > 100 -- Enough inventory to transfer
)
SELECT 
    gs.LocationName AS FromWarehouse,
    gt.LocationName AS ToWarehouse,
    p.SKU,
    p.ProductName,
    tc.SourceDwell / 60.0 AS SourceDwellHours,
    tc.TargetDwell / 60.0 AS TargetDwellHours,
    tc.AvailableQuantity AS SuggestedTransferQty,
    tc.TargetDemand AS TargetWeeklyPicks
FROM TransferCandidates tc
INNER JOIN DimGeography gs ON tc.SourceWarehouse = gs.GeographyKey
INNER JOIN DimGeography gt ON tc.TargetWarehouse = gt.GeographyKey
INNER JOIN DimProductGravity p ON tc.ProductKey = p.ProductKey
ORDER BY (tc.SourceDwell - tc.TargetDwell) DESC;
```

This skill provides comprehensive coverage for deploying, querying, and maintaining the LogiFleet Pulse logistics data warehouse system.
