---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data warehouse"
  - "implement logistics star schema in SQL Server"
  - "create Power BI logistics dashboard"
  - "build cross-modal supply chain analytics"
  - "deploy LogiFleet warehouse gravity zones"
  - "integrate fleet telemetry with warehouse operations"
  - "set up predictive logistics bottleneck detection"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a multi-fact star schema data warehouse. Built on MS SQL Server with Power BI visualization, it provides cross-modal analytics linking inventory management, fleet optimization, and predictive bottleneck detection through a semantic data layer.

**Core Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Warehouse gravity zone optimization
- Real-time fleet performance monitoring
- Cross-fact KPI harmonization (warehouse + fleet + supply chain)
- Predictive bottleneck and anomaly detection
- Role-based security and multilingual dashboards

## Installation

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- SQL Server Management Studio (SSMS)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry feeds

### Database Setup

1. **Clone and extract the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the core schema:**
```sql
-- Connect to your SQL Server instance
-- Execute the main schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Run the schema deployment script
:r .\sql\01_create_schema.sql
:r .\sql\02_create_dimensions.sql
:r .\sql\03_create_facts.sql
:r .\sql\04_create_views.sql
:r .\sql\05_create_procedures.sql
```

3. **Configure connection strings:**
```json
{
  "connections": {
    "sql_server": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "wms_api": "${WMS_API_ENDPOINT}",
    "telemetry_endpoint": "${FLEET_TELEMETRY_URL}",
    "weather_api_key": "${WEATHER_API_KEY}"
  }
}
```

## Core Data Model

### Dimension Tables

```sql
-- Time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    Date DATE,
    TimeOfDay TIME,
    HourOfDay INT,
    DayOfWeek INT,
    WeekOfYear INT,
    MonthNumber INT,
    QuarterNumber INT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);

-- Product gravity dimension (velocity-based classification)
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU NVARCHAR(50),
    ProductName NVARCHAR(200),
    Category NVARCHAR(100),
    GravityScore DECIMAL(5,2), -- 0-100 scale
    VelocityClass NVARCHAR(20), -- Fast/Medium/Slow
    ValueClass NVARCHAR(20), -- High/Medium/Low
    FragilityScore INT,
    OptimalZoneID INT,
    LastCalculated DATETIME2
);

-- Geography dimension with hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY,
    LocationID NVARCHAR(50),
    LocationName NVARCHAR(200),
    LocationType NVARCHAR(50), -- Warehouse, DC, Route Node
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone NVARCHAR(50)
);

-- Supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierID NVARCHAR(50),
    SupplierName NVARCHAR(200),
    LeadTimeMean DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore INT, -- 0-100
    ReliabilityGrade NCHAR(1) -- A-F
);
```

### Fact Tables

```sql
-- Warehouse operations fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    LocationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    OperationType NVARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    DurationMinutes DECIMAL(8,2),
    UnitsProcessed INT,
    DwellTimeHours DECIMAL(8,2),
    ZoneID INT,
    WorkerID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    QualityFlag BIT
);

-- Fleet operations fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID NVARCHAR(50),
    DriverID NVARCHAR(50),
    RouteID NVARCHAR(50),
    OriginKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(8,2),
    FuelLiters DECIMAL(8,2),
    IdleMinutes DECIMAL(8,2),
    LoadWeight DECIMAL(10,2),
    AverageSpeed DECIMAL(5,2),
    WeatherCondition NVARCHAR(50),
    DelayMinutes INT,
    DelayReason NVARCHAR(200)
);

-- Cross-dock operations fact
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    UnitsTransferred INT,
    DockDwellMinutes DECIMAL(8,2),
    TemperatureCompliant BIT,
    QualityCheckPassed BIT
);
```

## Key Stored Procedures

### Data Loading Procedures

