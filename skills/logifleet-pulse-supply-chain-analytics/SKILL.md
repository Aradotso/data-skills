---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics data warehouse with multi-fact star schema for fleet, warehouse, and supply chain KPI harmonization
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure power bi for fleet and warehouse dashboards"
  - "create multi-fact star schema for logistics"
  - "integrate warehouse and fleet telemetry data"
  - "build cross-modal supply chain analytics"
  - "implement warehouse gravity zones"
  - "set up predictive bottleneck detection"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for logistics and supply chain analytics. It implements a multi-fact star schema that harmonizes:

- **Warehouse operations** (receiving, putaway, picking, packing, dwell time)
- **Fleet telemetry** (routes, fuel consumption, idle time, maintenance)
- **Cross-dock transfers** (inbound-to-outbound without storage)
- **External data** (weather, traffic, supplier reliability)

The platform uses time-phased dimensions, bridge tables for many-to-many relationships, and composite KPIs that link warehouse efficiency with fleet performance.

Key differentiators:
- **Cross-fact KPI harmonization** — correlate inventory turnover with fuel burn rates
- **Warehouse Gravity Zones™** — spatial optimization based on pick frequency, value, and fragility
- **Adaptive fleet triage** — prioritize maintenance by revenue impact
- **Temporal elasticity modeling** — run what-if scenarios across time-phased data

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Database permissions: CREATE TABLE, CREATE VIEW, CREATE PROCEDURE

### Step 1: Deploy SQL Schema

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Enable snapshot isolation for time-phased queries
ALTER DATABASE LogiFleetPulse
SET ALLOW_SNAPSHOT_ISOLATION ON;
ALTER DATABASE LogiFleetPulse
SET READ_COMMITTED_SNAPSHOT ON;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Day INT,
    Hour INT,
    Minute15Block INT, -- 0-3 for 15-minute intervals
    DayOfWeek NVARCHAR(10),
    FiscalPeriod NVARCHAR(10),
    IsBusinessHour BIT
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    NodeID NVARCHAR(50) UNIQUE NOT NULL,
    NodeType NVARCHAR(20), -- 'Warehouse', 'DistributionCenter', 'RouteSegment'
    Continent NVARCHAR(50),
    Country NVARCHAR(50),
    Region NVARCHAR(50),
    City NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    UnitValue DECIMAL(10,2),
    FragilityScore INT, -- 1-10
    PickFrequencyPerDay DECIMAL(8,2),
    GravityScore AS (PickFrequencyPerDay * UnitValue / NULLIF(FragilityScore, 0)) PERSISTED,
    OptimalZone NVARCHAR(20) -- 'High', 'Medium', 'Low'
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    AvgLeadTimeDays DECIMAL(5,2),
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore INT, -- 0-100
    ReliabilityTier NVARCHAR(20) -- 'Platinum', 'Gold', 'Silver', 'Bronze'
);

CREATE TABLE DimFleetVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) UNIQUE NOT NULL,
    VehicleType NVARCHAR(50), -- 'Box Truck', 'Refrigerated', 'Flatbed'
    Capacity DECIMAL(8,2),
    FuelType NVARCHAR(20),
    AcquisitionDate DATE,
    MaintenanceSchedule NVARCHAR(50)
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    ErrorCount INT,
    StorageZone NVARCHAR(20)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    VehicleKey INT FOREIGN KEY REFERENCES DimFleetVehicle(VehicleKey),
    StartTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TotalDistanceKm DECIMAL(8,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    OnTimeDelivery BIT,
    WeatherDelay BIT,
    TrafficDelay BIT
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    Quantity INT,
    TransferTimeMinutes INT
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey);

-- Create columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse 
ON FactWarehouseOperations(TimeKey, GeographyKey, ProductKey, Quantity, DwellTimeMinutes);
```

### Step 2: Populate Dimension Tables

```sql
-- Generate time dimension (15-minute granularity for 2 years)
DECLARE @StartDate DATETIME2 = '2025-01-01';
DECLARE @EndDate DATETIME2 = '2027-01-01';

