---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform with MS SQL Server data warehousing and Power BI dashboards for warehouse, fleet, and cross-dock operations
triggers:
  - "set up logistics analytics dashboard"
  - "configure warehouse and fleet data model"
  - "build supply chain intelligence with Power BI"
  - "implement multi-fact star schema for logistics"
  - "create real-time fleet optimization dashboard"
  - "deploy warehouse gravity zone analytics"
  - "integrate logistics KPI tracking system"
  - "analyze cross-dock operations and dwell time"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is an advanced logistics intelligence engine that unifies warehouse operations, fleet telemetry, and supply chain metrics into a single semantic data layer. Built on MS SQL Server with Power BI visualizations, it uses a multi-fact star schema to enable cross-domain KPI analysis (e.g., correlating warehouse dwell time with fleet fuel costs).

## What It Does

- **Unified data model** linking warehouse, fleet, and supplier data through time-phased dimensions
- **Real-time dashboards** refreshing every 15 minutes with operational KPIs
- **Predictive bottleneck detection** using historical multi-fact correlations
- **Warehouse gravity zone optimization** based on pick frequency and item value
- **Fleet triage engine** prioritizing maintenance by revenue impact
- **Cross-fact queries** (e.g., "delayed shipments from high-dwell zones due to weather")

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to warehouse management system (WMS) and fleet telemetry data sources

### Setup Steps

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema:**
```bash
# Connect to your SQL Server instance
sqlcmd -S YOUR_SERVER -d YOUR_DATABASE -U YOUR_USER -P $SQL_PASSWORD -i schema/01_create_tables.sql
sqlcmd -S YOUR_SERVER -d YOUR_DATABASE -U YOUR_USER -P $SQL_PASSWORD -i schema/02_create_relationships.sql
sqlcmd -S YOUR_SERVER -d YOUR_DATABASE -U YOUR_USER -P $SQL_PASSWORD -i schema/03_create_stored_procedures.sql
```

3. **Configure data sources:**
```bash
cp config_sample.json config.json
# Edit config.json with your connection strings
```

4. **Import Power BI template:**
```
Open LogiFleet_Pulse_Master.pbit in Power BI Desktop
Enter SQL Server connection details when prompted
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    DurationMinutes INT,
    DwellTimeHours DECIMAL(10,2),
    QuantityHandled INT,
    EmployeeKey INT,
    ZoneKey INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Create clustered columnstore index for analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations
ON FactWarehouseOperations;
```

**FactFleetTrips** - Fleet telemetry and route data:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(12,2),
    OnTimeStatus BIT,
    WeatherConditionKey INT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips
ON FactFleetTrips;
```

**FactCrossDock** - Cross-dock transfer operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    InboundShipmentKey INT NOT NULL,
    OutboundShipmentKey INT NOT NULL,
    ProductKey INT NOT NULL,
    QuantityTransferred INT,
    DockDwellMinutes INT,
    QualityCheckPassed BIT,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Dimension Tables

**DimTime** - Time intelligence with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    DateKey INT,
    TimeOfDay TIME,
    HourOfDay INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    FiscalYear INT
);

-- Populate with time spine
CREATE PROCEDURE PopulateDimTime
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    DECLARE @CurrentTime DATETIME2 = @StartDate;
    
    WHILE @CurrentTime < @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, HourOfDay, MinuteBucket, DayOfWeek, IsWeekend, FiscalPeriod, FiscalYear)
        VALUES (
            CONVERT(INT, FORMAT(@CurrentTime, 'yyyyMMddHHmm')),
            @CurrentTime,
            CONVERT(INT, FORMAT(@CurrentTime, 'yyyyMMdd')),
            CONVERT(TIME, @CurrentTime),
            DATEPART(HOUR, @CurrentTime),
            (DATEPART(MINUTE, @CurrentTime) / 15) * 15,
            DATENAME(WEEKDAY, @CurrentTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentTime) IN (1, 7) THEN 1 ELSE 0 END,
            'FQ' + CAST(DATEPART(QUARTER, @CurrentTime) AS VARCHAR),
            YEAR(@CurrentTime)
        );
        
        SET @CurrentTime = DATEADD(MINUTE, 15, @CurrentTime);
    END
END;
```

