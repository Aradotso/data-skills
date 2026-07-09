---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "configure power bi logistics dashboard"
  - "deploy sql server warehouse schema for fleet management"
  - "implement multi-fact star schema for logistics"
  - "create supply chain analytics data model"
  - "build warehouse gravity zone optimization"
  - "integrate fleet telemetry with warehouse operations"
  - "query cross-modal supply chain KPIs"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing template for supply chain and logistics analytics. It provides a multi-fact star schema combining warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. Built on MS SQL Server with Power BI visualization, it enables cross-fact KPI analysis like correlating warehouse dwell time with fleet routing efficiency.

**Primary Language:** SQL (T-SQL for MS SQL Server)  
**Visualization:** Power BI (`.pbit` templates)  
**Use Cases:** 3PL operations, retail distribution, food logistics, warehouse optimization, fleet management

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs, or sample data files

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script (typically named schema_deploy.sql)
-- This creates all dimension and fact tables, relationships, and stored procedures
```

**Key tables created:**

- `FactWarehouseOperations` - putaway, picking, packing events
- `FactFleetTrips` - route segments, fuel, idle time
- `FactCrossDock` - transfer operations
- `DimTime` - 15-minute granularity time dimension
- `DimGeography` - hierarchical location data
- `DimProductGravity` - velocity-based product classification
- `DimSupplierReliability` - supplier performance metrics

### Step 2: Configure Data Sources

Create a configuration file or update connection strings:

```sql
-- Example: Configure external data source for API ingestion
CREATE EXTERNAL DATA SOURCE WarehouseAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://yourwms.blob.core.windows.net/data',
    CREDENTIAL = WMS_CREDENTIAL
);

-- Create external table for streaming ingestion
CREATE EXTERNAL TABLE ext_WarehouseEvents (
    EventID BIGINT,
    EventTime DATETIME2,
    WarehouseID INT,
    OperationType VARCHAR(50),
    SKU VARCHAR(100),
    Quantity INT,
    DurationMinutes DECIMAL(10,2)
)
WITH (
    DATA_SOURCE = WarehouseAPI,
    LOCATION = 'warehouse_events/',
    FILE_FORMAT = JSON_FORMAT
);
```

**Environment variables for connection strings:**

```sql
-- Reference environment variables in connection logic
-- Example in application layer or SSIS packages:
-- Server: ${SQL_SERVER_HOST}
-- Database: ${SQL_DATABASE_NAME}
-- User: ${SQL_USER}
-- Password: ${SQL_PASSWORD}
```

### Step 3: Load Initial Data

```sql
-- Stored procedure for incremental data loading
EXEC dbo.sp_LoadWarehouseOperations
    @StartDate = '2026-01-01',
    @EndDate = '2026-01-31',
    @SourceType = 'API';

EXEC dbo.sp_LoadFleetTrips
    @StartDate = '2026-01-01',
    @EndDate = '2026-01-31';

-- Refresh time dimension for new date ranges
EXEC dbo.sp_PopulateTimeDimension
    @StartDate = '2026-01-01',
    @EndDate = '2026-12-31';
```

### Step 4: Connect Power BI

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter connection parameters:
   - Server: Your SQL Server instance
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server
3. Power BI auto-detects relationships from SQL foreign keys
4. Publish to Power BI Service for scheduled refresh

## Core Data Model Patterns

### Multi-Fact Star Schema

```sql
-- Query pattern: Cross-fact KPI analysis
-- Example: Correlate warehouse dwell time with fleet idle time by product category

SELECT
    p.ProductCategory,
    g.WarehouseRegion,
    t.FiscalYear,
    t.FiscalQuarter,
    
    -- Warehouse metrics
    AVG(w.DwellTimeHours) AS AvgDwellTime,
    SUM(w.UnitsHandled) AS TotalUnitsHandled,
    
    -- Fleet metrics
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleTime,
    AVG(f.FuelConsumptionGallons) AS AvgFuelPerTrip,
    
    -- Cross-fact derived KPI
    (AVG(f.IdleTimeMinutes) / NULLIF(AVG(w.DwellTimeHours * 60), 0)) AS IdleToDwellRatio

FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
    LEFT JOIN FactFleetTrips f ON w.ShipmentKey = f.ShipmentKey
        AND f.TimeKey BETWEEN t.TimeKey AND t.TimeKey + 96 -- within 24 hours

WHERE t.FiscalYear = 2026
    AND p.ProductCategory IN ('Electronics', 'Perishables')

GROUP BY
    p.ProductCategory,
    g.WarehouseRegion,
    t.FiscalYear,
    t.FiscalQuarter

ORDER BY IdleToDwellRatio DESC;
```

### Warehouse Gravity Zones

```sql
-- Calculate product gravity score (velocity + value + fragility)
-- Higher score = closer to shipping dock placement recommended

