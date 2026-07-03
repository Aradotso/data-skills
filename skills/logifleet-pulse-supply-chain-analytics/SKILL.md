---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics analytics dashboard"
  - "create supply chain data warehouse"
  - "build fleet management analytics with Power BI"
  - "implement warehouse operations reporting"
  - "design multi-fact star schema for logistics"
  - "configure SQL Server for supply chain data"
  - "integrate telematics data with warehouse metrics"
  - "deploy LogiFleet Pulse analytics platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and supply chain data
- **Power BI dashboards** for real-time logistics visibility
- **MS SQL Server database schema** with time-phased dimensions
- **Cross-fact KPI harmonization** (e.g., inventory dwell time vs. fleet fuel costs)
- **Predictive analytics** for bottleneck detection and maintenance prioritization

The platform connects warehouse management data, vehicle telematics, supplier information, and external factors (weather, traffic) into a unified semantic layer for decision-making.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version recommended)
- Access to data sources: WMS, telematics/GPS, ERP, or sample data files

### Step 1: Deploy SQL Schema

1. Clone or download the repository:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. Connect to your SQL Server instance and create a new database:
```sql
CREATE DATABASE LogiFleetPulse;
GO
USE LogiFleetPulse;
GO
```

3. Execute the schema creation scripts (run in order):
```sql
-- Create dimension tables
:r dim_time.sql
:r dim_geography.sql
:r dim_product_gravity.sql
:r dim_supplier_reliability.sql

-- Create fact tables
:r fact_warehouse_operations.sql
:r fact_fleet_trips.sql
:r fact_cross_dock.sql

-- Create bridge tables for many-to-many relationships
:r bridge_route_storage_zone.sql

-- Create views and stored procedures
:r views_kpi_aggregations.sql
:r sp_incremental_load.sql
:r sp_alert_monitoring.sql
```

### Step 2: Configure Data Sources

Create a configuration file for connection strings (do not commit to version control):

```json
{
  "data_sources": {
    "wms_connection": "Server=${WMS_SQL_SERVER};Database=${WMS_DB};User Id=${WMS_USER};Password=${WMS_PASSWORD};",
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_URL}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "erp_connection": "Server=${ERP_SERVER};Database=${ERP_DB};Integrated Security=true;",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "refresh_intervals": {
    "warehouse_operations": 15,
    "fleet_trips": 15,
    "supplier_data": 1440
  }
}
```

### Step 3: Load Power BI Template

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection details when prompted
3. The template will automatically detect fact tables and build relationships
4. Configure row-level security in the security table if needed

## Key Database Schema Components

### Core Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:
```sql
SELECT 
    TimeKey,
    FullDateTime,
    HourOfDay,
    DayOfWeek,
    FiscalPeriod,
    IsBusinessHour,
    ShiftNumber
FROM DimTime
WHERE FullDateTime >= DATEADD(HOUR, -24, GETDATE());
```

**DimGeography** - Hierarchical location dimension:
```sql
SELECT 
    GeographyKey,
    ContinentName,
    CountryName,
    RegionName,
    WarehouseCode,
    Latitude,
    Longitude
FROM DimGeography
WHERE IsActive = 1;
```

**DimProductGravity** - Product classification with velocity scoring:
```sql
SELECT 
    ProductKey,
    SKU,
    ProductName,
    GravityScore,  -- Composite: velocity + value + fragility
    GravityZone,   -- HIGH, MEDIUM, LOW
    OptimalStorageLocation
FROM DimProductGravity
ORDER BY GravityScore DESC;
```

### Core Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics:
```sql
-- Query average dwell time by product gravity zone
SELECT 
    pg.GravityZone,
    AVG(DATEDIFF(HOUR, wo.ReceiveTime, wo.ShipTime)) AS AvgDwellHours,
    COUNT(*) AS TotalTransactions,
    SUM(wo.UnitsHandled) AS TotalUnits
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity pg ON wo.ProductKey = pg.ProductKey
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY pg.GravityZone
ORDER BY AvgDwellHours DESC;
```