**DimProductGravity** - Warehouse gravity zone scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50),
    ProductName VARCHAR(200),
    CategoryHierarchy VARCHAR(500),
    GravityScore DECIMAL(5,2), -- 0-100, higher = faster moving
    PickFrequencyScore DECIMAL(5,2),
    ValueScore DECIMAL(5,2),
    FragilityScore DECIMAL(5,2),
    OptimalZoneType VARCHAR(20), -- 'High-Gravity', 'Mid-Gravity', 'Low-Gravity'
    LastRecalculatedDate DATETIME2
);

-- Calculate gravity score
CREATE PROCEDURE CalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET 
        GravityScore = (PickFrequencyScore * 0.5) + (ValueScore * 0.3) + ((100 - FragilityScore) * 0.2),
        OptimalZoneType = CASE
            WHEN (PickFrequencyScore * 0.5) + (ValueScore * 0.3) + ((100 - FragilityScore) * 0.2) >= 70 THEN 'High-Gravity'
            WHEN (PickFrequencyScore * 0.5) + (ValueScore * 0.3) + ((100 - FragilityScore) * 0.2) >= 40 THEN 'Mid-Gravity'
            ELSE 'Low-Gravity'
        END,
        LastRecalculatedDate = GETDATE();
END;
```

## Key Queries and Patterns

### Cross-Fact KPI Analysis

**Correlate warehouse dwell time with fleet delays:**
```sql
-- Find products with high warehouse dwell that caused fleet delays
SELECT 
    p.SKU,
    p.ProductName,
    AVG(w.DwellTimeHours) AS AvgDwellHours,
    COUNT(DISTINCT f.TripKey) AS AffectedTrips,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    SUM(CASE WHEN f.OnTimeStatus = 0 THEN 1 ELSE 0 END) AS DelayedShipments
FROM FactWarehouseOperations w
INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips f ON w.TimeKey = f.TimeKey AND w.ProductKey = f.ProductKey
WHERE w.DwellTimeHours > 72
    AND f.TripStartTime >= DATEADD(MONTH, -3, GETDATE())
GROUP BY p.SKU, p.ProductName
HAVING AVG(w.DwellTimeHours) > 96
ORDER BY DelayedShipments DESC;
```

### Predictive Bottleneck Detection

**Identify congestion risk zones:**
```sql
CREATE VIEW vw_BottleneckRiskIndex AS
SELECT 
    w.ZoneKey,
    z.ZoneName,
    t.DateKey,
    t.HourOfDay,
    COUNT(*) AS OperationCount,
    AVG(w.DurationMinutes) AS AvgDurationMinutes,
    STDEV(w.DurationMinutes) AS StdDevDuration,
    -- Bottleneck score: high volume + high variance + above-average duration
    (COUNT(*) * 0.4) + 
    (STDEV(w.DurationMinutes) * 0.3) + 
    ((AVG(w.DurationMinutes) - (SELECT AVG(DurationMinutes) FROM FactWarehouseOperations)) * 0.3) AS BottleneckScore
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimZone z ON w.ZoneKey = z.ZoneKey
WHERE t.DateKey >= CONVERT(INT, FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd'))
GROUP BY w.ZoneKey, z.ZoneName, t.DateKey, t.HourOfDay;

-- Query high-risk periods
SELECT TOP 20
    ZoneName,
    DateKey,
    HourOfDay,
    BottleneckScore,
    OperationCount,
    AvgDurationMinutes
FROM vw_BottleneckRiskIndex
WHERE BottleneckScore > 50
ORDER BY BottleneckScore DESC;
```

### Fleet Triage Engine

**Prioritize maintenance by revenue impact:**
```sql
WITH VehicleMetrics AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.TelemetryEngineStatus,
        v.TirePressureStatus,
        COUNT(DISTINCT f.TripKey) AS TripCount,
        AVG(f.LoadWeightKG) AS AvgLoadWeight,
        SUM(CASE WHEN p.ValueScore >= 80 THEN 1 ELSE 0 END) AS HighValueTripCount
    FROM DimVehicle v
    LEFT JOIN FactFleetTrips f ON v.VehicleKey = f.VehicleKey
    LEFT JOIN DimProduct p ON f.ProductKey = p.ProductKey
    WHERE f.TripStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID, v.TelemetryEngineStatus, v.TirePressureStatus
)
SELECT 
    VehicleID,
    TripCount,
    AvgLoadWeight,
    HighValueTripCount,
    -- Triage score: issues * load weight * high-value trips
    CASE 
        WHEN TelemetryEngineStatus = 'Warning' THEN 50
        WHEN TelemetryEngineStatus = 'Critical' THEN 100
        ELSE 0
    END +
    CASE 
        WHEN TirePressureStatus = 'Low' THEN 30
        WHEN TirePressureStatus = 'Critical' THEN 80
        ELSE 0
    END * (AvgLoadWeight / 1000) * (HighValueTripCount * 1.5) AS TriageScore
