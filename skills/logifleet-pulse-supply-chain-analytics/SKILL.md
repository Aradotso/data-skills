---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse template for logistics intelligence, warehouse operations, fleet management, and cross-modal supply chain analytics
triggers:
  - set up LogiFleet Pulse supply chain analytics
  - deploy logistics data warehouse with Power BI
  - configure warehouse and fleet tracking analytics
  - implement multi-fact star schema for logistics
  - build supply chain KPI dashboard
  - create warehouse gravity zone analytics
  - set up fleet telemetry and route optimization
  - implement logistics bottleneck detection
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics operations. It provides a multi-fact star schema combining warehouse operations, fleet telemetry, inventory management, and supply chain KPIs into a unified Power BI visualization layer backed by MS SQL Server.

**Key capabilities:**
- Multi-fact star schema linking warehouse, fleet, and cross-dock operations
- Real-time Power BI dashboards with 15-minute refresh cycles
- Warehouse gravity zone optimization for inventory placement
- Predictive bottleneck detection and fleet triage scoring
- Cross-fact KPI harmonization (e.g., dwell time vs. fleet costs)
- Time-phased dimensional modeling for temporal analytics

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to source data: WMS, TMS, telemetry feeds, or sample data

### Database Schema Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Execute the main schema script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Hour TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    Month TINYINT,
    Quarter TINYINT,
    Year SMALLINT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    TimeSlot15Min TINYINT -- 0-95 for 96 slots per day
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    Continent VARCHAR(50),
    Country VARCHAR(50),
    Region VARCHAR(100),
    LocationCode VARCHAR(20),
    LocationName VARCHAR(200),
    LocationType VARCHAR(20), -- 'Warehouse', 'Route Node', 'Distribution Center'
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6)
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / lead_time
    GravityZone VARCHAR(20) -- 'High', 'Medium', 'Low'
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) NOT NULL,
    SupplierName VARCHAR(200),
    LeadTimeMean DECIMAL(5,1),
    LeadTimeStdDev DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(3,2), -- 0.00 to 1.00
    LastEvaluationDate DATE
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    QuantityUnits INT,
    DwellTimeMinutes INT,
    ProcessingTimeSeconds INT,
    StorageZone VARCHAR(20),
    OperatorID VARCHAR(20),
    BatchNumber VARCHAR(50)
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(20),
    DriverID VARCHAR(20),
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    AvgSpeed DECIMAL(5,2),
    MaintenanceAlert BIT
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    TransferTimeMinutes INT,
    QuantityUnits INT
);

-- Create indexes for performance
CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);

CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
CREATE INDEX IX_FactFleet_Destination ON FactFleetTrips(DestinationGeographyKey);

CREATE INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey);
CREATE INDEX IX_FactCrossDock_Product ON FactCrossDock(ProductKey);
```

### Data Loading Procedures

**Incremental load stored procedure for warehouse operations:**

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from staging table populated by WMS integration
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, QuantityUnits, DwellTimeMinutes,
        ProcessingTimeSeconds, StorageZone, OperatorID, BatchNumber
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.QuantityUnits,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        stg.ProcessingTimeSeconds,
        stg.StorageZone,
        stg.OperatorID,
        stg.BatchNumber
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON CAST(stg.OperationDateTime AS DATE) = CAST(t.DateTime AS DATE)
        AND DATEPART(HOUR, stg.OperationDateTime) = t.Hour
        AND (DATEPART(MINUTE, stg.OperationDateTime) / 15) = t.TimeSlot15Min
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierCode = s.SupplierCode
    WHERE stg.OperationDateTime BETWEEN @StartDateTime AND @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.BatchNumber = stg.BatchNumber
                AND f.OperationType = stg.OperationType
        );
END;
GO
```

**Fleet telemetry integration:**