;WITH TimeCTE AS (
    SELECT @StartDate AS dt
    UNION ALL
    SELECT DATEADD(MINUTE, 15, dt)
    FROM TimeCTE
    WHERE dt < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Day, Hour, Minute15Block, DayOfWeek, FiscalPeriod, IsBusinessHour)
SELECT 
    CONVERT(INT, FORMAT(dt, 'yyyyMMddHHmm')),
    dt,
    YEAR(dt),
    DATEPART(QUARTER, dt),
    MONTH(dt),
    DAY(dt),
    DATEPART(HOUR, dt),
    DATEPART(MINUTE, dt) / 15,
    DATENAME(WEEKDAY, dt),
    CONCAT('FY', YEAR(dt), 'Q', DATEPART(QUARTER, dt)),
    CASE WHEN DATEPART(HOUR, dt) BETWEEN 8 AND 17 AND DATEPART(WEEKDAY, dt) NOT IN (1, 7) THEN 1 ELSE 0 END
FROM TimeCTE
OPTION (MAXRECURSION 0);
```

### Step 3: Create Stored Procedures for Data Loading

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LoadStartTime DATETIME2,
    @LoadEndTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assuming source data is in staging tables
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, Quantity, DwellTimeMinutes, ProcessingTimeMinutes, ErrorCount, StorageZone
    )
    SELECT 
        CONVERT(INT, FORMAT(stg.OperationDateTime, 'yyyyMMddHHmm')),
        geo.GeographyKey,
        prod.ProductKey,
        supp.SupplierKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime),
        stg.ProcessingTimeMinutes,
        stg.ErrorCount,
        stg.StorageZone
    FROM StagingWarehouseOps stg
    INNER JOIN DimGeography geo ON stg.WarehouseID = geo.NodeID
    INNER JOIN DimProductGravity prod ON stg.SKU = prod.SKU
    LEFT JOIN DimSupplierReliability supp ON stg.SupplierID = supp.SupplierID
    WHERE stg.OperationDateTime BETWEEN @LoadStartTime AND @LoadEndTime;
    
    RETURN @@ROWCOUNT;
END;
GO

-- Cross-fact KPI view: Inventory turnover vs. fleet efficiency
CREATE VIEW vw_CrossFactLogisticsKPI AS
SELECT 
    t.Year,
    t.Month,
    g.Region,
    p.Category,
    SUM(wo.Quantity) AS TotalUnitsProcessed,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.TotalDistanceKm, 0)) AS AvgFuelEfficiency,
    SUM(CASE WHEN ft.IdleTimeMinutes > ft.TotalDistanceKm * 2 THEN 1 ELSE 0 END) AS HighIdleTrips,
    AVG(p.GravityScore) AS AvgGravityScore
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON g.GeographyKey = ft.OriginGeographyKey AND t.TimeKey = ft.StartTimeKey
GROUP BY t.Year, t.Month, g.Region, p.Category;
GO
```

### Step 4: Configure Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Connect to your SQL Server instance
3. Update connection parameters:
   - Server: `${SQL_SERVER_INSTANCE}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use credentials from environment)

```m
// Power Query M language - Connection setup
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_INSTANCE"), 
        "LogiFleetPulse",
        [Query="SELECT * FROM vw_CrossFactLogisticsKPI"]
    )
in
    Source
```

## Key Configuration

### Environment Variables

Set these in your deployment environment:

```bash
# SQL Server connection
export SQL_SERVER_INSTANCE="your-server.database.windows.net"
export SQL_DB_NAME="LogiFleetPulse"
export SQL_USER="${SQL_ADMIN_USER}"
export SQL_PASSWORD="${SQL_ADMIN_PASSWORD}"

# External API integrations (optional)
export WEATHER_API_KEY="${WEATHER_API_KEY}"
export TRAFFIC_API_KEY="${TRAFFIC_API_KEY}"

