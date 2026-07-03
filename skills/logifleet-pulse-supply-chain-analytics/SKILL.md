---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics intelligence with multi-fact star schema and real-time fleet optimization
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logifleet warehouse and fleet intelligence system
  - configure power bi logistics dashboard with sql server
  - implement multi-fact star schema for supply chain data
  - create warehouse gravity zones and fleet optimization
  - build cross-modal logistics kpi dashboard
  - connect logifleet pulse to warehouse management system
  - setup real-time fleet telemetry analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines MS SQL Server data warehousing with Power BI visualization to create a unified view of warehouse operations, fleet management, and supply chain metrics. It uses a multi-fact star schema architecture with time-phased dimensions to enable cross-modal analytics and predictive bottleneck detection.

**Key Capabilities:**
- Multi-fact star schema linking warehouse, fleet, and supplier data
- Real-time dashboard updates (15-minute granularity)
- Warehouse gravity zone optimization
- Predictive fleet maintenance triage
- Cross-fact KPI harmonization
- Temporal elasticity modeling for scenario planning

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate database permissions (CREATE TABLE, CREATE PROCEDURE, etc.)

### Step 1: Deploy SQL Schema

Clone the repository and locate the SQL schema files:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

Connect to your SQL Server instance and execute the schema creation script:

```sql
-- Connect to your target database first
USE [YourLogisticsDB];
GO

-- Execute the main schema script
-- This creates dimension tables, fact tables, views, and stored procedures
:r schema/01_create_dimensions.sql
:r schema/02_create_facts.sql
:r schema/03_create_relationships.sql
:r schema/04_create_views.sql
:r schema/05_create_procedures.sql
GO
```

### Step 2: Configure Data Sources

Update the configuration file with your data source connections:

```json
// config.json
{
  "dataSources": {
    "wms": {
      "type": "REST_API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "authToken": "${WMS_API_TOKEN}"
    },
    "fleetTelemetry": {
      "type": "STREAMING",
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "apiKey": "${FLEET_API_KEY}"
    },
    "erp": {
      "type": "SQL_SERVER",
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "integratedSecurity": true
    }
  },
  "refreshInterval": 900,
  "alertThresholds": {
    "fleetIdleTimePercent": 15,
    "warehouseDwellHours": 72,
    "temperatureToleranceMinutes": 20
  }
}
```

### Step 3: Import Power BI Template

Open Power BI Desktop and import the template:

1. Open `LogiFleet_Pulse_Master.pbit`
2. Enter SQL Server connection details when prompted
3. Configure row-level security based on user roles
4. Publish to Power BI Service (if using cloud deployment)

## Core Data Model

### Dimension Tables

#### DimTime (Time-Phased Dimension)

```sql
-- Query DimTime for 15-minute bucket granularity
SELECT 
    TimeKey,
    FullDateTime,
    HourOfDay,
    DayOfWeek,
    FiscalPeriod,
    IsBusinessHour,
    IsWeekend
FROM DimTime
WHERE FullDateTime >= DATEADD(HOUR, -24, GETDATE());
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
-- View product gravity scores
SELECT 
    ProductKey,
    ProductSKU,
    ProductName,
    GravityScore,
    VelocityTier,
    ValueTier,
    FragilityScore,
    RecommendedZone
FROM DimProductGravity
WHERE GravityScore > 75  -- High-priority items
ORDER BY GravityScore DESC;
```

#### DimGeography (Hierarchical Location)

```sql
-- Geographic hierarchy for route and warehouse mapping
SELECT 
    GeographyKey,
    Continent,
    Country,
    Region,
    LocationName,
    LocationType,  -- 'WAREHOUSE', 'ROUTE_NODE', 'CUSTOMER_SITE'
    Latitude,
    Longitude
FROM DimGeography
WHERE LocationType = 'WAREHOUSE';
```

### Fact Tables

#### FactWarehouseOperations