```sql
CREATE PROCEDURE sp_LoadFleetTrips
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, OriginGeographyKey, DestinationGeographyKey,
        VehicleID, DriverID, DistanceKM, FuelLiters,
        IdleTimeMinutes, LoadingTimeMinutes, UnloadingTimeMinutes,
        LoadWeightKG, DelayMinutes, DelayReason, AvgSpeed, MaintenanceAlert
    )
    SELECT 
        t.TimeKey,
        og.GeographyKey AS OriginGeographyKey,
        dg.GeographyKey AS DestinationGeographyKey,
        stg.VehicleID,
        stg.DriverID,
        stg.DistanceKM,
        stg.FuelLiters,
        stg.IdleTimeMinutes,
        stg.LoadingTimeMinutes,
        stg.UnloadingTimeMinutes,
        stg.LoadWeightKG,
        stg.DelayMinutes,
        stg.DelayReason,
        stg.AvgSpeed,
        CASE 
            WHEN stg.TirePressurePSI < 30 OR stg.EngineTempC > 105 THEN 1
            ELSE 0
        END AS MaintenanceAlert
    FROM StagingFleetTrips stg
    INNER JOIN DimTime t ON CAST(stg.TripStartDateTime AS DATE) = CAST(t.DateTime AS DATE)
        AND DATEPART(HOUR, stg.TripStartDateTime) = t.Hour
    INNER JOIN DimGeography og ON stg.OriginLocationCode = og.LocationCode
    INNER JOIN DimGeography dg ON stg.DestinationLocationCode = dg.LocationCode
    WHERE stg.TripStartDateTime BETWEEN @StartDateTime AND @EndDateTime;
END;
GO
```

### Power BI Configuration

1. **Open the Power BI template:**
```bash
# Open LogiFleet_Pulse_Master.pbit in Power BI Desktop
```

2. **Configure data source connection:**
   - When prompted, enter your SQL Server instance name
   - Enter database name: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use environment variables for credentials)

3. **Connection string format:**
```
Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;User ID=${SQL_USER};Password=${SQL_PASSWORD}
```

## Key Analytics Views

### Warehouse Gravity Zone Analysis

**SQL view for gravity zone performance:**

```sql
CREATE VIEW vw_GravityZonePerformance AS
SELECT 
    p.GravityZone,
    p.Category,
    COUNT(DISTINCT w.OperationKey) AS TotalOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTimeMin,
    AVG(w.ProcessingTimeSeconds) AS AvgProcessingTimeSec,
    SUM(w.QuantityUnits) AS TotalUnitsProcessed,
    AVG(p.GravityScore) AS AvgGravityScore
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.GravityZone, p.Category;
```

### Fleet Efficiency Metrics

**Cross-fact KPI query:**

```sql
CREATE VIEW vw_FleetEfficiencyKPI AS
SELECT 
    f.VehicleID,
    COUNT(f.TripKey) AS TotalTrips,
    SUM(f.DistanceKM) AS TotalDistanceKM,
    SUM(f.FuelLiters) AS TotalFuelLiters,
    SUM(f.IdleTimeMinutes) AS TotalIdleTimeMin,
    SUM(f.DelayMinutes) AS TotalDelayMin,
    (SUM(f.DistanceKM) / NULLIF(SUM(f.FuelLiters), 0)) AS KmPerLiter,
    (SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, 0, 
        DATEADD(MINUTE, f.IdleTimeMinutes + f.LoadingTimeMinutes + f.UnloadingTimeMinutes, 0))), 0)) AS IdleTimePercent,
    SUM(CASE WHEN f.MaintenanceAlert = 1 THEN 1 ELSE 0 END) AS MaintenanceAlertCount
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
GROUP BY f.VehicleID;
```

### Predictive Bottleneck Detection

**Identifying potential bottlenecks:**