```sql
-- Incremental warehouse operations load
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Merge from staging to fact table
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            t.TimeKey,
            p.ProductKey,
            g.GeographyKey,
            s.OperationType,
            s.OperationStartTime,
            s.OperationEndTime,
            DATEDIFF(MINUTE, s.OperationStartTime, s.OperationEndTime) AS DurationMinutes,
            s.UnitsProcessed,
            s.DwellTimeHours,
            s.ZoneID,
            s.WorkerID,
            s.EquipmentID,
            s.QualityFlag
        FROM StagingWarehouseOps s
        INNER JOIN DimTime t ON CAST(s.OperationStartTime AS DATE) = t.Date 
            AND DATEPART(HOUR, s.OperationStartTime) = t.HourOfDay
        INNER JOIN DimProductGravity p ON s.SKU = p.SKU
        INNER JOIN DimGeography g ON s.LocationID = g.LocationID
        WHERE s.OperationStartTime BETWEEN @StartDate AND @EndDate
    ) AS source
    ON 1=0 -- Always insert
    WHEN NOT MATCHED THEN
        INSERT (TimeKey, ProductKey, LocationKey, OperationType, OperationStartTime, 
                OperationEndTime, DurationMinutes, UnitsProcessed, DwellTimeHours, 
                ZoneID, WorkerID, EquipmentID, QualityFlag)
        VALUES (source.TimeKey, source.ProductKey, source.GeographyKey, source.OperationType,
                source.OperationStartTime, source.OperationEndTime, source.DurationMinutes,
                source.UnitsProcessed, source.DwellTimeHours, source.ZoneID, 
                source.WorkerID, source.EquipmentID, source.QualityFlag);
END;
GO

-- Fleet telemetry load
CREATE PROCEDURE sp_LoadFleetTrips
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleID, DriverID, RouteID, OriginKey, DestinationKey,
        TripStartTime, TripEndTime, DistanceKM, FuelLiters, IdleMinutes,
        LoadWeight, AverageSpeed, WeatherCondition, DelayMinutes, DelayReason
    )
    SELECT 
        t.TimeKey,
        s.VehicleID,
        s.DriverID,
        s.RouteID,
        o.GeographyKey AS OriginKey,
        d.GeographyKey AS DestinationKey,
        s.TripStartTime,
        s.TripEndTime,
        s.DistanceKM,
        s.FuelLiters,
        s.IdleMinutes,
        s.LoadWeight,
        s.AverageSpeed,
        s.WeatherCondition,
        s.DelayMinutes,
        s.DelayReason
    FROM StagingFleetTrips s
    INNER JOIN DimTime t ON CAST(s.TripStartTime AS DATE) = t.Date
    INNER JOIN DimGeography o ON s.OriginLocationID = o.LocationID
    INNER JOIN DimGeography d ON s.DestinationLocationID = d.LocationID
    WHERE s.TripStartTime BETWEEN @StartDate AND @EndDate
    AND NOT EXISTS (
        SELECT 1 FROM FactFleetTrips f 
        WHERE f.VehicleID = s.VehicleID 
        AND f.TripStartTime = s.TripStartTime
    );
END;
GO
```

### Analytics Views

```sql
-- Cross-fact KPI: Warehouse dwell time impact on fleet efficiency
CREATE VIEW vw_DwellTimeFleetImpact AS
SELECT 
    t.Date,
    t.WeekOfYear,
    g.LocationName,
    p.Category,
    AVG(w.DwellTimeHours) AS AvgDwellTimeHours,
    AVG(f.IdleMinutes) AS AvgFleetIdleMinutes,
    AVG(f.DelayMinutes) AS AvgDelayMinutes,
    COUNT(DISTINCT w.OperationKey) AS WarehouseOperations,
    COUNT(DISTINCT f.TripKey) AS FleetTrips
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON w.LocationKey = g.GeographyKey
LEFT JOIN FactFleetTrips f ON f.OriginKey = g.GeographyKey 
    AND CAST(f.TripStartTime AS DATE) = t.Date
GROUP BY t.Date, t.WeekOfYear, g.LocationName, p.Category;
GO

-- Gravity zone optimization recommendations
CREATE VIEW vw_GravityZoneRecommendations AS
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneID AS CurrentZone,
    w.ZoneID AS ActualZone,
    AVG(w.DurationMinutes) AS AvgPickTimeMinutes,
    COUNT(*) AS OperationCount,
    CASE 
        WHEN p.OptimalZoneID != w.ZoneID THEN 'Reassign to Zone ' + CAST(p.OptimalZoneID AS VARCHAR)
        ELSE 'Optimal'
    END AS Recommendation
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
WHERE w.OperationType = 'Picking'
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneID, w.ZoneID
HAVING COUNT(*) > 10;
GO

-- Predictive bottleneck index
CREATE VIEW vw_BottleneckPrediction AS
WITH HourlyMetrics AS (
    SELECT 
        TimeKey,
        LocationKey,
        COUNT(*) AS OperationCount,
        AVG(DurationMinutes) AS AvgDuration,
        STDEV(DurationMinutes) AS StdDevDuration
    FROM FactWarehouseOperations
    WHERE OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY TimeKey, LocationKey
)
SELECT 
    t.FullDateTime,
    g.LocationName,
    hm.OperationCount,
    hm.AvgDuration,
    hm.StdDevDuration,
    CASE 
        WHEN hm.OperationCount > (SELECT AVG(OperationCount) * 1.5 FROM HourlyMetrics) 
             AND hm.StdDevDuration > 5 
        THEN 'High Risk'
        WHEN hm.OperationCount > (SELECT AVG(OperationCount) * 1.2 FROM HourlyMetrics)
        THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM HourlyMetrics hm
INNER JOIN DimTime t ON hm.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON hm.LocationKey = g.GeographyKey;
GO
```