```sql
-- Warehouse operational metrics
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    OperationType VARCHAR(50),  -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    UnitsProcessed INT,
    OperatorID VARCHAR(50),
    ZoneID VARCHAR(20),
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- Query warehouse dwell time by gravity zone
SELECT 
    pg.RecommendedZone,
    pg.VelocityTier,
    AVG(fwo.DwellTimeHours) AS AvgDwellHours,
    SUM(fwo.UnitsProcessed) AS TotalUnits,
    COUNT(DISTINCT fwo.ProductKey) AS DistinctSKUs
FROM FactWarehouseOperations fwo
JOIN DimProductGravity pg ON fwo.ProductKey = pg.ProductKey
JOIN DimTime t ON fwo.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY pg.RecommendedZone, pg.VelocityTier
ORDER BY AvgDwellHours DESC;
```

#### FactFleetTrips

```sql
-- Fleet telemetry and route performance
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    RouteID VARCHAR(50),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    TotalDelayMinutes DECIMAL(10,2),
    DelayReason VARCHAR(100),
    AverageSpeed DECIMAL(10,2),
    MaxSpeed DECIMAL(10,2),
    CONSTRAINT FK_Fleet_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- Calculate fleet idle time percentage by route
SELECT 
    fft.RouteID,
    gOrg.LocationName AS Origin,
    gDst.LocationName AS Destination,
    AVG(fft.IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, tStart.FullDateTime, tEnd.FullDateTime), 0) * 100) AS IdleTimePercent,
    AVG(fft.FuelConsumedLiters) AS AvgFuelConsumption,
    COUNT(*) AS TripCount
FROM FactFleetTrips fft
JOIN DimTime tStart ON fft.StartTimeKey = tStart.TimeKey
JOIN DimTime tEnd ON fft.EndTimeKey = tEnd.TimeKey
JOIN DimGeography gOrg ON fft.OriginGeographyKey = gOrg.GeographyKey
JOIN DimGeography gDst ON fft.DestinationGeographyKey = gDst.GeographyKey
WHERE tStart.FullDateTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY fft.RouteID, gOrg.LocationName, gDst.LocationName
HAVING AVG(fft.IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, tStart.FullDateTime, tEnd.FullDateTime), 0) * 100) > 15
ORDER BY IdleTimePercent DESC;
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Stored procedure for incremental warehouse data loading
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDateTime DATETIME,
    @EndDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new warehouse operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        ProductKey,
        GeographyKey,
        OperationType,
        DurationMinutes,
        DwellTimeHours,
        UnitsProcessed,
        OperatorID,
        ZoneID
    )
    SELECT 
        t.TimeKey,
        pg.ProductKey,
        g.GeographyKey,
        stg.OperationType,
        stg.DurationMinutes,
        stg.DwellTimeHours,
        stg.UnitsProcessed,
        stg.OperatorID,
        stg.ZoneID
    FROM StagingWarehouseOperations stg
    JOIN DimTime t ON DATEADD(MINUTE, (DATEPART(MINUTE, stg.OperationDateTime) / 15) * 15, 
                              DATEADD(HOUR, DATEDIFF(HOUR, 0, stg.OperationDateTime), 0)) = t.FullDateTime
    JOIN DimProductGravity pg ON stg.ProductSKU = pg.ProductSKU
    JOIN DimGeography g ON stg.WarehouseCode = g.LocationCode
    WHERE stg.OperationDateTime >= @StartDateTime
      AND stg.OperationDateTime < @EndDateTime
      AND NOT EXISTS (
          SELECT 1 FROM FactWarehouseOperations fwo
          WHERE fwo.TimeKey = t.TimeKey
            AND fwo.ProductKey = pg.ProductKey
            AND fwo.OperatorID = stg.OperatorID
      );
    
    RETURN @@ROWCOUNT;
END;
GO

-- Execute the load procedure
DECLARE @RowsLoaded INT;
EXEC @RowsLoaded = sp_LoadWarehouseOperations 
    @StartDateTime = '2026-07-01 00:00:00',
    @EndDateTime = '2026-07-02 00:00:00';
PRINT 'Rows loaded: ' + CAST(@RowsLoaded AS VARCHAR(10));
```

### Cross-Fact KPI Calculation