```sql
CREATE PROCEDURE sp_DetectBottlenecks
    @ThresholdDwellTimeMin INT = 180,
    @ThresholdIdleTimePercent DECIMAL(5,2) = 15.0
AS
BEGIN
    -- Warehouse bottlenecks
    SELECT 
        'Warehouse' AS BottleneckType,
        g.LocationName,
        p.GravityZone,
        p.Category,
        AVG(w.DwellTimeMinutes) AS AvgDwellTimeMin,
        COUNT(w.OperationKey) AS AffectedOperations
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY g.LocationName, p.GravityZone, p.Category
    HAVING AVG(w.DwellTimeMinutes) > @ThresholdDwellTimeMin
    
    UNION ALL
    
    -- Fleet bottlenecks
    SELECT 
        'Fleet' AS BottleneckType,
        f.VehicleID AS LocationName,
        'N/A' AS GravityZone,
        'Vehicle Utilization' AS Category,
        AVG(f.IdleTimeMinutes) AS AvgDwellTimeMin,
        COUNT(f.TripKey) AS AffectedOperations
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY f.VehicleID
    HAVING (SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, 0, 
            DATEADD(MINUTE, f.IdleTimeMinutes + 
                DATEDIFF(MINUTE, 0, DATEADD(SECOND, (f.DistanceKM / NULLIF(f.AvgSpeed, 0)) * 60, 0)), 0))), 0)) 
        > @ThresholdIdleTimePercent;
END;
GO
```

## Configuration Files

### Data Source Configuration

Create `config.json` (use environment variables for sensitive data):

```json
{
  "database": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "user": "${SQL_USER}",
    "password": "${SQL_PASSWORD}",
    "port": 1433
  },
  "dataRefresh": {
    "intervalMinutes": 15,
    "batchSize": 10000,
    "parallelStreams": 4
  },
  "alerts": {
    "emailEnabled": true,
    "smtpServer": "${SMTP_SERVER}",
    "fromAddress": "${ALERT_EMAIL_FROM}",
    "recipients": [
      "${ALERT_EMAIL_TO}"
    ]
  },
  "externalAPIs": {
    "weatherAPI": {
      "endpoint": "https://api.weather.com/v3/wx/",
      "apiKey": "${WEATHER_API_KEY}"
    },
    "trafficAPI": {
      "endpoint": "https://api.traffic.com/v2/",
      "apiKey": "${TRAFFIC_API_KEY}"
    }
  }
}
```

### Automated Alerting Setup

**SQL Agent job for KPI breach alerts:**

```sql
CREATE PROCEDURE sp_CheckKPIAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for excessive dwell times
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY w.GeographyKey
        HAVING AVG(w.DwellTimeMinutes) > 240
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse dwell time exceeded 4 hours in the last hour. Check vw_GravityZonePerformance for details.';
        
        -- Insert into alert log
        INSERT INTO AlertLog (AlertDateTime, AlertType, AlertMessage, Severity)
        VALUES (GETDATE(), 'Warehouse Dwell Time', @AlertMessage, 'High');
        
        -- Execute email notification (configure Database Mail)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetPulse',
            @recipients = '${ALERT_EMAIL_TO}',
            @subject = 'LogiFleet Pulse - High Dwell Time Alert',
            @body = @AlertMessage;
    END;
    
    -- Check for fleet idle time
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips f
        INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
        WHERE t.DateTime >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY f.VehicleID
        HAVING (SUM(f.IdleTimeMinutes) * 100.0 / NULLIF(SUM(DATEDIFF(MINUTE, 0, 
                DATEADD(MINUTE, f.IdleTimeMinutes + 60, 0))), 0)) > 20
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 20% in the last hour. Review vw_FleetEfficiencyKPI.';
        
        INSERT INTO AlertLog (AlertDateTime, AlertType, AlertMessage, Severity)
        VALUES (GETDATE(), 'Fleet Idle Time', @AlertMessage, 'Medium');
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetPulse',
            @recipients = '${ALERT_EMAIL_TO}',
            @subject = 'LogiFleet Pulse - High Idle Time Alert',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule the alert job to run every 15 minutes
```

## Common Patterns

### Pattern 1: Cross-Fact Analysis

**Correlating warehouse performance with fleet delays:**

