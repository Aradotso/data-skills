---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform using MS SQL Server star schema and Power BI for warehouse, fleet, and inventory analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure multi-fact star schema for logistics"
  - "create Power BI dashboard for warehouse and fleet data"
  - "implement cross-modal supply chain analytics"
  - "query warehouse operations and fleet telemetry"
  - "build logistics KPI harmonization layer"
  - "deploy SQL Server data warehouse for logistics"
  - "integrate WMS and fleet tracking with Power BI"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that integrates warehouse operations, fleet telemetry, inventory management, and external data sources into a unified analytical layer. Built on MS SQL Server with Power BI visualization, it implements a multi-fact star schema architecture enabling cross-domain KPI analysis.

**Core capabilities:**
- Multi-fact data warehouse with time-phased dimensions
- Unified semantic layer linking warehouse, fleet, and supplier data
- Real-time Power BI dashboards with 15-minute refresh cycles
- Predictive bottleneck detection and fleet optimization
- Warehouse gravity zone modeling for spatial optimization
- Cross-fact KPI harmonization (e.g., inventory turnover vs. fuel costs)

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Database Schema Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation script
-- Assumes schema.sql exists in the repository
:r schema.sql
GO
```

3. **Configure data source connections:**
```json
// config.json (create from config_sample.json)
{
  "sql_server": {
    "host": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}",
    "port": 1433
  },
  "wms_api": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "api_key": "${WMS_API_KEY}"
  },
  "fleet_telemetry": {
    "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
    "api_key": "${FLEET_API_KEY}"
  },
  "refresh_interval_minutes": 15
}
```

4. **Import Power BI template:**
- Open Power BI Desktop
- File → Import → Power BI Template
- Select `LogiFleet_Pulse_Master.pbit`
- Enter SQL Server connection details when prompted

## Core Data Model

### Star Schema Architecture

The platform uses a multi-fact star schema with shared dimensions:

```sql
-- Fact Tables (grain definitions)
-- FactWarehouseOperations: One row per warehouse operation cycle
-- FactFleetTrips: One row per route segment
-- FactCrossDock: One row per cross-dock transfer
-- FactInventorySnapshot: One row per SKU per 15-minute time bucket

-- Dimension Tables
-- DimTime: 15-minute granularity with fiscal calendar
-- DimGeography: Hierarchical (continent → country → region → warehouse)
-- DimProduct: Product hierarchy with gravity scoring
-- DimSupplier: Supplier reliability metrics
-- DimVehicle: Fleet asset details
-- DimWarehouseZone: Storage zones with gravity classification
```

### Key Dimension: DimTime (Time-Phased)

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour TINYINT NOT NULL,
    MinuteBucket TINYINT NOT NULL, -- 0, 15, 30, 45
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    FiscalQuarter TINYINT NOT NULL,
    FiscalPeriod TINYINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Index for typical queries
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime 
ON DimTime (FullDateTime) INCLUDE (TimeKey, FiscalPeriod);
```

### Fact Table: FactWarehouseOperations

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    WarehouseZoneKey INT NOT NULL FOREIGN KEY REFERENCES DimWarehouseZone(ZoneKey),
    OperationType VARCHAR(20) NOT NULL, -- 'Putaway', 'Pick', 'Pack', 'Ship'
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT NULL, -- Time SKU spent in zone
    CycleTimeSeconds INT NOT NULL, -- Operation completion time
    EmployeeID INT NULL,
    BatchID VARCHAR(50) NULL,
    AnomalyFlag BIT DEFAULT 0
);

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseZoneKey, OperationType, QuantityHandled, DwellTimeMinutes);
```

### Fact Table: FactFleetTrips

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT NOT NULL FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKm DECIMAL(10,2) NOT NULL,
    FuelConsumedLiters DECIMAL(8,2) NOT NULL,
    IdleTimeMinutes INT NOT NULL,
    LoadingTimeMinutes INT NOT NULL,
    UnloadingTimeMinutes INT NOT NULL,
    AverageSpe DECIMAL(6,2) NULL,
    WeatherCondition VARCHAR(20) NULL, -- 'Clear', 'Rain', 'Snow'
    DelayReasonCode VARCHAR(10) NULL
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON FactFleetTrips (StartTimeKey, VehicleKey, DistanceKm, FuelConsumedLiters, IdleTimeMinutes);
```