```sql
-- Calculate dwell time vs. fleet idle cost correlation
CREATE VIEW vw_CrossFactDwellVsIdleCost AS
SELECT 
    t.FiscalPeriod,
    pg.VelocityTier,
    g.Region,
    AVG(fwo.DwellTimeHours) AS AvgDwellHours,
    SUM(fft.IdleTimeMinutes * 0.5) AS TotalIdleCostUSD,  -- $0.50 per minute idle cost
    COUNT(DISTINCT fwo.ProductKey) AS DistinctProducts,
    COUNT(DISTINCT fft.VehicleID) AS DistinctVehicles
FROM FactWarehouseOperations fwo
JOIN DimTime t ON fwo.TimeKey = t.TimeKey
JOIN DimProductGravity pg ON fwo.ProductKey = pg.ProductKey
JOIN DimGeography g ON fwo.GeographyKey = g.GeographyKey
LEFT JOIN FactFleetTrips fft ON fft.StartTimeKey = t.TimeKey 
    AND fft.OriginGeographyKey = g.GeographyKey
GROUP BY t.FiscalPeriod, pg.VelocityTier, g.Region;
GO

-- Query the cross-fact view
SELECT * FROM vw_CrossFactDwellVsIdleCost
WHERE FiscalPeriod = '2026-Q3'
ORDER BY TotalIdleCostUSD DESC;
```

### Predictive Bottleneck Detection

```sql
-- Stored procedure for bottleneck prediction
CREATE PROCEDURE sp_PredictBottlenecks
    @ForecastHours INT = 24
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Identify patterns from historical data
    WITH HistoricalPatterns AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            fwo.ZoneID,
            AVG(fwo.DurationMinutes) AS AvgDuration,
            STDEV(fwo.DurationMinutes) AS StdDevDuration,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations fwo
        JOIN DimTime t ON fwo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(DAY, -90, GETDATE())
        GROUP BY t.HourOfDay, t.DayOfWeek, fwo.ZoneID
    ),
    FutureTimeSlots AS (
        SELECT 
            TimeKey,
            HourOfDay,
            DayOfWeek,
            FullDateTime
        FROM DimTime
        WHERE FullDateTime BETWEEN GETDATE() AND DATEADD(HOUR, @ForecastHours, GETDATE())
    )
    SELECT 
        fts.FullDateTime AS PredictedTime,
        hp.ZoneID,
        hp.AvgDuration AS ExpectedDurationMinutes,
        hp.StdDevDuration,
        hp.OperationCount AS HistoricalVolume,
        CASE 
            WHEN hp.AvgDuration > 45 THEN 'HIGH_RISK'
            WHEN hp.AvgDuration > 30 THEN 'MEDIUM_RISK'
            ELSE 'LOW_RISK'
        END AS BottleneckRisk
    FROM FutureTimeSlots fts
    JOIN HistoricalPatterns hp ON fts.HourOfDay = hp.HourOfDay 
        AND fts.DayOfWeek = hp.DayOfWeek
    WHERE hp.AvgDuration > 30  -- Only show potential bottlenecks
    ORDER BY fts.FullDateTime, hp.AvgDuration DESC;
END;
GO

-- Execute bottleneck prediction
EXEC sp_PredictBottlenecks @ForecastHours = 48;
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

```dax
// Fleet Idle Time Percentage
Fleet Idle Time % = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    CALCULATE(
        SUMX(
            FactFleetTrips,
            DATEDIFF(
                RELATED(DimTime[FullDateTime]),
                RELATED(DimTime[FullDateTime]),
                MINUTE
            )
        )
    ),
    0
) * 100

// Warehouse Velocity Index
Warehouse Velocity Index = 
DIVIDE(
    COUNTROWS(FactWarehouseOperations),
    DISTINCTCOUNT(FactWarehouseOperations[ProductKey]) * 
    CALCULATE(
        MAX(FactWarehouseOperations[DwellTimeHours])
    ),
    0
) * 1000

// Cross-Fact Efficiency Score
Efficiency Score = 
VAR WarehouseScore = 100 - AVERAGE(FactWarehouseOperations[DwellTimeHours]) / 72 * 100
VAR FleetScore = 100 - [Fleet Idle Time %]
RETURN (WarehouseScore + FleetScore) / 2