FROM VehicleMetrics
WHERE TelemetryEngineStatus IN ('Warning', 'Critical') OR TirePressureStatus IN ('Low', 'Critical')
ORDER BY TriageScore DESC;
```

## Stored Procedures for ETL

### Incremental Load Pattern

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load new records from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        OperationStartTime, OperationEndTime, DurationMinutes, 
        DwellTimeHours, QuantityHandled, EmployeeKey, ZoneKey
    )
    SELECT 
        CONVERT(INT, FORMAT(s.OperationStartTime, 'yyyyMMddHHmm')) / 15 * 15 AS TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        s.OperationType,
        s.OperationStartTime,
        s.OperationEndTime,
        DATEDIFF(MINUTE, s.OperationStartTime, s.OperationEndTime) AS DurationMinutes,
        DATEDIFF(HOUR, s.LastMovementTime, s.OperationStartTime) AS DwellTimeHours,
        s.QuantityHandled,
        e.EmployeeKey,
        z.ZoneKey
    FROM StagingWarehouseOps s
    INNER JOIN DimWarehouse w ON s.WarehouseID = w.WarehouseID
    INNER JOIN DimProduct p ON s.SKU = p.SKU
    LEFT JOIN DimEmployee e ON s.EmployeeID = e.EmployeeID
    LEFT JOIN DimZone z ON s.ZoneID = z.ZoneID
    WHERE s.ImportedTime > @LastLoadTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.OperationStartTime = s.OperationStartTime
                AND f.ProductKey = p.ProductKey
        );
    
    -- Update last load timestamp
    UPDATE ETLControl
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

## Power BI Integration

### DAX Measures for Cross-Fact KPIs

**Warehouse-to-Fleet Efficiency Ratio:**
```dax
FleetEfficiencyRatio = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdleTime = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR FleetOnTimeRate = 
    DIVIDE(
        CALCULATE(COUNT(FactFleetTrips[TripKey]), FactFleetTrips[OnTimeStatus] = TRUE()),
        COUNT(FactFleetTrips[TripKey])
    )
RETURN
    DIVIDE(FleetOnTimeRate * 100, AvgDwellTime + (AvgIdleTime / 60))
```

**Gravity Zone Compliance Score:**
```dax
GravityZoneCompliance = 
VAR TotalOps = COUNT(FactWarehouseOperations[OperationKey])
VAR MismatchedOps = 
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FILTER(
            FactWarehouseOperations,
            FactWarehouseOperations[ActualZoneType] <> RELATED(DimProductGravity[OptimalZoneType])
        )
    )