## Common Queries and Patterns

### Cross-Fact KPI: Inventory Turnover vs. Fleet Utilization

```sql
-- Calculate inventory turnover by product category
-- and correlate with fleet idle time for deliveries
WITH InventoryTurnover AS (
    SELECT 
        p.CategoryName,
        dt.FiscalQuarter,
        SUM(fwo.QuantityHandled) / NULLIF(AVG(fis.QuantityOnHand), 0) AS TurnoverRatio
    FROM FactWarehouseOperations fwo
    JOIN DimProduct p ON fwo.ProductKey = p.ProductKey
    JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    JOIN FactInventorySnapshot fis ON p.ProductKey = fis.ProductKey 
        AND dt.DateKey = fis.DateKey
    WHERE fwo.OperationType = 'Ship'
        AND dt.FiscalYear = 2026
    GROUP BY p.CategoryName, dt.FiscalQuarter
),
FleetEfficiency AS (
    SELECT 
        dt.FiscalQuarter,
        AVG(CAST(fft.IdleTimeMinutes AS FLOAT) / 
            NULLIF(DATEDIFF(MINUTE, dtStart.FullDateTime, dtEnd.FullDateTime), 0)) AS IdlePercentage
    FROM FactFleetTrips fft
    JOIN DimTime dtStart ON fft.StartTimeKey = dtStart.TimeKey
    JOIN DimTime dtEnd ON fft.EndTimeKey = dtEnd.TimeKey
    JOIN DimTime dt ON fft.StartTimeKey = dt.TimeKey
    WHERE dt.FiscalYear = 2026
    GROUP BY dt.FiscalQuarter
)
SELECT 
    it.CategoryName,
    it.FiscalQuarter,
    it.TurnoverRatio,
    fe.IdlePercentage,
    -- Correlation hypothesis: higher turnover should reduce idle time
    it.TurnoverRatio * (1 - fe.IdlePercentage) AS EfficiencyScore
FROM InventoryTurnover it
JOIN FleetEfficiency fe ON it.FiscalQuarter = fe.FiscalQuarter
ORDER BY it.FiscalQuarter, it.CategoryName;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify SKUs in wrong gravity zones (high-velocity items in low-gravity zones)
SELECT 
    p.SKU,
    p.ProductName,
    wz.ZoneName,
    wz.GravityScore AS CurrentZoneGravity,
    COUNT(*) AS PickFrequency,
    AVG(fwo.CycleTimeSeconds) AS AvgCycleTime,
    -- Calculate recommended gravity score based on velocity
    CASE 
        WHEN COUNT(*) > 100 THEN 'High' -- Fast-moving
        WHEN COUNT(*) > 50 THEN 'Medium'
        ELSE 'Low'
    END AS RecommendedGravity
FROM FactWarehouseOperations fwo
JOIN DimProduct p ON fwo.ProductKey = p.ProductKey
JOIN DimWarehouseZone wz ON fwo.WarehouseZoneKey = wz.ZoneKey
JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE fwo.OperationType = 'Pick'
    AND dt.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY p.SKU, p.ProductName, wz.ZoneName, wz.GravityScore
HAVING 
    -- Misalignment: high frequency in low-gravity zone
    (COUNT(*) > 100 AND wz.GravityScore = 'Low')
    OR (COUNT(*) < 20 AND wz.GravityScore = 'High')
ORDER BY PickFrequency DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouse zones approaching capacity with trending dwell time increases
WITH DwellTimeTrend AS (
    SELECT 
        wz.ZoneName,
        dt.DateKey,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations fwo
    JOIN DimWarehouseZone wz ON fwo.WarehouseZoneKey = wz.ZoneKey
    JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE())
        AND fwo.DwellTimeMinutes IS NOT NULL
    GROUP BY wz.ZoneName, dt.DateKey
),
TrendAnalysis AS (
    SELECT 
        ZoneName,
        AVG(AvgDwellTime) AS CurrentAvgDwell,
        AVG(AvgDwellTime) - LAG(AVG(AvgDwellTime), 7) 
            OVER (PARTITION BY ZoneName ORDER BY DateKey) AS WeekOverWeekChange,
        SUM(OperationCount) AS TotalOps
    FROM DwellTimeTrend
    GROUP BY ZoneName, DateKey
)
SELECT 
    ZoneName,
    CurrentAvgDwell,
    WeekOverWeekChange,
    TotalOps,
    CASE 
        WHEN WeekOverWeekChange > 20 THEN 'High Risk'
        WHEN WeekOverWeekChange > 10 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM TrendAnalysis
WHERE WeekOverWeekChange IS NOT NULL
ORDER BY WeekOverWeekChange DESC;
```