// Predictive Alert Trigger
Bottleneck Alert = 
IF(
    AND(
        [Warehouse Velocity Index] < 50,
        [Fleet Idle Time %] > 15
    ),
    "CRITICAL",
    IF(
        OR(
            [Warehouse Velocity Index] < 75,
            [Fleet Idle Time %] > 10
        ),
        "WARNING",
        "NORMAL"
    )
)
```

### Row-Level Security Configuration

```dax
// RLS for regional managers
[Region] = USERNAME() 
    || USEROBJECTID() IN (
        SELECT UserID FROM SecurityTable 
        WHERE Role = "GLOBAL_ADMIN"
    )

// RLS for warehouse supervisors
[GeographyKey] IN (
    SELECT GeographyKey FROM UserWarehouseAccess
    WHERE UserEmail = USERPRINCIPALNAME()
)
```

## Automated Alerting System

```sql
-- Create alert monitoring table
CREATE TABLE AlertLog (
    AlertID BIGINT IDENTITY(1,1) PRIMARY KEY,
    AlertType VARCHAR(50),
    Severity VARCHAR(20),
    EntityID VARCHAR(100),
    MetricValue DECIMAL(10,2),
    ThresholdValue DECIMAL(10,2),
    AlertDateTime DATETIME DEFAULT GETDATE(),
    NotificationSent BIT DEFAULT 0,
    ResolvedDateTime DATETIME NULL
);
GO

-- Stored procedure for real-time alert generation
CREATE PROCEDURE sp_GenerateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check fleet idle time threshold
    INSERT INTO AlertLog (AlertType, Severity, EntityID, MetricValue, ThresholdValue)
    SELECT 
        'FLEET_IDLE_EXCESSIVE',
        'HIGH',
        VehicleID,
        IdleTimePercent,
        15.0
    FROM (
        SELECT 
            VehicleID,
            AVG(IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, tStart.FullDateTime, tEnd.FullDateTime), 0) * 100) AS IdleTimePercent
        FROM FactFleetTrips fft
        JOIN DimTime tStart ON fft.StartTimeKey = tStart.TimeKey
        JOIN DimTime tEnd ON fft.EndTimeKey = tEnd.TimeKey
        WHERE tStart.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        GROUP BY VehicleID
    ) AS FleetMetrics
    WHERE IdleTimePercent > 15
      AND NOT EXISTS (
          SELECT 1 FROM AlertLog al
          WHERE al.EntityID = FleetMetrics.VehicleID
            AND al.AlertType = 'FLEET_IDLE_EXCESSIVE'
            AND al.AlertDateTime >= DATEADD(HOUR, -1, GETDATE())
      );
    
    -- Check warehouse dwell time threshold
    INSERT INTO AlertLog (AlertType, Severity, EntityID, MetricValue, ThresholdValue)
    SELECT 
        'WAREHOUSE_DWELL_EXCESSIVE',
        CASE WHEN AvgDwellHours > 96 THEN 'CRITICAL' ELSE 'MEDIUM' END,
        ZoneID,
        AvgDwellHours,
        72.0
    FROM (
        SELECT 
            ZoneID,
            AVG(DwellTimeHours) AS AvgDwellHours
        FROM FactWarehouseOperations
        WHERE TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE FullDateTime >= DATEADD(HOUR, -12, GETDATE())
        )
        GROUP BY ZoneID
    ) AS WarehouseMetrics
    WHERE AvgDwellHours > 72
      AND NOT EXISTS (
          SELECT 1 FROM AlertLog al
          WHERE al.EntityID = WarehouseMetrics.ZoneID
            AND al.AlertType = 'WAREHOUSE_DWELL_EXCESSIVE'
            AND al.AlertDateTime >= DATEADD(HOUR, -6, GETDATE())
      );
    
    -- Temperature tolerance alert (for perishables)
    INSERT INTO AlertLog (AlertType, Severity, EntityID, MetricValue, ThresholdValue)
    SELECT 
        'TEMPERATURE_BREACH',
        'CRITICAL',
        CONCAT(VehicleID, '_', CAST(StartTimeKey AS VARCHAR)),
        TemperatureDriftMinutes,
        20.0
    FROM (
        SELECT 
            VehicleID,
            StartTimeKey,
            SUM(CASE WHEN TemperatureInRange = 0 THEN 1 ELSE 0 END) AS TemperatureDriftMinutes
        FROM FleetTemperatureLog  -- Assuming telemetry table exists
        WHERE LogDateTime >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY VehicleID, StartTimeKey
    ) AS TempMetrics
    WHERE TemperatureDriftMinutes > 20;
    
    RETURN @@ROWCOUNT;