UPDATE DimProductGravity
SET GravityScore =
    (PickFrequencyRank * 0.5) +  -- 50% weight on velocity
    (UnitValueRank * 0.3) +       -- 30% weight on value
    (FragilityScore * 0.2)        -- 20% weight on handling care
WHERE LastCalculatedDate < DATEADD(DAY, -7, GETDATE());

-- Query: Recommend zone reassignments
SELECT
    p.SKU,
    p.ProductCategory,
    p.CurrentZoneID,
    z.OptimalZoneID,
    z.DistanceToDockMeters AS CurrentDistance,
    oz.DistanceToDockMeters AS OptimalDistance,
    (z.DistanceToDockMeters - oz.DistanceToDockMeters) AS DistanceSavings
FROM DimProductGravity p
    INNER JOIN WarehouseZones z ON p.CurrentZoneID = z.ZoneID
    CROSS APPLY (
        SELECT TOP 1 ZoneID, DistanceToDockMeters
        FROM WarehouseZones
        WHERE GravityMinScore <= p.GravityScore
            AND GravityMaxScore >= p.GravityScore
            AND AvailableCapacityUnits >= p.AverageInventoryUnits
        ORDER BY DistanceToDockMeters ASC
    ) oz
WHERE z.ZoneID <> oz.ZoneID
    AND p.GravityScore > 75  -- High-gravity items only
ORDER BY DistanceSavings DESC;
```

### Fleet Triage Engine

```sql
-- Proactive maintenance queue ranked by revenue impact
-- Combines telemetry diagnostics with shipment value

WITH FleetIssues AS (
    SELECT
        f.VehicleID,
        f.TripID,
        f.TelemetryEngineCode,
        f.TelemetryTirePressure,
        f.LoadWeightKg,
        s.ShipmentValue,
        s.IsPerishable,
        
        -- Calculate severity score
        CASE
            WHEN f.TelemetryEngineCode LIKE 'CRITICAL%' THEN 100
            WHEN f.TelemetryTirePressure < 30 THEN 75
            WHEN f.TelemetryBrakePadMM < 3 THEN 60
            ELSE 0
        END AS SeverityScore,
        
        -- Revenue at risk
        s.ShipmentValue * 
        CASE WHEN s.IsPerishable = 1 THEN 1.5 ELSE 1.0 END AS RevenueAtRisk
        
    FROM FactFleetTrips f
        INNER JOIN Shipments s ON f.ShipmentKey = s.ShipmentKey
    WHERE f.TripStatus = 'InProgress'
        AND f.TripStartTime >= DATEADD(HOUR, -24, GETDATE())
)
SELECT
    VehicleID,
    TripID,
    SeverityScore,
    RevenueAtRisk,
    -- Priority ranking
    (SeverityScore * 0.6 + (RevenueAtRisk / 1000) * 0.4) AS TriageScore
FROM FleetIssues
WHERE SeverityScore > 0
ORDER BY TriageScore DESC;
```

## Common Analytical Queries

### Cross-Modal Bottleneck Detection

```sql
-- Identify operations where warehouse delays correlate with fleet delays
-- Uses time-phased join to find causal relationships

DECLARE @ThresholdMinutes INT = 30;

SELECT
    w.WarehouseID,
    w.OperationType,
    f.RouteID,
    COUNT(*) AS IncidentCount,
    AVG(w.DurationMinutes) AS AvgWarehouseDelay,
    AVG(f.DelayMinutes) AS AvgFleetDelay,
    CORR(w.DurationMinutes, f.DelayMinutes) AS DelayCorrelation
FROM FactWarehouseOperations w
    INNER JOIN FactFleetTrips f
        ON w.ShipmentKey = f.ShipmentKey
        AND f.TripStartTime BETWEEN
            DATEADD(MINUTE, -60, w.CompletionTime)
            AND DATEADD(MINUTE, 60, w.CompletionTime)
WHERE w.DurationMinutes > w.StandardDurationMinutes + @ThresholdMinutes
    AND f.DelayMinutes > 0
    AND w.OperationTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY w.WarehouseID, w.OperationType, f.RouteID
HAVING COUNT(*) >= 5 AND CORR(w.DurationMinutes, f.DelayMinutes) > 0.5
ORDER BY DelayCorrelation DESC, IncidentCount DESC;
```

### Temporal Elasticity Simulation

```sql
-- Simulate impact of warehouse capacity changes on fleet utilization
-- Uses historical patterns to project outcomes