# Alert thresholds
export DWELL_TIME_THRESHOLD_MINUTES="120"
export FLEET_IDLE_THRESHOLD_PERCENT="15"
export FUEL_EFFICIENCY_MIN="8.5"
```

### Row-Level Security in Power BI

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName NVARCHAR(50),
    UserEmail NVARCHAR(200),
    AllowedRegions NVARCHAR(MAX), -- JSON array
    AllowedCategories NVARCHAR(MAX) -- JSON array
);

-- Example: Restrict by region
CREATE FUNCTION fn_SecurityFilter(@UserEmail NVARCHAR(200))
RETURNS TABLE
AS
RETURN
(
    SELECT Region
    FROM SecurityRoles
    CROSS APPLY OPENJSON(AllowedRegions) WITH (Region NVARCHAR(50) '$')
    WHERE UserEmail = @UserEmail
);
GO

-- Apply in Power BI model (DAX):
-- [Region] IN VALUES(fn_SecurityFilter(USERPRINCIPALNAME()))
```

## Common Patterns & Use Cases

### Pattern 1: Identify High Dwell Time SKUs in Wrong Gravity Zones

```sql
-- Find products with high pick frequency stuck in low-gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone AS RecommendedZone,
    wo.StorageZone AS CurrentZone,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
  AND p.OptimalZone = 'High'
  AND wo.StorageZone IN ('Low', 'Medium')
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, wo.StorageZone
HAVING AVG(wo.DwellTimeMinutes) > 60
ORDER BY AvgDwellTime DESC;
```

### Pattern 2: Fleet Idling Cost by Route Segment

```sql
-- Calculate idle time cost by geography and time of day
DECLARE @FuelCostPerLiter DECIMAL(5,2) = 1.45; -- USD
DECLARE @IdleFuelRateLitersPerHour DECIMAL(4,2) = 2.5;

SELECT 
    g.Region,
    g.City,
    t.Hour,
    COUNT(ft.TripKey) AS TotalTrips,
    SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
    SUM(ft.IdleTimeMinutes / 60.0 * @IdleFuelRateLitersPerHour * @FuelCostPerLiter) AS IdleCost,
    AVG(ft.IdleTimeMinutes * 1.0 / NULLIF(DATEDIFF(MINUTE, ts.FullDateTime, te.FullDateTime), 0)) AS IdlePercentage
FROM FactFleetTrips ft
INNER JOIN DimGeography g ON ft.OriginGeographyKey = g.GeographyKey
INNER JOIN DimTime ts ON ft.StartTimeKey = ts.TimeKey
INNER JOIN DimTime te ON ft.EndTimeKey = te.TimeKey
INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
WHERE ts.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY g.Region, g.City, t.Hour
HAVING SUM(ft.IdleTimeMinutes) > 0
ORDER BY IdleCost DESC;
```

### Pattern 3: Predictive Bottleneck Detection (Temporal Elasticity)

```sql
-- Simulate warehouse capacity increase and predict impact on dwell time
WITH HistoricalBaseline AS (
    SELECT 
        p.Category,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        STDEV(wo.DwellTimeMinutes) AS StdDevDwellTime,
        SUM(wo.Quantity) AS TotalVolume
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.TimeKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMddHHmm'))
    GROUP BY p.Category
),
ScenarioSimulation AS (
    SELECT 
        Category,
        AvgDwellTime,
        TotalVolume,
        -- Assume 20% capacity increase reduces dwell time by 15% (empirical)
        AvgDwellTime * 0.85 AS ProjectedDwellTime,
        (AvgDwellTime - AvgDwellTime * 0.85) * TotalVolume / 60.0 AS HoursSaved
    FROM HistoricalBaseline
)
SELECT 
    Category,
    AvgDwellTime AS CurrentAvgDwellTime,
    ProjectedDwellTime,
    HoursSaved AS EstimatedHoursSavedPerQuarter
FROM ScenarioSimulation
ORDER BY HoursSaved DESC;
```

### Pattern 4: Cross-Dock Efficiency Analysis