END;
GO

-- Schedule via SQL Server Agent Job (example)
-- Job Name: LogiFleet_AlertMonitoring
-- Schedule: Every 15 minutes
-- Step Command: EXEC sp_GenerateAlerts;
```

## Common Patterns & Use Cases

### Pattern 1: Gravity Zone Optimization Analysis

```sql
-- Identify products that should be reassigned to different zones
WITH CurrentPlacement AS (
    SELECT 
        pg.ProductKey,
        pg.ProductSKU,
        pg.RecommendedZone AS CurrentZone,
        pg.GravityScore,
        AVG(fwo.DwellTimeHours) AS AvgDwellTime,
        SUM(fwo.UnitsProcessed) AS TotalVolume
    FROM DimProductGravity pg
    JOIN FactWarehouseOperations fwo ON pg.ProductKey = fwo.ProductKey
    WHERE fwo.TimeKey IN (
        SELECT TimeKey FROM DimTime 
        WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE())
    )
    GROUP BY pg.ProductKey, pg.ProductSKU, pg.RecommendedZone, pg.GravityScore
),
OptimalPlacement AS (
    SELECT 
        ProductKey,
        CASE 
            WHEN GravityScore >= 80 THEN 'ZONE_A'  -- High-gravity, close to shipping
            WHEN GravityScore >= 60 THEN 'ZONE_B'
            WHEN GravityScore >= 40 THEN 'ZONE_C'
            ELSE 'ZONE_D'
        END AS OptimalZone
    FROM CurrentPlacement
)
SELECT 
    cp.ProductSKU,
    cp.CurrentZone,
    op.OptimalZone,
    cp.GravityScore,
    cp.AvgDwellTime,
    cp.TotalVolume,
    CASE 
        WHEN cp.CurrentZone <> op.OptimalZone THEN 'REASSIGN_RECOMMENDED'
        ELSE 'CURRENT_OPTIMAL'
    END AS Action
FROM CurrentPlacement cp
JOIN OptimalPlacement op ON cp.ProductKey = op.ProductKey
WHERE cp.CurrentZone <> op.OptimalZone
ORDER BY cp.GravityScore DESC, cp.TotalVolume DESC;
```

### Pattern 2: Route Efficiency vs. Warehouse Dwell Correlation

```sql
-- Cross-fact analysis: Does faster warehouse processing correlate with route efficiency?
SELECT 
    g.Region,
    t.DayOfWeek,
    AVG(fwo.DwellTimeHours) AS AvgWarehouseDwell,
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdle,
    AVG(fft.FuelConsumedLiters / NULLIF(fft.DistanceKM, 0)) AS AvgFuelEfficiency,
    COUNT(DISTINCT fwo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT fft.TripKey) AS FleetTrips,
    CORR(fwo.DwellTimeHours, fft.IdleTimeMinutes) OVER (PARTITION BY g.Region) AS Correlation
FROM FactWarehouseOperations fwo
JOIN DimTime t ON fwo.TimeKey = t.TimeKey
JOIN DimGeography g ON fwo.GeographyKey = g.GeographyKey
JOIN FactFleetTrips fft ON fft.OriginGeographyKey = g.GeographyKey 
    AND ABS(DATEDIFF(HOUR, t.FullDateTime, tStart.FullDateTime)) <= 4
JOIN DimTime tStart ON fft.StartTimeKey = tStart.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -60, GETDATE())
GROUP BY g.Region, t.DayOfWeek
ORDER BY Correlation DESC;
```

### Pattern 3: Supplier Reliability Impact on Operations

```sql
-- Correlate supplier reliability with warehouse processing times
SELECT 
    ds.SupplierName,
    ds.LeadTimeVariance,
    ds.DefectPercentage,
    AVG(fwo.DurationMinutes) AS AvgProcessingTime,
    AVG(fwo.DwellTimeHours) AS AvgDwellTime,
    COUNT(*) AS TotalShipments,
    SUM(CASE WHEN fwo.DurationMinutes > 60 THEN 1 ELSE 0 END) AS DelayedShipments