WITH CapacityScenarios AS (
    SELECT 80 AS CapacityPct UNION ALL
    SELECT 85 UNION ALL
    SELECT 90 UNION ALL
    SELECT 95
),
HistoricalPatterns AS (
    SELECT
        FLOOR(w.CapacityUtilization / 5) * 5 AS CapacityBucket,
        AVG(f.UtilizationPct) AS AvgFleetUtilization,
        AVG(f.IdleTimeMinutes) AS AvgIdleTime,
        COUNT(DISTINCT f.TripID) AS TripCount
    FROM FactWarehouseOperations w
        INNER JOIN FactFleetTrips f ON w.DateKey = f.DateKey
    WHERE w.OperationTime >= DATEADD(MONTH, -6, GETDATE())
    GROUP BY FLOOR(w.CapacityUtilization / 5) * 5
)
SELECT
    cs.CapacityPct AS ScenarioCapacity,
    hp.AvgFleetUtilization AS ProjectedFleetUtilization,
    hp.AvgIdleTime AS ProjectedIdleTime,
    hp.TripCount AS BasedOnTrips
FROM CapacityScenarios cs
    LEFT JOIN HistoricalPatterns hp
        ON cs.CapacityPct BETWEEN hp.CapacityBucket - 2 AND hp.CapacityBucket + 2
ORDER BY cs.CapacityPct;
```

## Automated Alerting

```sql
-- Stored procedure for threshold-based alerts
-- Called via SQL Agent job every 15 minutes

CREATE PROCEDURE dbo.sp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertTable TABLE (
        AlertType VARCHAR(100),
        Entity VARCHAR(200),
        CurrentValue DECIMAL(18,2),
        ThresholdValue DECIMAL(18,2),
        AlertTime DATETIME2
    );
    
    -- Alert 1: Fleet idling above threshold
    INSERT INTO @AlertTable
    SELECT
        'FleetIdling',
        VehicleID,
        (SUM(IdleTimeMinutes) / NULLIF(SUM(TripDurationMinutes), 0)) * 100,
        15.0,  -- 15% threshold
        GETDATE()
    FROM FactFleetTrips
    WHERE TripStartTime >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY VehicleID
    HAVING (SUM(IdleTimeMinutes) / NULLIF(SUM(TripDurationMinutes), 0)) * 100 > 15;
    
    -- Alert 2: Warehouse dwell time exceeding standards
    INSERT INTO @AlertTable
    SELECT
        'WarehouseDwell',
        CONCAT(WarehouseID, '-', SKU),
        AVG(DwellTimeHours),
        72.0,  -- 72 hours threshold
        GETDATE()
    FROM FactWarehouseOperations
    WHERE OperationTime >= DATEADD(DAY, -1, GETDATE())
        AND OperationType = 'Storage'
    GROUP BY WarehouseID, SKU
    HAVING AVG(DwellTimeHours) > 72;
    
    -- Send alerts (integrate with email/SMS/Teams)
    IF EXISTS (SELECT 1 FROM @AlertTable)
    BEGIN
        INSERT INTO AlertLog (AlertType, Entity, CurrentValue, ThresholdValue, AlertTime, Status)
        SELECT AlertType, Entity, CurrentValue, ThresholdValue, AlertTime, 'Pending'
        FROM @AlertTable;
        
        -- Call external notification service
        -- EXEC msdb.dbo.sp_send_dbmail ...
    END;
END;
GO
```

## Power BI Integration Patterns

### DAX Measures for Cross-Fact KPIs

```dax
// Composite measure: Dwell Time per Unit of Fleet Idle Time
DwellToIdleRatio =
VAR AvgDwell =
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]),
        USERELATIONSHIP(DimTime[TimeKey], FactWarehouseOperations[TimeKey])
    )
VAR AvgIdle =
    CALCULATE(
        AVERAGE(FactFleetTrips[IdleTimeMinutes]),
        USERELATIONSHIP(DimTime[TimeKey], FactFleetTrips[TimeKey])
    )
RETURN
    DIVIDE(AvgDwell * 60, AvgIdle, 0)

// Time intelligence: Comparative period analysis
DwellTime_PriorQuarter =
CALCULATE(
    [AvgDwellTime],
    DATEADD(DimTime[Date], -1, QUARTER)
)

// Dynamic filtering by gravity score
HighGravityItems =
CALCULATE(
    COUNTROWS(DimProductGravity),
    DimProductGravity[GravityScore] >= 75
)
```

### Row-Level Security

```dax
// RLS rule: Users see only their assigned warehouses
[WarehouseRegion] = USERPRINCIPALNAME()

// Or via security table lookup
VAR UserEmail = USERPRINCIPALNAME()
VAR UserWarehouses =
    SELECTCOLUMNS(
        FILTER(UserSecurityTable, UserSecurityTable[Email] = UserEmail),
        "Warehouse", UserSecurityTable[WarehouseID]
    )