### Fleet Route Optimization with Weather Correlation

```sql
-- Analyze fleet delays by weather condition and route
SELECT 
    geoOrigin.RegionName AS OriginRegion,
    geoDest.RegionName AS DestinationRegion,
    fft.WeatherCondition,
    COUNT(*) AS TripCount,
    AVG(DATEDIFF(MINUTE, dtStart.FullDateTime, dtEnd.FullDateTime)) AS AvgTripDurationMinutes,
    AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
    AVG(fft.FuelConsumedLiters) AS AvgFuelConsumption,
    -- Flag routes significantly impacted by weather
    CASE 
        WHEN fft.WeatherCondition IN ('Rain', 'Snow') 
            AND AVG(fft.IdleTimeMinutes) > 60 
        THEN 'Reroute Recommended'
        ELSE 'Normal'
    END AS Recommendation
FROM FactFleetTrips fft
JOIN DimGeography geoOrigin ON fft.OriginGeographyKey = geoOrigin.GeographyKey
JOIN DimGeography geoDest ON fft.DestinationGeographyKey = geoDest.GeographyKey
JOIN DimTime dtStart ON fft.StartTimeKey = dtStart.TimeKey
JOIN DimTime dtEnd ON fft.EndTimeKey = dtEnd.TimeKey
WHERE dtStart.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
GROUP BY geoOrigin.RegionName, geoDest.RegionName, fft.WeatherCondition
HAVING COUNT(*) > 5
ORDER BY AvgIdleTime DESC;
```

## Data Loading Procedures

### Incremental Warehouse Operations Load