**FactFleetTrips** - Vehicle telemetry and route data:
```sql
-- Identify routes with excessive idle time
SELECT 
    ft.RouteID,
    g.RegionName,
    SUM(ft.TripDurationMinutes) AS TotalTripMinutes,
    SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
    CAST(SUM(ft.IdleTimeMinutes) AS FLOAT) / SUM(ft.TripDurationMinutes) * 100 AS IdlePercentage,
    AVG(ft.FuelConsumptionLiters) AS AvgFuelConsumption
FROM FactFleetTrips ft
INNER JOIN DimGeography g ON ft.DestinationGeographyKey = g.GeographyKey
INNER JOIN DimTime t ON ft.DepartureTimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(DAY, -7, GETDATE())
GROUP BY ft.RouteID, g.RegionName
HAVING CAST(SUM(ft.IdleTimeMinutes) AS FLOAT) / SUM(ft.TripDurationMinutes) > 0.15
ORDER BY IdlePercentage DESC;
```

## Cross-Fact KPI Queries

### Correlate Warehouse Dwell Time with Fleet Delays

```sql
-- Find products with high warehouse dwell time that also experience fleet delays
WITH WarehouseDwell AS (
    SELECT 
        wo.ProductKey,
        AVG(DATEDIFF(HOUR, wo.ReceiveTime, wo.ShipTime)) AS AvgDwellHours
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY wo.ProductKey
    HAVING AVG(DATEDIFF(HOUR, wo.ReceiveTime, wo.ShipTime)) > 48
),
FleetDelays AS (
    SELECT 
        ft.ProductKey,
        AVG(DATEDIFF(MINUTE, ft.ScheduledArrival, ft.ActualArrival)) AS AvgDelayMinutes
    FROM FactFleetTrips ft
    INNER JOIN DimTime t ON ft.DepartureTimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
        AND ft.ActualArrival > ft.ScheduledArrival
    GROUP BY ft.ProductKey
)
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone,
    wd.AvgDwellHours,
    fd.AvgDelayMinutes
FROM WarehouseDwell wd
INNER JOIN FleetDelays fd ON wd.ProductKey = fd.ProductKey
INNER JOIN DimProductGravity p ON wd.ProductKey = p.ProductKey
ORDER BY wd.AvgDwellHours DESC, fd.AvgDelayMinutes DESC;
```

### Calculate Total Logistics Cost per SKU

```sql
-- Combine warehouse handling costs with fleet delivery costs
SELECT 
    p.SKU,
    p.ProductName,
    SUM(wo.HandlingCost) AS TotalWarehouseCost,
    SUM(ft.DeliveryCost) AS TotalFleetCost,
    SUM(wo.HandlingCost) + SUM(ft.DeliveryCost) AS TotalLogisticsCost,
    SUM(wo.UnitsHandled) AS TotalUnits,
    (SUM(wo.HandlingCost) + SUM(ft.DeliveryCost)) / NULLIF(SUM(wo.UnitsHandled), 0) AS CostPerUnit
FROM DimProductGravity p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
LEFT JOIN FactFleetTrips ft ON p.ProductKey = ft.ProductKey
WHERE wo.TimeKey IN (
    SELECT TimeKey FROM DimTime 
    WHERE FullDateTime >= DATEADD(MONTH, -3, GETDATE())
)
GROUP BY p.SKU, p.ProductName
ORDER BY TotalLogisticsCost DESC;
```

## Incremental Data Loading

Use the provided stored procedure for efficient data updates:

```sql
-- Incremental load for warehouse operations
EXEC sp_IncrementalLoadWarehouse
    @StartDateTime = '2026-07-01 00:00:00',
    @EndDateTime = '2026-07-02 23:59:59',
    @SourceConnection = '${WMS_CONNECTION_STRING}';

-- Incremental load for fleet trips
EXEC sp_IncrementalLoadFleet
    @StartDateTime = '2026-07-01 00:00:00',
    @EndDateTime = '2026-07-02 23:59:59',
    @TelematicsAPIKey = '${TELEMATICS_API_KEY}';
```

Schedule these using SQL Server Agent jobs for automated refresh:

```sql
-- Create SQL Server Agent job for 15-minute refresh
USE msdb;
GO

EXEC sp_add_job 
    @job_name = N'LogiFleet_IncrementalRefresh',
    @enabled = 1;

EXEC sp_add_jobstep 
    @job_name = N'LogiFleet_IncrementalRefresh',
    @step_name = N'Load Warehouse Data',
    @subsystem = N'TSQL',
    @command = N'EXEC LogiFleetPulse.dbo.sp_IncrementalLoadWarehouse 
                  @StartDateTime = DATEADD(MINUTE, -15, GETDATE()),
                  @EndDateTime = GETDATE();';

EXEC sp_add_schedule 
    @schedule_name = N'Every15Minutes',
    @freq_type = 4,  -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4,  -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule 
    @job_name = N'LogiFleet_IncrementalRefresh',
    @schedule_name = N'Every15Minutes';

EXEC sp_add_jobserver 
    @job_name = N'LogiFleet_IncrementalRefresh';
```

## Automated Alerting

Configure threshold-based alerts:

```sql
-- Alert when fleet idle time exceeds 15%
EXEC sp_ConfigureAlert
    @AlertName = 'HighFleetIdleTime',
    @MetricType = 'FleetIdlePercentage',
    @Threshold = 15.0,
    @ComparisonOperator = '>',
    @NotificationEmails = '${OPERATIONS_EMAIL}',
    @CheckIntervalMinutes = 30;

-- Alert when warehouse dwell time exceeds 72 hours for high-gravity products
EXEC sp_ConfigureAlert
    @AlertName = 'HighGravityProductDelay',
    @MetricType = 'WarehouseDwellHours',
    @Threshold = 72.0,
    @ComparisonOperator = '>',
    @FilterCondition = 'GravityZone = ''HIGH''',
    @NotificationEmails = '${WAREHOUSE_MANAGER_EMAIL}',
    @CheckIntervalMinutes = 60;
```

## Power BI Dashboard Configuration

### Key Measures (DAX)

**Average Dwell Time**:
```dax
AvgDwellTime = 
CALCULATE(
    AVERAGE(
        DATEDIFF(
            FactWarehouseOperations[ReceiveTime],
            FactWarehouseOperations[ShipTime],
            HOUR
        )
    ),
    FactWarehouseOperations[ShipTime] <> BLANK()
)
```

**Fleet Utilization Rate**:
```dax
FleetUtilization = 
VAR TotalTripTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR TotalIdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
DIVIDE(
    TotalTripTime - TotalIdleTime,
    TotalTripTime,
    0
)
```

**Cross-Fact Efficiency Score**:
```dax
LogisticsEfficiencyScore = 
VAR WarehouseThroughput = 
    DIVIDE(
        SUM(FactWarehouseOperations[UnitsHandled]),
        DISTINCTCOUNT(FactWarehouseOperations[OperationID])
    )
VAR FleetOnTimeRate = 
    DIVIDE(
        COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[ActualArrival] <= FactFleetTrips[ScheduledArrival])),
        COUNTROWS(FactFleetTrips)
    )
RETURN
(WarehouseThroughput * 0.5) + (FleetOnTimeRate * 100 * 0.5)
```

### Row-Level Security

Configure RLS in Power BI for role-based access:

```dax
-- Regional Manager can only see their region
[RegionName] = USERNAME()

-- Warehouse Supervisor can only see their warehouse
[WarehouseCode] = LOOKUPVALUE(
    SecurityTable[WarehouseCode],
    SecurityTable[UserEmail],
    USERNAME()
)
```