RETURN
    [WarehouseID] IN UserWarehouses
```

## Configuration & Customization

### Indexing Strategy

```sql
-- Recommended indexes for query performance

-- Fact table covering indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeSKU
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (DwellTimeHours, UnitsHandled, OperationType);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeRoute
ON FactFleetTrips (TimeKey, RouteID)
INCLUDE (IdleTimeMinutes, FuelConsumptionGallons, TripDurationMinutes);

-- Dimension filtered indexes
CREATE NONCLUSTERED INDEX IX_DimProduct_HighGravity
ON DimProductGravity (GravityScore, ProductCategory)
WHERE GravityScore >= 75;

-- Columnstore for large aggregations
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey, DwellTimeHours, UnitsHandled);
```

### Partitioning for Scale

```sql
-- Partition fact tables by time range (monthly)
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601,
    20260701, 20260801, 20260901, 20261001, 20261101, 20261201
);

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly
ALL TO ([PRIMARY]);

-- Recreate fact table on partition scheme
ALTER TABLE FactWarehouseOperations
DROP CONSTRAINT PK_FactWarehouse;

ALTER TABLE FactWarehouseOperations
ADD CONSTRAINT PK_FactWarehouse PRIMARY KEY (OperationID, TimeKey)
ON PS_Monthly(TimeKey);
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution:** Ensure time dimension is properly indexed and fact tables have foreign key relationships defined:

```sql
-- Verify relationships
SELECT
    fk.name AS ForeignKey,
    OBJECT_NAME(fk.parent_object_id) AS TableName,
    COL_NAME(fkc.parent_object_id, fkc.parent_column_id) AS ColumnName
FROM sys.foreign_keys fk
    INNER JOIN sys.foreign_key_columns fkc ON fk.object_id = fkc.constraint_object_id
WHERE OBJECT_NAME(fk.parent_object_id) LIKE 'Fact%';

-- Add missing indexes
CREATE STATISTICS STAT_FactWarehouse_TimeKey
ON FactWarehouseOperations (TimeKey)
WITH FULLSCAN;
```

### Issue: Power BI Refresh Failures

**Solution:** Check gateway connectivity and query folding:

```sql
-- Test connection from gateway machine
EXEC sp_WhoIsActive;  -- Verify no blocking queries

-- Optimize view for query folding
CREATE VIEW vw_WarehouseMetrics
WITH SCHEMABINDING
AS
SELECT
    TimeKey,
    WarehouseKey,
    SUM(DwellTimeHours) AS TotalDwellTime,
    COUNT_BIG(*) AS OperationCount
FROM dbo.FactWarehouseOperations
GROUP BY TimeKey, WarehouseKey;

-- Create indexed view
CREATE UNIQUE CLUSTERED INDEX IX_vwWarehouse
ON vw_WarehouseMetrics (TimeKey, WarehouseKey);
```

### Issue: Incorrect Gravity Score Calculations

**Solution:** Verify dimension data completeness and calculation logic:

```sql
-- Audit missing or invalid data
SELECT
    'Missing PickFrequency' AS Issue,
    COUNT(*) AS AffectedRows
FROM DimProductGravity
WHERE PickFrequencyRank IS NULL
UNION ALL
SELECT
    'Invalid GravityScore',
    COUNT(*)
FROM DimProductGravity
WHERE GravityScore < 0 OR GravityScore > 100;

-- Recalculate with validation
UPDATE DimProductGravity
SET GravityScore = CASE
    WHEN PickFrequencyRank IS NULL THEN 0
    WHEN UnitValueRank IS NULL THEN 0
    ELSE (PickFrequencyRank * 0.5) + (UnitValueRank * 0.3) + (FragilityScore * 0.2)
END
WHERE LastCalculatedDate < DATEADD(DAY, -7, GETDATE());
```

## Best Practices

1. **Incremental Loading:** Use watermark columns (`LastModifiedDate`) to load only changed records
2. **Time Granularity:** Match dimension granularity (15-min) to business requirements; coarser = faster queries
3. **Composite Keys:** Always include `TimeKey` in fact table primary keys for partition alignment
4. **Security:** Implement row-level security in both SQL (views) and Power BI (DAX)
5. **Documentation:** Annotate custom measures and calculated columns with business logic comments
6. **Testing:** Validate cross-fact joins with small date ranges before production queries
7. **Monitoring:** Set up SQL Server Query Store to track performance regressions

## Additional Resources

- SQL scripts: `schema_deploy.sql`, `stored_procedures.sql`, `sample_queries.sql`
- Power BI template: `LogiFleet_Pulse_Master.pbit`
- Sample data generators: `data_generation/` folder for testing
- Security configuration: `security_roles.sql` for role-based access setup
