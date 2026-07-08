---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "implement multi-fact star schema for warehousing"
  - "build fleet telemetry data warehouse"
  - "create supply chain KPI dashboard"
  - "deploy logistics intelligence platform"
  - "integrate warehouse and fleet analytics"
  - "set up cross-modal supply chain reporting"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:
- **MS SQL Server data warehouse** with multi-fact star schema for warehouse operations and fleet management
- **Power BI dashboards** for real-time logistics visualization
- **Cross-fact KPI harmonization** linking inventory, warehouse operations, and fleet telemetry
- **Predictive analytics** for bottleneck detection and fleet optimization
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and item value

The platform integrates data from WMS, telematics, GPS feeds, supplier portals, and external APIs into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (supports external tables and polybase)
- Power BI Desktop (latest version)
- Database admin access for schema deployment

### Step 1: Deploy SQL Schema

```sql
-- Execute the main schema deployment script
-- This creates all fact tables, dimension tables, views, and stored procedures

-- Core fact tables created:
-- - FactWarehouseOperations
-- - FactFleetTrips
-- - FactCrossDock

-- Dimension tables:
-- - DimTime (15-minute granularity)
-- - DimGeography (hierarchical)
-- - DimProductGravity
-- - DimSupplierReliability

-- Example: Create FactWarehouseOperations
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    QuantityHandled INT,
    CycleTimeMinutes DECIMAL(10,2),
    GravityZoneID INT,
    OperatorID INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WH_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWH_GravityZone ON FactWarehouseOperations(GravityZoneID);
```

### Step 2: Configure Data Sources

Create a configuration file (not included in repo):

```json
{
  "connections": {
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "username": "${SQL_USER}",
      "password": "${SQL_PASSWORD}",
      "trusted_connection": false
    },
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "telematics_feed": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.weatherapi.com/v1",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_ops": 15,
    "fleet_telemetry": 5,
    "supplier_data": 60
  }
}
```

### Step 3: Set Up External Data Ingestion

```sql
-- Create external data source for streaming WMS data
CREATE EXTERNAL DATA SOURCE WMSFeed
WITH (
    TYPE = RDBMS,
    LOCATION = '${WMS_SERVER}',
    DATABASE_NAME = 'WarehouseManagement',
    CREDENTIAL = WMSCredential
);

-- Create external table for incremental loading
CREATE EXTERNAL TABLE ext_WarehouseOperations (
    OperationTimestamp DATETIME2,
    OperationType VARCHAR(50),
    SKU VARCHAR(100),
    Quantity INT,
    WarehouseZone VARCHAR(50),
    DwellTimeMinutes INT
)
WITH (
    DATA_SOURCE = WMSFeed,
    SCHEMA_NAME = 'dbo',
    OBJECT_NAME = 'LiveOperations'
);

-- Incremental load stored procedure
CREATE PROCEDURE sp_LoadWarehouseOperations
AS
BEGIN
    DECLARE @LastLoadTime DATETIME2;
    SELECT @LastLoadTime = MAX(LoadTimestamp) FROM FactWarehouseOperations;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        DwellTimeMinutes, QuantityHandled, CycleTimeMinutes
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        ext.OperationType,
        ext.DwellTimeMinutes,
        ext.Quantity,
        DATEDIFF(MINUTE, ext.OperationTimestamp, GETDATE())
    FROM ext_WarehouseOperations ext
    INNER JOIN DimTime t ON CAST(ext.OperationTimestamp AS DATE) = t.CalendarDate
        AND DATEPART(HOUR, ext.OperationTimestamp) = t.HourOfDay
        AND DATEPART(MINUTE, ext.OperationTimestamp) / 15 = t.QuarterHour
    INNER JOIN DimProduct p ON ext.SKU = p.SKU
    INNER JOIN DimWarehouse w ON ext.WarehouseZone = w.ZoneName
    WHERE ext.OperationTimestamp > @LastLoadTime;
END;
```

## Power BI Integration

### Step 1: Import Template

1. Download `LogiFleet_Pulse_Master.pbit` from the repository
2. Open in Power BI Desktop
3. Enter SQL Server connection details when prompted

### Step 2: Configure Data Model

The Power BI template includes pre-built relationships:

```dax
// Measure: Average Dwell Time by Gravity Zone
AvgDwellTimeByGravity = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[GravityScore])
)

// Measure: Fleet Efficiency Score
FleetEfficiencyScore = 
DIVIDE(
    SUM(FactFleetTrips[DistanceMiles]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + SUM(FactFleetTrips[DrivingTimeMinutes]),
    0
) * 100

// Measure: Cross-Fact KPI - Cost per Unit Shipped
CostPerUnitShipped = 
DIVIDE(
    SUM(FactFleetTrips[FuelCost]) + SUM(FactWarehouseOperations[LaborCost]),
    SUM(FactWarehouseOperations[QuantityHandled]),
    0
)
```