## Common Patterns

### Pattern 1: Identify Gravity Zone Misallocations

```sql
-- Find high-velocity products in low-gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityZone AS CurrentZone,
    p.GravityScore,
    AVG(wo.PickTimeSeconds) AS AvgPickTime,
    COUNT(*) AS PickFrequency
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'PICK'
    AND p.GravityZone = 'LOW'
GROUP BY p.SKU, p.ProductName, p.GravityZone, p.GravityScore
HAVING COUNT(*) > 100  -- High frequency picks
    AND AVG(wo.PickTimeSeconds) > 45  -- Slow pick times
ORDER BY PickFrequency DESC;
```

### Pattern 2: Predictive Maintenance Scoring

```sql
-- Score vehicles for maintenance priority
WITH VehicleHealth AS (
    SELECT 
        VehicleID,
        AVG(CASE WHEN TirePressureWarning = 1 THEN 3 ELSE 0 END) AS TireScore,
        AVG(CASE WHEN EngineWarningLight = 1 THEN 5 ELSE 0 END) AS EngineScore,
        AVG(CASE WHEN MaintenanceOverdue = 1 THEN 4 ELSE 0 END) AS OverdueScore,
        SUM(TripDurationMinutes) / 60.0 AS TotalHoursUsed
    FROM FactFleetTrips
    WHERE DepartureTimeKey IN (
        SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE())
    )
    GROUP BY VehicleID
),
VehicleRevenue AS (
    SELECT 
        VehicleID,
        AVG(CargoValue) AS AvgCargoValue,
        COUNT(DISTINCT ProductKey) AS ProductDiversity
    FROM FactFleetTrips
    WHERE DepartureTimeKey IN (
        SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE())
    )
    GROUP BY VehicleID
)
SELECT 
    vh.VehicleID,
    (vh.TireScore + vh.EngineScore + vh.OverdueScore) AS HealthRiskScore,
    vr.AvgCargoValue,
    ((vh.TireScore + vh.EngineScore + vh.OverdueScore) * vr.AvgCargoValue / 1000.0) AS MaintenancePriorityScore
FROM VehicleHealth vh
INNER JOIN VehicleRevenue vr ON vh.VehicleID = vr.VehicleID
WHERE (vh.TireScore + vh.EngineScore + vh.OverdueScore) > 0
ORDER BY MaintenancePriorityScore DESC;
```

### Pattern 3: Temporal Elasticity Analysis

```sql
-- Simulate impact of warehouse capacity changes on fleet utilization
WITH CapacityScenarios AS (
    SELECT 
        'Current_80pct' AS Scenario,
        0.80 AS CapacityFactor
    UNION ALL
    SELECT 'Increased_95pct', 0.95
),
BaselineMetrics AS (
    SELECT 
        AVG(DATEDIFF(HOUR, wo.ReceiveTime, wo.ShipTime)) AS AvgDwellHours,
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdle
    FROM FactWarehouseOperations wo
    INNER JOIN FactFleetTrips ft ON wo.ProductKey = ft.ProductKey
        AND wo.TimeKey = ft.DepartureTimeKey
    WHERE wo.TimeKey IN (
        SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(DAY, -30, GETDATE())
    )
)
SELECT 
    cs.Scenario,
    bm.AvgDwellHours AS BaselineDwellHours,
    bm.AvgDwellHours * (1 / cs.CapacityFactor) AS ProjectedDwellHours,
    bm.AvgFleetIdle AS BaselineFleetIdle,
    bm.AvgFleetIdle * (1 - (cs.CapacityFactor - 0.80) * 0.5) AS ProjectedFleetIdle
FROM CapacityScenarios cs
CROSS JOIN BaselineMetrics bm;
```

## Troubleshooting

### Issue: Power BI Refresh Failures

**Symptom**: Power BI reports "Unable to connect to data source"