```sql
-- Measure transfer time variance at cross-dock facilities
SELECT 
    g.City AS CrossDockLocation,
    p.Category,
    COUNT(cd.CrossDockKey) AS TransferCount,
    AVG(cd.TransferTimeMinutes) AS AvgTransferTime,
    STDEV(cd.TransferTimeMinutes) AS TransferTimeVariance,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY cd.TransferTimeMinutes) OVER (PARTITION BY g.City) AS P95TransferTime
FROM FactCrossDock cd
INNER JOIN FactFleetTrips ft ON cd.InboundTripKey = ft.TripKey
INNER JOIN DimGeography g ON ft.DestinationGeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
WHERE g.NodeType = 'CrossDock'
GROUP BY g.City, p.Category
HAVING COUNT(cd.CrossDockKey) > 10
ORDER BY TransferTimeVariance DESC;
```

## Power BI DAX Measures

### Composite KPI: Warehouse-Fleet Synchronization Index

```dax
// Measure: Synchronization Index (0-100)
// Higher = better alignment between warehouse processing speed and fleet availability
SyncIndex = 
VAR AvgWarehouseProcessing = AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes])
VAR AvgFleetIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR ProcessingEfficiency = 1 / (1 + AvgWarehouseProcessing / 60)
VAR FleetUtilization = 1 - (AvgFleetIdleTime / (AvgFleetIdleTime + 60))
RETURN
    (ProcessingEfficiency + FleetUtilization) / 2 * 100
```

### Predictive Bottleneck Heatmap

```dax
// Measure: Bottleneck Risk Score
BottleneckRisk = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalP90 = PERCENTILEX.INC(
    FILTER(ALL(FactWarehouseOperations), FactWarehouseOperations[TimeKey] < MIN(DimTime[TimeKey])),
    FactWarehouseOperations[DwellTimeMinutes],
    0.90
)
VAR RiskScore = (CurrentDwell - HistoricalP90) / HistoricalP90 * 100
RETURN
    IF(RiskScore > 0, RiskScore, 0)
```

## Alert Configuration

### Automated Email Alerts via SQL Server Agent

```sql
-- Create alert stored procedure
CREATE PROCEDURE usp_SendHighDwellAlert
AS
BEGIN
    DECLARE @AlertThreshold INT = CONVERT(INT, '${DWELL_TIME_THRESHOLD_MINUTES}');
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    SELECT @AlertMessage = STRING_AGG(
        CONCAT('SKU: ', p.SKU, ', Zone: ', wo.StorageZone, ', Dwell: ', wo.DwellTimeMinutes, ' min'),
        CHAR(13) + CHAR(10)
    )
    FROM (
        SELECT TOP 10 ProductKey, StorageZone, DwellTimeMinutes
        FROM FactWarehouseOperations
        WHERE TimeKey >= CONVERT(INT, FORMAT(DATEADD(HOUR, -1, GETDATE()), 'yyyyMMddHHmm'))
          AND DwellTimeMinutes > @AlertThreshold
        ORDER BY DwellTimeMinutes DESC
    ) wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey;
    
    IF @AlertMessage IS NOT NULL
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'operations@company.com',
            @subject = 'High Dwell Time Alert',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule to run every 15 minutes via SQL Server Agent
```

## Troubleshooting

### Issue: Slow cross-fact queries

**Solution:** Ensure columnstore indexes are created and statistics are updated:

```sql
-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;

-- Check index fragmentation
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30;

-- Rebuild fragmented indexes
ALTER INDEX ALL ON FactWarehouseOperations REBUILD;
```

### Issue: Time dimension missing records

**Solution:** Regenerate time dimension with extended range:

```sql
-- Delete and regenerate
TRUNCATE TABLE DimTime;

-- Re-run time generation script with new date range
DECLARE @StartDate DATETIME2 = '2024-01-01';
DECLARE @EndDate DATETIME2 = '2028-01-01';
-- (use script from Step 2)
```

### Issue: Power BI refresh fails with timeout

**Solution:** Implement incremental refresh:

1. In Power BI, go to Model → Manage Parameters
2. Create `RangeStart` and `RangeEnd` parameters
3. Filter fact tables: `WHERE TimeKey >= RangeStart AND TimeKey < RangeEnd`
4. Configure incremental refresh policy: detect changes by TimeKey

### Issue: Gravity scores not updating

**Solution:** Recalculate computed column:

```sql
-- Drop and recreate computed column
ALTER TABLE DimProductGravity DROP COLUMN GravityScore;
ALTER TABLE DimProductGravity ADD GravityScore AS (PickFrequencyPerDay * UnitValue / NULLIF(FragilityScore, 0)) PERSISTED;
```

## Advanced: External Data Integration

### Weather API Integration (Azure Logic App + SQL)

```sql
-- Create external data table
CREATE TABLE ExternalWeather (
    WeatherKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    WindSpeed DECIMAL(5,2),
    Conditions NVARCHAR(50)
);

-- Link weather delays to fleet trips
SELECT 
    ft.TripKey,
    ft.TotalDistanceKm,
    ft.WeatherDelay,
    w.Conditions,
    w.Precipitation
FROM FactFleetTrips ft
INNER JOIN ExternalWeather w ON ft.StartTimeKey = w.TimeKey 
    AND ft.OriginGeographyKey = w.GeographyKey
WHERE ft.WeatherDelay = 1
  AND w.Precipitation > 10; -- mm
```

## Performance Tuning

### Partition fact tables by time

```sql
-- Create partition function (monthly)
CREATE PARTITION FUNCTION pf_MonthlyTime (INT)
AS RANGE RIGHT FOR VALUES (202501, 202502, 202503, 202504, 202505, 202506);

CREATE PARTITION SCHEME ps_MonthlyTime
AS PARTITION pf_MonthlyTime
ALL TO ([PRIMARY]);

-- Move fact table to partitioned scheme
CREATE TABLE FactWarehouseOperations_New (
    -- same schema as original
) ON ps_MonthlyTime(TimeKey);

-- Switch partitions (minimizes downtime)
ALTER TABLE FactWarehouseOperations SWITCH TO FactWarehouseOperations_New;
```

### Memory-optimized tables for staging

```sql
-- Enable memory-optimized tables
ALTER DATABASE LogiFleetPulse ADD FILEGROUP LogiFleet_MemOpt CONTAINS MEMORY_OPTIMIZED_DATA;
ALTER DATABASE LogiFleetPulse ADD FILE (
    NAME = 'LogiFleet_MemOpt_File',
    FILENAME = 'C:\Data\LogiFleet_MemOpt'
) TO FILEGROUP LogiFleet_MemOpt;

-- Create staging table
CREATE TABLE StagingWarehouseOps (
    -- columns here
) WITH (MEMORY_OPTIMIZED = ON, DURABILITY = SCHEMA_ONLY);
```

## Best Practices

1. **Incremental loading**: Always use `@LoadStartTime` and `@LoadEndTime` parameters
2. **Time-phased snapshots**: Run dimension updates every 15 minutes to match fact grain
3. **Bridge tables**: Use for many-to-many (e.g., routes visiting multiple warehouses)
4. **Surrogate keys**: Never use business keys (SKU, VehicleID) in fact table relationships
5. **Audit columns**: Add `LoadDateTime` and `SourceSystem` to all fact tables
6. **Data lineage**: Tag each row with batch ID for troubleshooting

## Summary

LogiFleet Pulse provides a complete data warehouse solution for logistics analytics. Key capabilities:

- **Multi-fact star schema** linking warehouse, fleet, and external data
- **15-minute time granularity** for near-real-time operational insights
- **Computed columns** for gravity scores and derived KPIs
- **Cross-fact views** for harmonized metrics (e.g., inventory turnover vs. fuel efficiency)
- **Power BI integration** with incremental refresh and row-level security
- **Alert framework** via SQL Server Agent and email notifications

Use this skill to deploy, configure, and optimize the platform for supply chain decision-making.