```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load from external WMS table (assumes linked server or external table)
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseZoneKey, OperationType,
        QuantityHandled, DwellTimeMinutes, CycleTimeSeconds,
        EmployeeID, BatchID
    )
    SELECT 
        dt.TimeKey,
        p.ProductKey,
        wz.ZoneKey,
        wms.OperationType,
        wms.Quantity,
        wms.DwellTimeMinutes,
        wms.CycleTimeSeconds,
        wms.EmployeeID,
        wms.BatchID
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimTime dt ON 
        DATEADD(MINUTE, (DATEPART(MINUTE, wms.OperationDateTime) / 15) * 15, 
                DATEADD(HOUR, DATEDIFF(HOUR, 0, wms.OperationDateTime), 0)) = dt.FullDateTime
    INNER JOIN DimProduct p ON wms.SKU = p.SKU
    INNER JOIN DimWarehouseZone wz ON wms.ZoneCode = wz.ZoneCode
    WHERE wms.OperationDateTime > @LastLoadDateTime
        AND wms.OperationDateTime <= GETDATE();
    
    -- Update last load timestamp
    UPDATE ETL_Control
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Fleet Telemetry Streaming Load

```sql
CREATE PROCEDURE usp_LoadFleetTrips
    @JsonPayload NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Parse JSON from fleet telemetry API
    INSERT INTO FactFleetTrips (
        StartTimeKey, EndTimeKey, VehicleKey,
        OriginGeographyKey, DestinationGeographyKey,
        DistanceKm, FuelConsumedLiters, IdleTimeMinutes,
        LoadingTimeMinutes, UnloadingTimeMinutes,
        AverageSpeed, WeatherCondition
    )
    SELECT 
        dtStart.TimeKey,
        dtEnd.TimeKey,
        v.VehicleKey,
        geoOrigin.GeographyKey,
        geoDest.GeographyKey,
        JSON_VALUE(trip.value, '$.distance_km'),
        JSON_VALUE(trip.value, '$.fuel_liters'),
        JSON_VALUE(trip.value, '$.idle_minutes'),
        JSON_VALUE(trip.value, '$.loading_minutes'),
        JSON_VALUE(trip.value, '$.unloading_minutes'),
        JSON_VALUE(trip.value, '$.avg_speed'),
        JSON_VALUE(trip.value, '$.weather')
    FROM OPENJSON(@JsonPayload, '$.trips') trip
    CROSS APPLY (
        SELECT TimeKey FROM DimTime 
        WHERE FullDateTime = CAST(JSON_VALUE(trip.value, '$.start_time') AS DATETIME2)
    ) dtStart
    CROSS APPLY (
        SELECT TimeKey FROM DimTime 
        WHERE FullDateTime = CAST(JSON_VALUE(trip.value, '$.end_time') AS DATETIME2)
    ) dtEnd
    INNER JOIN DimVehicle v ON JSON_VALUE(trip.value, '$.vehicle_id') = v.VehicleID
    INNER JOIN DimGeography geoOrigin ON JSON_VALUE(trip.value, '$.origin_code') = geoOrigin.LocationCode
    INNER JOIN DimGeography geoDest ON JSON_VALUE(trip.value, '$.destination_code') = geoDest.LocationCode;
END;
GO
```

## Power BI Integration

### DAX Measures for Cross-Fact KPIs

```dax
// Measure: Warehouse Efficiency Score
Warehouse Efficiency = 
VAR TotalCycleTime = SUM(FactWarehouseOperations[CycleTimeSeconds])
VAR TotalOperations = COUNT(FactWarehouseOperations[OperationKey])
VAR AvgCycleTime = DIVIDE(TotalCycleTime, TotalOperations, 0)
VAR BenchmarkCycleTime = 120 -- seconds
RETURN
    DIVIDE(BenchmarkCycleTime, AvgCycleTime, 0)

// Measure: Fleet Idle Cost
Fleet Idle Cost = 
VAR IdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
VAR CostPerIdleHour = 25 -- USD
RETURN
    (IdleMinutes / 60) * CostPerIdleHour

// Measure: Cross-Fact KPI - Cost per Unit Shipped
Cost Per Unit Shipped = 
VAR FleetCost = [Fleet Idle Cost] + [Fleet Fuel Cost]
VAR UnitsShipped = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Ship"
    )
RETURN
    DIVIDE(FleetCost, UnitsShipped, 0)

// Measure: Dwell Time Anomaly Detection
Dwell Time Anomaly Flag = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DATEADD(DimTime[FullDateTime], -30, DAY)
    )
VAR Threshold = HistoricalAvg * 1.5
RETURN
    IF(CurrentDwell > Threshold, "Alert", "Normal")
```

### Power BI Refresh Configuration

Create a refresh schedule via Power BI Service or use Power BI REST API:

```powershell
# PowerShell script to trigger dataset refresh
$datasetId = "${POWERBI_DATASET_ID}"
$accessToken = "${POWERBI_ACCESS_TOKEN}"

$headers = @{
    "Authorization" = "Bearer $accessToken"
    "Content-Type" = "application/json"
}

$refreshUrl = "https://api.powerbi.com/v1.0/myorg/datasets/$datasetId/refreshes"