RETURN
    DIVIDE(TotalOps - MismatchedOps, TotalOps) * 100
```

### Power BI Q&A Natural Language Examples

- "Show me shipments delayed due to weather from cold storage zones"
- "Which products have dwell time over 72 hours this quarter"
- "Fleet idle time by route and weather condition last month"
- "Warehouse gravity zone mismatches by product category"

## Configuration

### Connection String Template

```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "${SQL_DATABASE_NAME}",
    "authentication": {
      "type": "SqlLogin",
      "username": "${SQL_USERNAME}",
      "password": "${SQL_PASSWORD}"
    },
    "options": {
      "encrypt": true,
      "trustServerCertificate": false
    }
  },
  "dataRefresh": {
    "intervalMinutes": 15,
    "incrementalLoadEnabled": true
  },
  "externalAPIs": {
    "weatherAPI": {
      "endpoint": "${WEATHER_API_URL}",
      "apiKey": "${WEATHER_API_KEY}"
    },
    "telematicsAPI": {
      "endpoint": "${TELEMATICS_API_URL}",
      "apiKey": "${TELEMATICS_API_KEY}"
    }
  },
  "alerting": {
    "emailServer": "${SMTP_SERVER}",
    "emailFrom": "${ALERT_EMAIL_FROM}",
    "teamsWebhook": "${TEAMS_WEBHOOK_URL}"
  }
}
```

### Row-Level Security Setup

```sql
-- Create security roles in Power BI
CREATE ROLE WarehouseManager;
CREATE ROLE FleetManager;
CREATE ROLE ExecutiveView;

-- Define RLS filters
-- WarehouseManager: see only their warehouse
CREATE FUNCTION dbo.fn_SecurityWarehouse(@UserEmail VARCHAR(100))
RETURNS TABLE
AS
RETURN
    SELECT WarehouseKey
    FROM DimWarehouse w
    INNER JOIN UserWarehouseAccess u ON w.WarehouseKey = u.WarehouseKey
    WHERE u.UserEmail = @UserEmail;

-- Apply in Power BI DAX filter
-- FactWarehouseOperations[WarehouseKey] IN dbo.fn_SecurityWarehouse(USERPRINCIPALNAME())
```

## Alerting System

### Automated Alert Stored Procedure

```sql
CREATE PROCEDURE sp_CheckAndSendAlerts
AS
BEGIN
    -- Check fleet idle time threshold
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips 
        WHERE TripStartTime >= DATEADD(HOUR, -4, GETDATE())
            AND IdleTimeMinutes > 30
        GROUP BY VehicleKey
        HAVING AVG(IdleTimeMinutes) > 45
    )
    BEGIN
        INSERT INTO AlertQueue (AlertType, AlertMessage, Severity, CreatedTime)
        VALUES (
            'FleetIdleTime',
            'Fleet idle time exceeded 45-minute average in last 4 hours',
            'High',
            GETDATE()
        );
    END
    
    -- Check warehouse dwell time threshold
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations
        WHERE OperationStartTime >= DATEADD(DAY, -1, GETDATE())
            AND DwellTimeHours > 96
        GROUP BY ProductKey
        HAVING COUNT(*) > 5
    )
    BEGIN
        INSERT INTO AlertQueue (AlertType, AlertMessage, Severity, CreatedTime)
        VALUES (
            'HighDwellTime',
            'Multiple products exceeding 96-hour dwell time',
            'Medium',
            GETDATE()
        );
    END
END;

-- Schedule via SQL Agent Job to run every 15 minutes
```

## Troubleshooting

### Performance Issues

**Problem:** Slow cross-fact queries
```sql
-- Check for missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.user_seeks,
    s.user_scans,
    s.last_user_seek,
    s.last_user_scan