### Step 3: Configure Row-Level Security

```dax
// Create role: Regional Manager
// Filter on DimGeography[Region]
[Region] = USERNAME()

// Create role: Warehouse Supervisor
// Filter on DimWarehouse[WarehouseID]
[WarehouseID] IN 
    (LOOKUPVALUE(
        UserWarehouseAccess[WarehouseID],
        UserWarehouseAccess[Username],
        USERNAME()
    ))
```

## Key SQL Patterns

### Pattern 1: Warehouse Gravity Zone Calculation

```sql
-- Calculate and assign gravity scores to products
WITH ProductMetrics AS (
    SELECT 
        ProductKey,
        AVG(CAST(PickFrequency AS FLOAT)) as AvgPickFreq,
        AVG(UnitValue) as AvgValue,
        MAX(FragilityScore) as MaxFragility
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    WHERE TimeKey >= DATEADD(DAY, -90, GETDATE())
    GROUP BY ProductKey
)
UPDATE DimProductGravity
SET GravityScore = 
    (pm.AvgPickFreq * 0.5) +  -- 50% weight on velocity
    (pm.AvgValue * 0.3) +      -- 30% weight on value
    (pm.MaxFragility * 0.2)    -- 20% weight on fragility
FROM ProductMetrics pm
WHERE DimProductGravity.ProductKey = pm.ProductKey;

-- Recommend zone reassignments
SELECT 
    p.SKU,
    p.ProductName,
    pg.CurrentZone,
    CASE 
        WHEN pg.GravityScore > 0.7 THEN 'Zone A - High Gravity'
        WHEN pg.GravityScore > 0.4 THEN 'Zone B - Medium Gravity'
        ELSE 'Zone C - Low Gravity'
    END AS RecommendedZone,
    pg.GravityScore
FROM DimProduct p
INNER JOIN DimProductGravity pg ON p.ProductKey = pg.ProductKey
WHERE pg.CurrentZone <> 
    CASE 
        WHEN pg.GravityScore > 0.7 THEN 'Zone A - High Gravity'
        WHEN pg.GravityScore > 0.4 THEN 'Zone B - Medium Gravity'
        ELSE 'Zone C - Low Gravity'
    END;
```

### Pattern 2: Fleet Triage Scoring

```sql
-- Proactive maintenance queue with revenue impact
WITH FleetDiagnostics AS (
    SELECT 
        VehicleID,
        TirePressurePSI,
        EngineTemperature,
        LastMaintenanceDate,
        DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) as DaysSinceMaintenance
    FROM DimVehicle
),
ActiveLoads AS (
    SELECT 
        ft.VehicleID,
        SUM(p.UnitValue * wh.QuantityHandled) as LoadValue,
        MAX(p.IsPerishable) as HasPerishables
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wh ON ft.ShipmentID = wh.ShipmentID
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    WHERE ft.TripStatus = 'In Transit'
    GROUP BY ft.VehicleID
)
SELECT 
    fd.VehicleID,
    CASE 
        WHEN fd.TirePressurePSI < 30 THEN 50  -- Critical tire pressure
        WHEN fd.EngineTemperature > 220 THEN 40  -- Overheating
        WHEN fd.DaysSinceMaintenance > 90 THEN 20  -- Overdue maintenance
        ELSE 0
    END AS RiskScore,
    al.LoadValue,
    al.HasPerishables,
    (CASE 
        WHEN fd.TirePressurePSI < 30 THEN 50
        WHEN fd.EngineTemperature > 220 THEN 40
        WHEN fd.DaysSinceMaintenance > 90 THEN 20
        ELSE 0
    END * (al.LoadValue / 1000.0) * (1 + al.HasPerishables)) AS PriorityScore
FROM FleetDiagnostics fd
LEFT JOIN ActiveLoads al ON fd.VehicleID = al.VehicleID
WHERE fd.TirePressurePSI < 35 
   OR fd.EngineTemperature > 210
   OR fd.DaysSinceMaintenance > 60
ORDER BY PriorityScore DESC;
```

### Pattern 3: Cross-Fact Analysis