Invoke-RestMethod -Uri $refreshUrl -Method Post -Headers $headers
```

## Automated Alerting

### Stored Procedure for Threshold Alerts

```sql
CREATE PROCEDURE usp_GenerateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert: Fleet idle time exceeds 15% threshold
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'Fleet Idle Threshold',
        'High',
        'Vehicle ' + v.VehicleID + ' idle time: ' + 
            CAST(AVG(fft.IdleTimeMinutes) AS VARCHAR(10)) + ' min on route ' +
            geoOrigin.RegionName + ' to ' + geoDest.RegionName,
        GETDATE()
    FROM FactFleetTrips fft
    JOIN DimVehicle v ON fft.VehicleKey = v.VehicleKey
    JOIN DimGeography geoOrigin ON fft.OriginGeographyKey = geoOrigin.GeographyKey
    JOIN DimGeography geoDest ON fft.DestinationGeographyKey = geoDest.GeographyKey
    JOIN DimTime dt ON fft.StartTimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY v.VehicleID, geoOrigin.RegionName, geoDest.RegionName
    HAVING AVG(CAST(fft.IdleTimeMinutes AS FLOAT) / 
               NULLIF(DATEDIFF(MINUTE, dt.FullDateTime, 
                      (SELECT FullDateTime FROM DimTime WHERE TimeKey = fft.EndTimeKey)), 0)) > 0.15;
    
    -- Alert: Warehouse zone dwell time spike
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'Dwell Time Spike',
        'Medium',
        'Zone ' + wz.ZoneName + ' avg dwell time: ' + 
            CAST(AVG(fwo.DwellTimeMinutes) AS VARCHAR(10)) + ' min (threshold: 180 min)',
        GETDATE()
    FROM FactWarehouseOperations fwo
    JOIN DimWarehouseZone wz ON fwo.WarehouseZoneKey = wz.ZoneKey
    JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
        AND fwo.DwellTimeMinutes IS NOT NULL
    GROUP BY wz.ZoneName
    HAVING AVG(fwo.DwellTimeMinutes) > 180;
END;
GO

-- Schedule via SQL Agent job (run every 15 minutes)
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- For time-series queries on fact tables
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time_Product
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (QuantityHandled, CycleTimeSeconds);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Time_Vehicle
ON FactFleetTrips (StartTimeKey, VehicleKey)
INCLUDE (IdleTimeMinutes, FuelConsumedLiters);

-- For dimensional filtering
CREATE NONCLUSTERED INDEX IX_DimProduct_Category
ON DimProduct (CategoryName)
INCLUDE (ProductKey, SKU);

CREATE NONCLUSTERED INDEX IX_DimGeography_Region
ON DimGeography (RegionName)
INCLUDE (GeographyKey, LocationCode);
```

### Partitioning for Large Fact Tables

```sql
-- Partition FactWarehouseOperations by month
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401, 20260501, 20260601);

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly ALL TO ([PRIMARY]);

-- Rebuild table on partition scheme
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- ... other columns
    CONSTRAINT PK_FactWarehouseOps_Partitioned 
        PRIMARY KEY (TimeKey, OperationKey)
) ON PS_Monthly(TimeKey);
```

### Row-Level Security for Multi-Tenant Scenarios

```sql
-- Create security predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@RegionName VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS AccessGranted
    WHERE 
        @RegionName = USER_NAME() 
        OR IS_MEMBER('GlobalAdmin') = 1;
GO

-- Apply to DimGeography
CREATE SECURITY POLICY RegionSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(RegionName)
ON DimGeography
WITH (STATE = ON);
```

## Troubleshooting

### Common Issues

**Issue: Power BI refresh fails with timeout**
```sql
-- Check long-running queries
SELECT 
    r.session_id,
    r.start_time,
    r.status,
    r.command,
    SUBSTRING(qt.text, 1, 200) AS query_text,
    r.total_elapsed_time / 1000 AS elapsed_seconds
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) qt
WHERE r.database_id = DB_ID('LogiFleetPulse')
ORDER BY r.total_elapsed_time DESC;

-- Solution: Add columnstore indexes or partition large fact tables
```

**Issue: DimTime missing entries for new dates**
```sql
-- Populate DimTime for next 12 months
DECLARE @StartDate DATETIME2 = GETDATE();
DECLARE @EndDate DATETIME2 = DATEADD(YEAR, 1, @StartDate);