FROM sys.dm_db_index_usage_stats s
INNER JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE OBJECT_NAME(s.object_id) IN ('FactWarehouseOperations', 'FactFleetTrips', 'FactCrossDock')
ORDER BY s.user_seeks + s.user_scans DESC;

-- Add missing indexes on foreign keys
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_TimeKey 
ON FactWarehouseOperations(TimeKey) INCLUDE (DwellTimeHours);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_TimeKey 
ON FactFleetTrips(TimeKey) INCLUDE (IdleTimeMinutes, OnTimeStatus);
```

### Data Quality Checks

**Validate dimension relationships:**
```sql
-- Check for orphaned fact records
SELECT 'FactWarehouseOperations' AS TableName, COUNT(*) AS OrphanedRecords
FROM FactWarehouseOperations f
LEFT JOIN DimTime t ON f.TimeKey = t.TimeKey
WHERE t.TimeKey IS NULL

UNION ALL

SELECT 'FactFleetTrips', COUNT(*)
FROM FactFleetTrips f
LEFT JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
WHERE v.VehicleKey IS NULL;
```

### Power BI Refresh Failures

**Check connection string in Power BI:**
1. Open Power BI Desktop
2. Go to Transform Data → Data source settings
3. Verify SQL Server connection string matches config
4. Test connection with `SELECT 1` query

**Enable detailed error logging:**
```sql
-- Create audit table
CREATE TABLE PowerBIRefreshLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    RefreshStartTime DATETIME2,
    RefreshEndTime DATETIME2,
    TableName VARCHAR(100),
    RowsAffected INT,
    ErrorMessage VARCHAR(MAX),
    Status VARCHAR(20)
);

-- Log from stored procedures
INSERT INTO PowerBIRefreshLog (RefreshStartTime, TableName, Status)
VALUES (GETDATE(), 'FactWarehouseOperations', 'Started');
```

## Common Workflow Patterns

### Daily Operations Workflow

1. **Morning Report Generation:**
```sql
EXEC sp_IncrementalLoadWarehouseOps @LastLoadTime = (SELECT LastLoadTime FROM ETLControl WHERE TableName = 'FactWarehouseOperations');
EXEC sp_IncrementalLoadFleetTrips @LastLoadTime = (SELECT LastLoadTime FROM ETLControl WHERE TableName = 'FactFleetTrips');
EXEC CalculateProductGravity;
EXEC sp_CheckAndSendAlerts;
```

2. **Weekly Gravity Zone Rebalancing:**
```sql
-- Generate zone rebalancing recommendations
SELECT 
    p.SKU,
    p.ProductName,
    p.OptimalZoneType,
    z.CurrentZoneType,
    p.GravityScore,
    COUNT(w.OperationKey) AS PicksLastWeek
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
INNER JOIN DimZone z ON w.ZoneKey = z.ZoneKey
WHERE w.OperationStartTime >= DATEADD(DAY, -7, GETDATE())
    AND p.OptimalZoneType <> z.CurrentZoneType
GROUP BY p.SKU, p.ProductName, p.OptimalZoneType, z.CurrentZoneType, p.GravityScore
ORDER BY p.GravityScore DESC, PicksLastWeek DESC;
```

3. **Monthly Executive Summary:**
```sql
-- Cross-fact summary for leadership
SELECT 
    'Warehouse Operations' AS Metric,
    COUNT(*) AS TotalOperations,
    AVG(DwellTimeHours) AS AvgDwellHours
FROM FactWarehouseOperations
WHERE OperationStartTime >= DATEADD(MONTH, -1, GETDATE())

UNION ALL

SELECT 
    'Fleet Trips',
    COUNT(*),
    AVG(IdleTimeMinutes / 60.0)
FROM FactFleetTrips
WHERE TripStartTime >= DATEADD(MONTH, -1, GETDATE());
```

This skill provides comprehensive coverage of deploying, configuring, and operating the LogiFleet Pulse logistics analytics platform, with real-world SQL patterns and Power BI integration examples.