### Alert Configuration

```sql
-- Create alert threshold table
CREATE TABLE AlertThresholds (
    AlertID INT PRIMARY KEY IDENTITY,
    AlertName NVARCHAR(100),
    MetricName NVARCHAR(100),
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator NVARCHAR(10), -- >, <, >=, <=, =
    AlertLevel NVARCHAR(20), -- Critical, Warning, Info
    NotificationEmails NVARCHAR(MAX),
    IsActive BIT DEFAULT 1
);

-- Alert evaluation procedure
CREATE PROCEDURE sp_EvaluateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertResults TABLE (
        AlertName NVARCHAR(100),
        CurrentValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        AlertLevel NVARCHAR(20),
        NotificationEmails NVARCHAR(MAX)
    );
    
    -- Fleet idle time alert
    INSERT INTO @AlertResults
    SELECT 
        'Fleet Idle Time Exceeded',
        AVG(f.IdleMinutes) AS CurrentValue,
        a.ThresholdValue,
        a.AlertLevel,
        a.NotificationEmails
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    CROSS APPLY (SELECT ThresholdValue, AlertLevel, NotificationEmails 
                 FROM AlertThresholds 
                 WHERE AlertName = 'Fleet Idle Time Exceeded' AND IsActive = 1) a
    WHERE t.Date = CAST(GETDATE() AS DATE)
    GROUP BY a.ThresholdValue, a.AlertLevel, a.NotificationEmails
    HAVING AVG(f.IdleMinutes) > a.ThresholdValue;
    
    -- Warehouse dwell time alert
    INSERT INTO @AlertResults
    SELECT 
        'Warehouse Dwell Time Exceeded',
        AVG(w.DwellTimeHours) AS CurrentValue,
        a.ThresholdValue,
        a.AlertLevel,
        a.NotificationEmails
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    CROSS APPLY (SELECT ThresholdValue, AlertLevel, NotificationEmails 
                 FROM AlertThresholds 
                 WHERE AlertName = 'Warehouse Dwell Time Exceeded' AND IsActive = 1) a
    WHERE t.Date = CAST(GETDATE() AS DATE)
    GROUP BY a.ThresholdValue, a.AlertLevel, a.NotificationEmails
    HAVING AVG(w.DwellTimeHours) > a.ThresholdValue;
    
    -- Return results for notification processing
    SELECT * FROM @AlertResults;
END;
GO
```

## Power BI Integration

### Connection Setup

```sql
-- Create read-only user for Power BI
CREATE LOGIN PowerBIReader WITH PASSWORD = '${POWERBI_PASSWORD}';
USE LogiFleetPulse;
CREATE USER PowerBIReader FOR LOGIN PowerBIReader;
GRANT SELECT ON SCHEMA::dbo TO PowerBIReader;
GRANT EXECUTE ON SCHEMA::dbo TO PowerBIReader;
GO
```

### DAX Measures for Cross-Fact Analysis

```dax
-- Warehouse efficiency score
Warehouse Efficiency Score = 
VAR AvgDuration = AVERAGE(FactWarehouseOperations[DurationMinutes])
VAR StdDev = STDEV.P(FactWarehouseOperations[DurationMinutes])
VAR Baseline = 15 -- Target minutes per operation
RETURN
IF(
    AvgDuration <= Baseline,
    100,
    MAX(0, 100 - ((AvgDuration - Baseline) / Baseline * 100))
)

-- Fleet utilization rate
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKM]),
    COUNTROWS(FactFleetTrips) * 500, -- Assume 500km optimal daily distance
    0
) * 100

-- Cross-modal efficiency index
Cross-Modal Efficiency = 
VAR WarehouseScore = [Warehouse Efficiency Score]
VAR FleetScore = [Fleet Utilization %]
VAR DwellImpact = 
    IF(
        AVERAGE(FactWarehouseOperations[DwellTimeHours]) > 48,
        0.9, -- 10% penalty
        1
    )
RETURN
(WarehouseScore * 0.5 + FleetScore * 0.5) * DwellImpact

-- Gravity zone compliance
Gravity Zone Compliance % = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR CompliantOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[ZoneID] = RELATED(DimProductGravity[OptimalZoneID])
    )
RETURN
DIVIDE(CompliantOps, TotalOps, 0) * 100
```