;WITH TimeSequence AS (
    SELECT @StartDate AS FullDateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, FullDateTime)
    FROM TimeSequence
    WHERE DATEADD(MINUTE, 15, FullDateTime) <= @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, Hour, MinuteBucket, DayOfWeek, DayName, FiscalYear, FiscalQuarter, FiscalPeriod, IsWeekend, IsHoliday)
SELECT 
    CAST(FORMAT(FullDateTime, 'yyyyMMddHHmm') AS INT),
    FullDateTime,
    CAST(FORMAT(FullDateTime, 'yyyyMMdd') AS INT),
    CAST(FullDateTime AS TIME),
    DATEPART(HOUR, FullDateTime),
    (DATEPART(MINUTE, FullDateTime) / 15) * 15,
    DATEPART(WEEKDAY, FullDateTime),
    DATENAME(WEEKDAY, FullDateTime),
    YEAR(FullDateTime),
    DATEPART(QUARTER, FullDateTime),
    MONTH(FullDateTime),
    CASE WHEN DATEPART(WEEKDAY, FullDateTime) IN (1, 7) THEN 1 ELSE 0 END,
    0 -- Holiday logic TBD
FROM TimeSequence
OPTION (MAXRECURSION 0);
```

**Issue: Cross-fact queries performing slowly**
```sql
-- Verify dimension relationships
SELECT 
    fk.name AS ForeignKeyName,
    tp.name AS ParentTable,
    tr.name AS ReferencedTable
FROM sys.foreign_keys fk
INNER JOIN sys.tables tp ON fk.parent_object_id = tp.object_id
INNER JOIN sys.tables tr ON fk.referenced_object_id = tr.object_id
WHERE tp.name LIKE 'Fact%';

-- Solution: Ensure all fact tables have indexed foreign keys
```

**Issue: Duplicate records in fact tables**
```sql
-- Identify duplicates in FactWarehouseOperations
WITH DuplicateCheck AS (
    SELECT 
        TimeKey, ProductKey, WarehouseZoneKey, OperationType,
        CycleTimeSeconds, BatchID,
        ROW_NUMBER() OVER (
            PARTITION BY TimeKey, ProductKey, WarehouseZoneKey, BatchID 
            ORDER BY OperationKey
        ) AS RowNum
    FROM FactWarehouseOperations
)
DELETE FROM DuplicateCheck WHERE RowNum > 1;

-- Prevention: Add unique constraint on business key
ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT UQ_WarehouseOps_BusinessKey 
UNIQUE (TimeKey, ProductKey, WarehouseZoneKey, BatchID);
```

## Advanced Patterns

### Temporal Elasticity Simulation

```sql
-- What-if analysis: Impact of 95% warehouse capacity on fleet utilization
WITH CapacityScenario AS (
    SELECT 
        dt.FiscalQuarter,
        wz.ZoneName,
        SUM(fwo.QuantityHandled) AS CurrentVolume,
        wz.MaxCapacity,
        SUM(fwo.QuantityHandled) / NULLIF(wz.MaxCapacity, 0) AS CurrentUtilization,
        -- Simulate 95% capacity
        CASE 
            WHEN SUM(fwo.QuantityHandled) / NULLIF(wz.MaxCapacity, 0) > 0.95 
            THEN SUM(fwo.QuantityHandled) * 1.2 -- Overflow requires extra trips
            ELSE SUM(fwo.QuantityHandled)
        END AS ProjectedVolume
    FROM FactWarehouseOperations fwo
    JOIN DimWarehouseZone wz ON fwo.WarehouseZoneKey = wz.ZoneKey
    JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.FiscalYear = 2026
    GROUP BY dt.FiscalQuarter, wz.ZoneName, wz.MaxCapacity
)
SELECT 
    cs.FiscalQuarter,
    SUM(cs.ProjectedVolume - cs.CurrentVolume) AS AdditionalVolumeNeeded,
    SUM(cs.