```sql
-- Correlate dwell time with fleet performance
SELECT 
    dt.CalendarDate,
    dt.DayOfWeek,
    AVG(wh.DwellTimeMinutes) as AvgWarehouseDwellTime,
    AVG(ft.IdleTimeMinutes) as AvgFleetIdleTime,
    SUM(ft.FuelCost) as TotalFuelCost,
    CORR(wh.DwellTimeMinutes, ft.IdleTimeMinutes) 
        OVER (ORDER BY dt.CalendarDate ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) 
        as RollingCorrelation
FROM DimTime dt
LEFT JOIN FactWarehouseOperations wh ON dt.TimeKey = wh.TimeKey
LEFT JOIN FactFleetTrips ft ON dt.TimeKey = ft.DepartureTimeKey
WHERE dt.CalendarDate >= DATEADD(MONTH, -3, GETDATE())
GROUP BY dt.CalendarDate, dt.DayOfWeek
ORDER BY dt.CalendarDate;
```

## Automated Alerting

```sql
-- Stored procedure for KPI threshold monitoring
CREATE PROCEDURE sp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertTable TABLE (
        AlertType VARCHAR(100),
        Severity VARCHAR(20),
        Details VARCHAR(500),
        AffectedEntity VARCHAR(100)
    );
    
    -- Check fleet idling threshold (>15%)
    INSERT INTO @AlertTable
    SELECT 
        'Fleet Idle Time Exceeded',
        'High',
        'Vehicle ' + CAST(VehicleID AS VARCHAR) + ' idle time: ' + 
        CAST(IdlePercentage AS VARCHAR) + '%',
        CAST(VehicleID AS VARCHAR)
    FROM (
        SELECT 
            VehicleID,
            (SUM(IdleTimeMinutes) * 100.0 / NULLIF(SUM(IdleTimeMinutes + DrivingTimeMinutes), 0)) as IdlePercentage
        FROM FactFleetTrips
        WHERE DepartureTimeKey >= DATEADD(DAY, -1, GETDATE())
        GROUP BY VehicleID
    ) idle
    WHERE IdlePercentage > 15;
    
    -- Check warehouse dwell time (>72 hours in cold storage)
    INSERT INTO @AlertTable
    SELECT 
        'Cold Storage Dwell Time Critical',
        'Critical',
        'SKU ' + p.SKU + ' in cold storage for ' + 
        CAST(wh.DwellTimeMinutes / 60 AS VARCHAR) + ' hours',
        p.SKU
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    INNER JOIN DimWarehouse w ON wh.WarehouseKey = w.WarehouseKey
    WHERE w.ZoneType = 'Cold Storage'
      AND wh.DwellTimeMinutes > (72 * 60)
      AND wh.OperationTimestamp >= DATEADD(HOUR, -24, GETDATE());
    
    -- Send alerts (integrate with email/Teams)
    SELECT * FROM @AlertTable;
    
    -- Log to alert history
    INSERT INTO AlertHistory (AlertTimestamp, AlertType, Severity, Details, AffectedEntity)
    SELECT GETDATE(), AlertType, Severity, Details, AffectedEntity
    FROM @AlertTable;
END;

-- Schedule with SQL Agent job (every 15 minutes)
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_KPI_Monitor';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Monitor',
    @step_name = 'Run Threshold Check',
    @command = 'EXEC sp_MonitorKPIThresholds';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Fact table partitioning by month
CREATE PARTITION FUNCTION PF_MonthlyPartition (INT)
AS RANGE RIGHT FOR VALUES (20260101, 20260201, 20260301, 20260401);

CREATE PARTITION SCHEME PS_MonthlyPartition
AS PARTITION PF_MonthlyPartition ALL TO ([PRIMARY]);

-- Apply to fact table
CREATE TABLE FactWarehouseOperations (
    -- columns...
    OperationTimestamp DATETIME2,
    TimeKey INT
) ON PS_MonthlyPartition(TimeKey);

-- Columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWH_Analytics
ON FactWarehouseOperations (
    TimeKey, ProductKey, WarehouseKey, 
    DwellTimeMinutes, QuantityHandled
);
```

### Data Retention Policy

```sql
-- Archive old data to separate table
CREATE PROCEDURE sp_ArchiveOldData
    @RetentionMonths INT = 24
AS
BEGIN
    DECLARE @ArchiveDate DATE = DATEADD(MONTH, -@RetentionMonths, GETDATE());
    
    -- Move to archive table
    INSERT INTO FactWarehouseOperations_Archive
    SELECT * FROM FactWarehouseOperations
    WHERE OperationTimestamp < @ArchiveDate;
    
    -- Delete from main table
    DELETE FROM FactWarehouseOperations
    WHERE OperationTimestamp < @ArchiveDate;
END;
```

## Troubleshooting

### Issue: Power BI Refresh Fails

**Symptoms**: Dashboard shows "unable to refresh data model"