### Power BI Template Configuration

1. Open `LogiFleet_Pulse_Master.pbit`
2. Enter connection parameters:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: SQL Server (username: `PowerBIReader`)
3. Refresh data model to load schema
4. Publish to Power BI Service for scheduled refresh

## Common Usage Patterns

### Daily Operations Dashboard Query

```sql
-- Executive summary for current day
SELECT 
    -- Warehouse metrics
    (SELECT COUNT(*) FROM FactWarehouseOperations w 
     INNER JOIN DimTime t ON w.TimeKey = t.TimeKey 
     WHERE t.Date = CAST(GETDATE() AS DATE)) AS TodayWarehouseOps,
    
    (SELECT AVG(DurationMinutes) FROM FactWarehouseOperations w 
     INNER JOIN DimTime t ON w.TimeKey = t.TimeKey 
     WHERE t.Date = CAST(GETDATE() AS DATE)) AS AvgOpDuration,
    
    -- Fleet metrics
    (SELECT COUNT(*) FROM FactFleetTrips f 
     INNER JOIN DimTime t ON f.TimeKey = t.TimeKey 
     WHERE t.Date = CAST(GETDATE() AS DATE)) AS TodayTrips,
    
    (SELECT AVG(IdleMinutes) FROM FactFleetTrips f 
     INNER JOIN DimTime t ON f.TimeKey = t.TimeKey 
     WHERE t.Date = CAST(GETDATE() AS DATE)) AS AvgIdleMinutes,
    
    -- Cross-modal
    (SELECT COUNT(*) FROM vw_BottleneckPrediction 
     WHERE CAST(FullDateTime AS DATE) = CAST(GETDATE() AS DATE) 
     AND BottleneckRisk = 'High Risk') AS HighRiskBottlenecks;
```

### Product Gravity Recalculation

```sql
-- Recalculate gravity scores based on last 30 days
CREATE PROCEDURE sp_RecalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Update gravity scores
    UPDATE p
    SET 
        p.GravityScore = scores.NewScore,
        p.VelocityClass = CASE 
            WHEN scores.NewScore >= 70 THEN 'Fast'
            WHEN scores.NewScore >= 40 THEN 'Medium'
            ELSE 'Slow'
        END,
        p.OptimalZoneID = CASE
            WHEN scores.NewScore >= 70 THEN 1 -- Closest to shipping
            WHEN scores.NewScore >= 40 THEN 2
            ELSE 3 -- Back of warehouse
        END,
        p.LastCalculated = GETDATE()
    FROM DimProductGravity p
    INNER JOIN (
        SELECT 
            ProductKey,
            -- Weighted score: 50% frequency, 30% value, 20% fragility
            (
                (COUNT(*) / NULLIF((SELECT MAX(OpCount) FROM (
                    SELECT COUNT(*) AS OpCount 
                    FROM FactWarehouseOperations 
                    GROUP BY ProductKey
                ) x), 0)) * 50 +
                (AVG(CAST(QualityFlag AS INT)) * 30) +
                (AVG(CAST(FragilityScore AS FLOAT)) / 10 * 20)
            ) AS NewScore
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    ) scores ON p.ProductKey = scores.ProductKey;
END;
GO
```

### Temporal Elasticity Simulation

```sql
-- Simulate impact of warehouse capacity change
CREATE PROCEDURE sp_SimulateCapacityChange
    @CapacityChangePercent DECIMAL(5,2),
    @SimulationDays INT = 7
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Create temporary results table
    CREATE TABLE #SimulationResults (
        ScenarioID INT,
        Date DATE,
        ProjectedOperations INT,
        ProjectedAvgDuration DECIMAL(8,2),
        ProjectedDwellTime DECIMAL(8,2),
        CapacityUtilization DECIMAL(5,2)
    );
    
    -- Baseline scenario
    INSERT INTO #SimulationResults
    SELECT 
        1 AS ScenarioID,
        t.Date,
        COUNT(*) AS ProjectedOperations,
        AVG(w.DurationMinutes) AS ProjectedAvgDuration,
        AVG(w.DwellTimeHours) AS ProjectedDwellTime,
        100.0 AS CapacityUtilization
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -@SimulationDays, GETDATE())
    GROUP BY t.Date;
    
    -- Adjusted scenario
    INSERT INTO #SimulationResults
    SELECT 
        2 AS ScenarioID,
        t.Date,
        CAST(COUNT(*) * (1 + @CapacityChangePercent / 100.0) AS INT),
        AVG(w.DurationMinutes) * (1 + (@CapacityChangePercent / 100.0 * 0.3)), -- Assume 30% duration impact
        AVG(w.DwellTimeHours) * (1 - (@CapacityChangePercent / 100.0 * 0.2)), -- Assume 20% dwell reduction
        100.0 + @CapacityChangePercent AS CapacityUtilization
    FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -@SimulationDays, GETDATE())
    GROUP BY t.Date;
    
    -- Return comparison
    SELECT * FROM #SimulationResults ORDER BY ScenarioID, Date;
    
    DROP TABLE #SimulationResults;
END;
GO
```