```sql
SELECT 
    g.LocationName AS Warehouse,
    AVG(w.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    COUNT(DISTINCT f.TripKey) AS OutboundTrips,
    AVG(f.DelayMinutes) AS AvgFleetDelayMin,
    SUM(f.DelayMinutes) AS TotalFleetDelayMin
FROM FactWarehouseOperations w
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN FactFleetTrips f ON f.OriginGeographyKey = g.GeographyKey
INNER JOIN DimTime tw ON w.TimeKey = tw.TimeKey
INNER JOIN DimTime tf ON f.TimeKey = tf.TimeKey
WHERE tw.DateTime >= DATEADD(DAY, -7, GETDATE())
    AND tf.DateTime >= DATEADD(DAY, -7, GETDATE())
    AND w.OperationType = 'Shipping'
GROUP BY g.LocationName
HAVING AVG(w.DwellTimeMinutes) > 120;
```

### Pattern 2: Temporal Elasticity Simulation

**Simulating capacity changes:**

```sql
WITH CurrentCapacity AS (
    SELECT 
        g.LocationName,
        COUNT(w.OperationKey) AS CurrentOperations,
        AVG(w.ProcessingTimeSeconds) AS AvgProcessingTime
    FROM FactWarehouseOperations w
    INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY g.LocationName
),
SimulatedLoad AS (
    SELECT 
        LocationName,
        CurrentOperations,
        CurrentOperations * 1.20 AS Simulated120Percent,
        AvgProcessingTime,
        -- Assume 10% degradation per 10% capacity increase
        AvgProcessingTime * (1 + (0.20 * 1.0)) AS ProjectedProcessingTime
    FROM CurrentCapacity
)
SELECT 
    LocationName,
    CurrentOperations,
    Simulated120Percent,
    AvgProcessingTime,
    ProjectedProcessingTime,
    (ProjectedProcessingTime - AvgProcessingTime) AS AdditionalTimePerOperation
FROM SimulatedLoad;
```

### Pattern 3: Gravity Zone Optimization

**Recommending storage zone reallocations:**

```sql
CREATE PROCEDURE sp_RecommendGravityReallocation
AS
BEGIN
    -- Identify products in wrong gravity zones
    SELECT 
        p.SKU,
        p.ProductName,
        p.GravityZone AS CurrentZone,
        CASE 
            WHEN p.GravityScore >= 75 THEN 'High'
            WHEN p.GravityScore >= 40 THEN 'Medium'
            ELSE 'Low'
        END AS RecommendedZone,
        p.GravityScore,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(w.OperationKey) AS RecentOperations
    FROM DimProductGravity p
    LEFT JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.SKU, p.ProductName, p.GravityZone, p.GravityScore
    HAVING p.GravityZone != CASE 
            WHEN p.GravityScore >= 75 THEN 'High'
            WHEN p.GravityScore >= 40 THEN 'Medium'
            ELSE 'Low'
        END
        AND COUNT(w.OperationKey) > 10;
END;
GO
```

## Power BI DAX Measures

### Key Performance Indicators

**Average dwell time with comparison:**

```dax
Avg Dwell Time = 
AVERAGE(FactWarehouseOperations[DwellTimeMinutes])

Avg Dwell Time LY = 
CALCULATE(
    [Avg Dwell Time],
    SAMEPERIODLASTYEAR(DimTime[DateTime])
)

Dwell Time Change % = 
DIVIDE(
    [Avg Dwell Time] - [Avg Dwell Time LY],
    [Avg Dwell Time LY],
    0
)
```

**Fleet utilization rate:**

```dax
Fleet Utilization % = 
VAR TotalTripTime = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] + 
        FactFleetTrips[LoadingTimeMinutes] + 
        FactFleetTrips[UnloadingTimeMinutes] +
        (FactFleetTrips[DistanceKM] / FactFleetTrips[AvgSpeed] * 60)
    )
VAR ProductiveTime = 
    SUMX(
        FactFleetTrips,
        (FactFleetTrips[DistanceKM] / FactFleetTrips[AvgSpeed] * 60) +
        FactFleetTrips[LoadingTimeMinutes] + 
        FactFleetTrips[UnloadingTimeMinutes]
    )
RETURN
    DIVIDE(ProductiveTime, TotalTripTime, 0)
```