**Solution**:
```sql
-- Check external table connectivity
SELECT TOP 10 * FROM ext_WarehouseOperations;

-- Verify credentials
SELECT * FROM sys.credentials WHERE name = 'WMSCredential';

-- Test polybase service
SELECT * FROM sys.dm_exec_external_work;
```

### Issue: Slow Cross-Fact Queries

**Symptoms**: Dashboards take >30 seconds to load

**Solution**:
```sql
-- Add covering indexes
CREATE NONCLUSTERED INDEX IX_FactWH_Composite
ON FactWarehouseOperations (TimeKey, ProductKey, WarehouseKey)
INCLUDE (DwellTimeMinutes, QuantityHandled);

CREATE NONCLUSTERED INDEX IX_FactFleet_Composite
ON FactFleetTrips (DepartureTimeKey, VehicleID)
INCLUDE (IdleTimeMinutes, FuelCost);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Incorrect Gravity Zone Assignments

**Symptoms**: Products in wrong zones despite high gravity scores

**Solution**:
```sql
-- Recalculate gravity scores with proper weights
UPDATE DimProductGravity
SET GravityScore = (
    SELECT 
        (COUNT(*) * 0.5) / 100.0 +  -- Normalized pick frequency
        (AVG(p.UnitValue) * 0.3) / 1000.0 +  -- Normalized value
        (MAX(p.FragilityScore) * 0.2)
    FROM FactWarehouseOperations wh
    INNER JOIN DimProduct p ON wh.ProductKey = p.ProductKey
    WHERE wh.ProductKey = DimProductGravity.ProductKey
      AND wh.OperationType = 'Picking'
      AND wh.TimeKey >= DATEADD(DAY, -90, GETDATE())
);

-- Force zone reassignment
EXEC sp_ReassignWarehouseZones;
```

### Issue: Missing Time Dimension Records

**Symptoms**: Null values in dashboards for recent dates

**Solution**:
```sql
-- Populate time dimension for next 2 years
DECLARE @StartDate DATE = GETDATE();
DECLARE @EndDate DATE = DATEADD(YEAR, 2, GETDATE());

WITH TimeSeries AS (
    SELECT @StartDate AS CalendarDate
    UNION ALL
    SELECT DATEADD(DAY, 1, CalendarDate)
    FROM TimeSeries
    WHERE CalendarDate < @EndDate
)
INSERT INTO DimTime (TimeKey, CalendarDate, HourOfDay, QuarterHour, DayOfWeek, FiscalPeriod)
SELECT 
    CAST(FORMAT(t.CalendarDate, 'yyyyMMdd') AS INT) * 100 + h.HourOfDay,
    t.CalendarDate,
    h.HourOfDay,
    q.QuarterHour,
    DATEPART(WEEKDAY, t.CalendarDate),
    'FY' + CAST(YEAR(DATEADD(MONTH, 6, t.CalendarDate)) AS VARCHAR)
FROM TimeSeries t
CROSS JOIN (SELECT 0 AS HourOfDay UNION ALL SELECT 1 UNION ALL SELECT 2 -- ... up to 23
) h
CROSS JOIN (SELECT 0 AS QuarterHour UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) q
OPTION (MAXRECURSION 1000);
```

## Common Patterns

### Pattern: Natural Language Query Support

Enable Power BI Q&A with synonyms:

```
// In Power BI model, add synonyms to tables/columns
DimProduct.SKU → "product code", "item number", "stock keeping unit"
FactWarehouseOperations.DwellTimeMinutes → "storage time", "wait time", "holding period"
FactFleetTrips.IdleTimeMinutes → "downtime", "waiting time", "non-driving time"
```

### Pattern: Temporal Elasticity Simulation

```sql
-- Simulate capacity increase impact
WITH BaselineMetrics AS (
    SELECT 
        AVG(DwellTimeMinutes) as AvgDwell,
        AVG(IdleTimeMinutes) as AvgIdle,
        SUM(QuantityHandled) as TotalVolume
    FROM FactWarehouseOperations wh
    INNER JOIN FactFleetTrips ft ON wh.ShipmentID = ft.ShipmentID
    WHERE wh.TimeKey >= DATEADD(MONTH, -3, GETDATE())
)
SELECT 
    'Current (80% capacity)' as Scenario,
    AvgDwell, AvgIdle, TotalVolume
FROM BaselineMetrics
UNION ALL
SELECT 
    '95% capacity projection',
    AvgDwell * 1.35,  -- Historical correlation factor
    AvgIdle * 1.22,
    TotalVolume * 1.1875  -- 95/80 ratio
FROM BaselineMetrics;
```