**Solution**:
1. Verify SQL Server allows remote connections
2. Check firewall rules for port 1433
3. Ensure service account has db_datareader role:
```sql
USE LogiFleetPulse;
CREATE USER [DOMAIN\PowerBIService] FOR LOGIN [DOMAIN\PowerBIService];
ALTER ROLE db_datareader ADD MEMBER [DOMAIN\PowerBIService];
```

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining multiple fact tables take >30 seconds

**Solution**: Add missing indexes on foreign keys:
```sql
-- Index fact tables on dimension keys
CREATE NONCLUSTERED INDEX IX_WarehouseOps_ProductTime 
ON FactWarehouseOperations(ProductKey, TimeKey)
INCLUDE (UnitsHandled, HandlingCost);

CREATE NONCLUSTERED INDEX IX_FleetTrips_ProductTime 
ON FactFleetTrips(ProductKey, DepartureTimeKey)
INCLUDE (TripDurationMinutes, DeliveryCost);

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

### Issue: Duplicate Records in Fact Tables

**Symptom**: KPI totals are inflated

**Solution**: Implement deduplication logic in incremental load:
```sql
-- Add to sp_IncrementalLoadWarehouse
MERGE FactWarehouseOperations AS target
USING (
    SELECT DISTINCT  -- Remove source duplicates
        OperationID,
        ProductKey,
        TimeKey,
        UnitsHandled,
        HandlingCost
    FROM staging.WarehouseOperations
    WHERE LoadDate >= @StartDateTime
) AS source
ON target.OperationID = source.OperationID
WHEN NOT MATCHED THEN
    INSERT (OperationID, ProductKey, TimeKey, UnitsHandled, HandlingCost)
    VALUES (source.OperationID, source.ProductKey, source.TimeKey, source.UnitsHandled, source.HandlingCost);
```

### Issue: Time Zone Inconsistencies

**Symptom**: Dashboard shows activity at wrong hours

**Solution**: Standardize all timestamps to UTC:
```sql
-- Convert local times to UTC in dimension table
UPDATE DimTime
SET FullDateTime = DATEADD(
    HOUR,
    DATEDIFF(HOUR, GETDATE(), GETUTCDATE()),
    FullDateTime
)
WHERE TimeZone <> 'UTC';

-- Mark time zone for clarity
ALTER TABLE DimTime ADD TimeZone VARCHAR(10) DEFAULT 'UTC';
```

## Best Practices

1. **Partition Large Fact Tables**: Use date-based partitioning for tables >10M rows
```sql
CREATE PARTITION FUNCTION pfYearMonth (DATETIME)
AS RANGE RIGHT FOR VALUES 
('2026-01-01', '2026-02-01', '2026-03-01', /* ... */);

CREATE PARTITION SCHEME psYearMonth
AS PARTITION pfYearMonth ALL TO ([PRIMARY]);

CREATE TABLE FactWarehouseOperations_Partitioned (
    -- columns
) ON psYearMonth(ReceiveTime);
```

2. **Use Incremental Refresh in Power BI**: Configure for fact tables only
3. **Monitor Query Performance**: Enable Query Store
```sql
ALTER DATABASE LogiFleetPulse SET QUERY_STORE = ON;
```

4. **Backup Regularly**: Schedule automated backups with retention policy
```sql
BACKUP DATABASE LogiFleetPulse
TO DISK = 'D:\Backups\LogiFleetPulse_Full.bak'
WITH FORMAT, COMPRESSION;
```

5. **Document Custom Calculations**: Add extended properties to columns
```sql
EXEC sp_addextendedproperty 
    @name = 'MS_Description',
    @value = 'Composite score: (velocity * 0.4) + (value * 0.3) + (fragility * 0.3)',
    @level0type = 'SCHEMA', @level0name = 'dbo',
    @level1type = 'TABLE', @level1name = 'DimProductGravity',
    @level2type = 'COLUMN', @level2name = 'GravityScore';
```