**Cross-fact composite KPI:**

```dax
Supply Chain Efficiency Score = 
VAR WarehouseScore = 1 - DIVIDE([Avg Dwell Time], 480, 0) // Normalize against 8 hours
VAR FleetScore = [Fleet Utilization %]
VAR WeightedScore = (WarehouseScore * 0.4) + (FleetScore * 0.6)
RETURN WeightedScore
```

## Troubleshooting

### Common Issues

**Issue: Power BI refresh fails with timeout**

Solution: Increase command timeout in Power BI connection settings

```powerquery
// In Power BI Power Query Editor
let
    Source = Sql.Database(
        "${SQL_SERVER_HOST}", 
        "LogiFleetPulse", 
        [CommandTimeout=#duration(0, 0, 10, 0)] // 10 minutes
    )
in
    Source
```

**Issue: Slow query performance on fact tables**

Solution: Verify indexes and consider partitioning

```sql
-- Check index fragmentation
SELECT 
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(
    DB_ID(), NULL, NULL, NULL, 'DETAILED') ips
INNER JOIN sys.indexes i ON ips.object_id = i.object_id 
    AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
    AND ips.page_count > 1000;

-- Rebuild fragmented indexes
ALTER INDEX ALL ON FactWarehouseOperations REBUILD;
ALTER INDEX ALL ON FactFleetTrips REBUILD;
```

**Issue: Missing relationships in Power BI**

Solution: Verify foreign key constraints and refresh Power BI model

```sql
-- Validate referential integrity
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations w
WHERE NOT EXISTS (SELECT 1 FROM DimTime t WHERE t.TimeKey = w.TimeKey)
    OR NOT EXISTS (SELECT 1 FROM DimGeography g WHERE g.GeographyKey = w.GeographyKey)
    OR NOT EXISTS (SELECT 1 FROM DimProductGravity p WHERE p.ProductKey = w.ProductKey);
```

**Issue: Gravity scores not calculating correctly**

Solution: Recalculate gravity scores using latest operational data

```sql
UPDATE DimProductGravity
SET GravityScore = (
    SELECT 
        (COUNT(w.OperationKey) * AVG(p.UnitWeight) * 100.0) / 
        NULLIF(AVG(DATEDIFF(DAY, s.LastEvaluationDate, GETDATE())), 0)
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    LEFT JOIN DimSupplierReliability s ON w.SupplierKey = s.SupplierKey
    WHERE p.ProductKey = DimProductGravity.ProductKey
        AND w.TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE DateTime >= DATEADD(DAY, -90, GETDATE())
        )
)
WHERE GravityScore IS NULL OR GravityScore = 0;

-- Update gravity zones based on new scores
UPDATE DimProductGravity
SET GravityZone = CASE 
    WHEN GravityScore >= 75 THEN 'High'
    WHEN GravityScore >= 40 THEN 'Medium'
    ELSE 'Low'
END;
```

## Best Practices

1. **Incremental loading**: Always use watermark-based incremental loads for fact tables to avoid full refreshes
2. **Time dimension population**: Pre-populate DimTime for at least 5 years to support historical and future analysis
3. **Row-level security**: Implement RLS in Power BI for multi-tenant or department-specific views
4. **Partition fact tables**: For databases with >10M rows, partition by time ranges (monthly or quarterly)
5. **Index maintenance**: Schedule weekly index reorganization and monthly rebuilds
6. **Backup strategy**: Daily differential backups with weekly full backups
7. **Monitor query performance**: Use SQL Server Query Store to identify expensive queries
8. **Data validation**: Implement CHECK constraints on fact tables to ensure data quality

## Additional Resources

- **SQL Server Performance Tuning**: Use Query Store and Execution Plans to optimize bottleneck queries
- **Power BI Optimization**: Implement aggregations for large fact tables to speed up visuals
- **External Data Integration**: Use Polybase for real-time API data ingestion (weather, traffic)
- **Machine Learning**: Integrate Azure ML for advanced predictive bottleneck detection beyond rule-based alerts