## Troubleshooting

### Performance Optimization

```sql
-- Create recommended indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeProduct 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DurationMinutes, DwellTimeHours);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeOriginDest 
ON FactFleetTrips(TimeKey, OriginKey, DestinationKey) 
INCLUDE (IdleMinutes, DelayMinutes);

CREATE NONCLUSTERED INDEX IX_DimTime_Date 
ON DimTime(Date) 
INCLUDE (HourOfDay, DayOfWeek);

-- Partition large fact tables by date
ALTER PARTITION SCHEME ps_MonthlyPartition NEXT USED [PRIMARY];
ALTER TABLE FactWarehouseOperations 
ADD CONSTRAINT PK_FactWarehouse PRIMARY KEY CLUSTERED (OperationKey, TimeKey)
ON ps_MonthlyPartition(TimeKey);
```

### Data Quality Checks

```sql
-- Identify orphaned records
SELECT 'Warehouse Ops with missing Time' AS Issue, COUNT(*) AS Count
FROM FactWarehouseOperations w
LEFT JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL

UNION ALL

SELECT 'Fleet Trips with missing Geography' AS Issue, COUNT(*) AS Count
FROM FactFleetTrips f
LEFT JOIN DimGeography g ON f.OriginKey = g.GeographyKey
WHERE g.GeographyKey IS NULL

UNION ALL

SELECT 'Products with outdated gravity scores' AS Issue, COUNT(*) AS Count
FROM DimProductGravity
WHERE LastCalculated < DATEADD(DAY, -30, GETDATE());
```

### Common Errors

**Error: "Timeout expired waiting for data load"**
- Increase batch size in `sp_LoadWarehouseOperations` parameters
- Partition staging tables by date range
- Use `WITH (TABLOCK)` hint for bulk inserts

**Error: "Power BI DirectQuery too slow"**
- Switch to Import mode for large fact tables
- Create aggregation tables for summary views:

```sql
CREATE TABLE AggWarehouseDaily (
    Date DATE PRIMARY KEY,
    TotalOperations INT,
    AvgDuration DECIMAL(8,2),
    AvgDwellTime DECIMAL(8,2)
);

-- Populate via scheduled job
INSERT INTO AggWarehouseDaily
SELECT 
    t.Date,
    COUNT(*),
    AVG(w.DurationMinutes),
    AVG(w.DwellTimeHours)
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
GROUP BY t.Date;
```

**Error: "Cross-fact joins returning incorrect results"**
- Verify dimensional conformity between fact tables
- Check for many-to-many relationships requiring bridge tables:

```sql
-- Create bridge table for product-route relationships
CREATE TABLE BridgeProductRoute (
    ProductKey INT,
    RouteID NVARCHAR(50),
    AllocationPercent DECIMAL(5,2),
    PRIMARY KEY (ProductKey, RouteID)
);
```

## Best Practices

1. **Incremental Loading**: Always use date-range parameters for fact table loads
2. **Dimension Management**: Use slowly changing dimension (SCD) Type 2 for products with changing gravity scores
3. **Security**: Implement row-level security in Power BI based on `DimGeography` for regional access control
4. **Monitoring**: Schedule `sp_EvaluateAlerts` every 15 minutes via SQL Agent job
5. **Backup Strategy**: Daily full backup of dimension tables, hourly differential for fact tables

## Additional Resources

- Power BI template location: `./powerbi/LogiFleet_Pulse_Master.pbit`
- Sample data generators: `./scripts/generate_sample_data.sql`
- Data dictionary: `./docs/data_dictionary.xlsx`
- Scheduled job configurations: `./sql/06_create_jobs.sql`