FROM FactWarehouseOperations fwo
JOIN DimProductGravity pg ON fwo.ProductKey = pg.ProductKey
JOIN DimSupplierReliability ds ON pg.SupplierKey = ds.SupplierKey
WHERE fwo.OperationType = 'PUTAWAY'
  AND fwo.TimeKey IN (
      SELECT TimeKey FROM DimTime 
      WHERE FullDateTime >= DATEADD(DAY, -90, GETDATE())
  )
GROUP BY ds.SupplierName, ds.LeadTimeVariance, ds.DefectPercentage
HAVING ds.LeadTimeVariance > 5  -- High variance suppliers
ORDER BY AvgProcessingTime DESC;
```

## Troubleshooting

### Issue: Slow Dashboard Refresh Times

**Symptom:** Power BI dashboards take longer than 15 minutes to refresh.

**Solution:**
1. Check for missing indexes on fact table foreign keys:

```sql
-- Create missing indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeKey 
ON FactWarehouseOperations(TimeKey) INCLUDE (ProductKey, DwellTimeHours);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_StartTimeKey 
ON FactFleetTrips(StartTimeKey) INCLUDE (VehicleID, IdleTimeMinutes);

-- Review existing index fragmentation
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
  AND ips.page_count > 1000
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Rebuild fragmented indexes
ALTER INDEX IX_FactWarehouseOps_TimeKey ON FactWarehouseOperations REBUILD;
```

2. Partition large fact tables by time range:

```sql
-- Create partition function for monthly partitions
CREATE PARTITION FUNCTION pf_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401, 20260501, 20260601);

-- Create partition scheme
CREATE PARTITION SCHEME ps_MonthlyPartition
AS PARTITION pf_MonthlyPartition ALL TO ([PRIMARY]);

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same columns as before
    ...
) ON ps_MonthlyPartition(TimeKey);
```

### Issue: Cross-Fact Queries Return Unexpected Results

**Symptom:** KPIs that combine warehouse and fleet data show inconsistent values.

**Solution:**
1. Verify time granularity alignment:

```sql
-- Check for misaligned time keys
SELECT 
    'Warehouse' AS Source,
    MIN(t.FullDateTime) AS MinTime,
    MAX(t.FullDateTime) AS MaxTime,
    COUNT(DISTINCT fwo.TimeKey) AS DistinctTimeKeys
FROM FactWarehouseOperations fwo
JOIN DimTime t ON fwo.TimeKey = t.TimeKey
UNION ALL
SELECT 
    'Fleet',
    MIN(t.FullDateTime),
    MAX(t.FullDateTime),
    COUNT(DISTINCT fft.StartTimeKey)
FROM FactFleetTrips fft
JOIN DimTime t ON fft.StartTimeKey = t.TimeKey;
```

2. Ensure consistent grain in bridge queries:

```sql
-- Use explicit time window for cross-fact joins
SELECT 
    DATEADD(MINUTE, (DATEPART(MINUTE, t.FullDateTime) / 15) * 15, 
            DATEADD(HOUR, DATEDIFF(HOUR, 0, t.FullDateTime), 0)) AS TimeWindow,
    COUNT(DISTINCT fwo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT fft.TripKey) AS FleetTrips
FROM DimTime t
LEFT JOIN FactWarehouseOperations fwo ON fwo.TimeKey = t.TimeKey
LEFT JOIN FactFleetTrips fft ON ABS(DATEDIFF(MINUTE, t.FullDateTime, tStart.FullDateTime)) <= 15
WHERE t.FullDateTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY DATEADD(MINUTE, (DATEPART(MINUTE, t.FullDateTime) / 15) * 15, 
                 DATEADD(HOUR, DATEDIFF(HOUR, 0, t.FullDateTime), 0));
```

### Issue: Alert Spam from Stored Procedure

**Symptom:** sp_GenerateAlerts creates duplicate alerts every 15 minutes.

**Solution:**
1. Add deduplication logic with temporal
