---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine with MS SQL Server data warehousing and Power BI dashboards for supply chain optimization
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse analytics"
  - "create cross-fact KPI queries for logistics"
  - "implement warehouse gravity zone optimization"
  - "set up real-time logistics dashboards"
  - "query fleet telemetry and warehouse operations"
  - "build supply chain bottleneck predictions"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a cohesive MS SQL Server data warehouse with Power BI visualization layer. It uses a multi-fact star schema architecture to enable cross-domain KPI analysis and predictive bottleneck detection.

## What It Does

- **Multi-modal data integration**: Combines warehouse management, fleet GPS/telemetry, supplier data, and external APIs (weather, traffic)
- **Unified semantic layer**: Single data model linking disparate logistics operations
- **Real-time dashboards**: Power BI reports refreshed every 15 minutes
- **Predictive analytics**: Bottleneck detection, maintenance prioritization, spatial optimization
- **Cross-fact analysis**: Query relationships between warehouse dwell time and fleet costs in single queries

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources: WMS, TMS/telematics, ERP systems

### 1. Deploy SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Deploy core dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(20),
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT NOT NULL,
    CONSTRAINT UQ_DimTime_FullDateTime UNIQUE (FullDateTime)
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Distribution Center'
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    IsActive BIT DEFAULT 1
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(200),
    Subcategory VARCHAR(200),
    GravityScore DECIMAL(5,2), -- Computed: velocity × value × fragility
    VelocityClass VARCHAR(20), -- 'Fast-Mover', 'Slow-Mover', 'Dead Stock'
    ValueTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2),
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    UpdatedAt DATETIME2 DEFAULT GETDATE()
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(300),
    LeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- 'Tier 1', 'Tier 2', 'At Risk'
    LastAssessmentDate DATE
);

-- Deploy fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    OperationID VARCHAR(100) UNIQUE NOT NULL,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT,
    DwellTimeMinutes INT,
    PickCycleSeconds INT,
    StorageZone VARCHAR(50),
    OperatorID VARCHAR(50),
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID VARCHAR(100) UNIQUE NOT NULL,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    RouteSegment VARCHAR(200),
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DelayMinutes INT,
    DelayReason VARCHAR(200)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    CrossDockID VARCHAR(100) UNIQUE NOT NULL,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT,
    DockDwellMinutes INT,
    TransferStartTime DATETIME2,
    TransferEndTime DATETIME2
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
```

### 2. Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "connections": {
    "sql_server": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetPulse",
      "authentication": "integrated",
      "trusted_connection": true
    },
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key_env": "WMS_API_KEY"
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key_env": "TELEMATICS_API_KEY"
    }
  },
  "refresh_interval_minutes": 15,
  "alert_thresholds": {
    "max_dwell_time_hours": 72,
    "max_idle_time_percent": 15,
    "min_compliance_score": 75
  }
}
```

### 3. Deploy Stored Procedures for ETL

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging
    INSERT INTO FactWarehouseOperations (
        OperationID, TimeKey, GeographyKey, ProductKey,
        OperationType, Quantity, DwellTimeMinutes,
        PickCycleSeconds, StorageZone, OperatorID,
        OperationStartTime, OperationEndTime
    )
    SELECT 
        stg.OperationID,
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.OperationStartTime, stg.OperationEndTime),
        stg.PickCycleSeconds,
        stg.StorageZone,
        stg.OperatorID,
        stg.OperationStartTime,
        stg.OperationEndTime
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON DATEADD(MINUTE, 
        (DATEPART(MINUTE, stg.OperationStartTime) / 15) * 15, 
        CAST(CAST(stg.OperationStartTime AS DATE) AS DATETIME2) + 
        CAST(CAST(DATEPART(HOUR, stg.OperationStartTime) AS VARCHAR) + ':00:00' AS TIME)
    ) = t.FullDateTime
    INNER JOIN DimGeography g ON stg.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    WHERE stg.OperationStartTime > @LastLoadDateTime
    AND NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f 
        WHERE f.OperationID = stg.OperationID
    );
END;
GO

-- Update product gravity scores
CREATE PROCEDURE usp_UpdateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Recalculate gravity scores based on recent activity
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(*) AS OperationCount,
            AVG(CAST(f.DwellTimeMinutes AS FLOAT)) AS AvgDwellTime,
            STDEV(f.DwellTimeMinutes) AS DwellTimeVariance
        FROM FactWarehouseOperations f
        INNER JOIN DimProductGravity p ON f.ProductKey = p.ProductKey
        WHERE f.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY p.ProductKey
    )
    UPDATE p
    SET 
        GravityScore = CASE 
            WHEN pm.AvgDwellTime < 24 THEN 90 + (p.ValueTier = 'High') * 10
            WHEN pm.AvgDwellTime < 72 THEN 60 + (p.ValueTier = 'High') * 10
            ELSE 30 + (p.ValueTier = 'High') * 10
        END,
        VelocityClass = CASE 
            WHEN pm.AvgDwellTime < 24 THEN 'Fast-Mover'
            WHEN pm.AvgDwellTime < 72 THEN 'Medium-Mover'
            ELSE 'Slow-Mover'
        END,
        UpdatedAt = GETDATE()
    FROM DimProductGravity p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO
```

## Key SQL Queries & Patterns

### Cross-Fact Analysis: Dwell Time vs Fleet Idle Time

```sql
-- Find products with high warehouse dwell time that also cause fleet delays
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    COUNT(DISTINCT ft.TripID) AS AffectedTrips,
    SUM(ft.FuelConsumedLiters * 1.5) AS EstimatedIdleCostUSD
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN FactCrossDock cd ON wo.ProductKey = cd.ProductKey
INNER JOIN FactFleetTrips ft ON cd.OutboundTripKey = ft.TripKey
WHERE wo.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY p.SKU, p.ProductName, p.GravityScore
HAVING AVG(wo.DwellTimeMinutes) > 72 
   AND AVG(ft.IdleTimeMinutes) > 30
ORDER BY EstimatedIdleCostUSD DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products that should be moved to high-gravity zones
WITH CurrentZonePerformance AS (
    SELECT 
        wo.StorageZone,
        p.ProductKey,
        p.SKU,
        p.GravityScore,
        AVG(wo.PickCycleSeconds) AS AvgPickTime,
        COUNT(*) AS PickFrequency
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    WHERE wo.OperationType = 'Picking'
      AND wo.OperationStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY wo.StorageZone, p.ProductKey, p.SKU, p.GravityScore
)
SELECT 
    SKU,
    StorageZone AS CurrentZone,
    CASE 
        WHEN GravityScore >= 80 THEN 'Zone-A-Express'
        WHEN GravityScore >= 60 THEN 'Zone-B-Standard'
        ELSE 'Zone-C-Bulk'
    END AS RecommendedZone,
    GravityScore,
    PickFrequency,
    AvgPickTime,
    (PickFrequency * AvgPickTime) / 60.0 AS TotalPickHours
FROM CurrentZonePerformance
WHERE StorageZone != CASE 
    WHEN GravityScore >= 80 THEN 'Zone-A-Express'
    WHEN GravityScore >= 60 THEN 'Zone-B-Standard'
    ELSE 'Zone-C-Bulk'
END
ORDER BY TotalPickHours DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify upcoming bottlenecks based on pattern analysis
WITH BottleneckPatterns AS (
    SELECT 
        g.LocationName,
        t.HourOfDay,
        t.DayOfWeek,
        AVG(wo.DwellTimeMinutes) AS AvgDwell,
        STDEV(wo.DwellTimeMinutes) AS DwellVariance,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.OperationStartTime >= DATEADD(DAY, -90, GETDATE())
    GROUP BY g.LocationName, t.HourOfDay, t.DayOfWeek
    HAVING COUNT(*) > 50
)
SELECT 
    LocationName,
    DayOfWeek,
    HourOfDay,
    AvgDwell,
    CASE 
        WHEN AvgDwell > 120 AND DwellVariance > 30 THEN 'Critical'
        WHEN AvgDwell > 90 THEN 'High Risk'
        WHEN AvgDwell > 60 THEN 'Moderate Risk'
        ELSE 'Normal'
    END AS BottleneckRisk,
    OperationCount
FROM BottleneckPatterns
WHERE AvgDwell > 60
ORDER BY AvgDwell DESC, DwellVariance DESC;
```

### Fleet Maintenance Prioritization

```sql
-- Rank vehicles by maintenance urgency weighted by revenue impact
WITH VehicleMetrics AS (
    SELECT 
        ft.VehicleID,
        COUNT(*) AS TripCount,
        SUM(ft.DistanceKm) AS TotalDistance,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelEfficiency,
        SUM(CASE WHEN ft.DelayMinutes > 0 THEN 1 ELSE 0 END) AS DelayCount,
        SUM(p.GravityScore * cd.Quantity) AS TotalRevenueWeight
    FROM FactFleetTrips ft
    LEFT JOIN FactCrossDock cd ON ft.TripKey = cd.OutboundTripKey
    LEFT JOIN DimProductGravity p ON cd.ProductKey = p.ProductKey
    WHERE ft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY ft.VehicleID
)
SELECT 
    VehicleID,
    TripCount,
    TotalDistance,
    AvgFuelEfficiency,
    DelayCount,
    TotalRevenueWeight,
    (DelayCount * 10 + (TotalDistance / 1000) * 5 + (10 - AvgFuelEfficiency) * 3) 
        * (TotalRevenueWeight / 1000) AS MaintenancePriorityScore
FROM VehicleMetrics
ORDER BY MaintenancePriorityScore DESC;
```

## Power BI Integration

### Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_HOST}`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)

### Create Calculated Measures (DAX)

```dax
-- Total Dwell Cost (combining warehouse and fleet)
TotalDwellCost = 
VAR WarehouseDwellCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] / 60 * 15
    )
VAR FleetIdleCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[IdleTimeMinutes] / 60 * 
        FactFleetTrips[FuelConsumedLiters] / 
        DIVIDE(FactFleetTrips[DistanceKm], 10) * 1.5
    )
RETURN WarehouseDwellCost + FleetIdleCost

-- Gravity Zone Effectiveness
GravityZoneEffectiveness = 
AVERAGEX(
    SUMMARIZE(
        FactWarehouseOperations,
        DimProductGravity[SKU],
        "ZoneMatch", 
        IF(
            AND(
                DimProductGravity[GravityScore] >= 80,
                FactWarehouseOperations[StorageZone] = "Zone-A-Express"
            ),
            1,
            IF(
                AND(
                    DimProductGravity[GravityScore] < 80,
                    DimProductGravity[GravityScore] >= 60,
                    FactWarehouseOperations[StorageZone] = "Zone-B-Standard"
                ),
                1,
                0
            )
        )
    ),
    [ZoneMatch]
) * 100

-- Predictive Bottleneck Index
BottleneckIndex = 
VAR CurrentHour = HOUR(NOW())
VAR CurrentDay = WEEKDAY(NOW())
VAR HistoricalAvg = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        FILTER(
            DimTime,
            DimTime[HourOfDay] = CurrentHour &&
            DimTime[DayOfWeek] = CurrentDay
        )
    )
RETURN 
    IF(HistoricalAvg > 90, "Critical", 
    IF(HistoricalAvg > 60, "Warning", "Normal"))
```

### Create Real-Time Dashboard

Key visuals to include:

1. **Map**: Fleet location by GPS coordinates (DimGeography[Latitude], [Longitude])
2. **Line chart**: Dwell time trend by 15-minute intervals
3. **Matrix**: Cross-fact KPI table (SKU, Warehouse Dwell, Fleet Idle, Cost)
4. **Gauge**: Gravity Zone Effectiveness (target: 85%)
5. **Table**: Top 10 bottleneck risks with priority scores
6. **Bar chart**: Fleet maintenance priority ranking

## Automated Alerting

```sql
-- Create alert procedure
CREATE PROCEDURE usp_GenerateLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert 1: Excessive dwell time
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'ExcessiveDwell',
        'High',
        CONCAT('SKU ', p.SKU, ' has dwell time of ', 
               AVG(wo.DwellTimeMinutes), ' minutes at ', g.LocationName),
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.OperationStartTime >= DATEADD(HOUR, -4, GETDATE())
    GROUP BY p.SKU, g.LocationName
    HAVING AVG(wo.DwellTimeMinutes) > 180;
    
    -- Alert 2: Fleet idle time threshold
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'FleetIdle',
        'Medium',
        CONCAT('Vehicle ', ft.VehicleID, ' idle time: ', 
               ft.IdleTimeMinutes, ' minutes on route ', ft.RouteSegment),
        GETDATE()
    FROM FactFleetTrips ft
    WHERE ft.TripStartTime >= DATEADD(HOUR, -2, GETDATE())
      AND (ft.IdleTimeMinutes * 100.0 / DATEDIFF(MINUTE, ft.TripStartTime, ft.TripEndTime)) > 15;
    
    -- Alert 3: Low supplier compliance
    INSERT INTO AlertLog (AlertType, Severity, Message, CreatedAt)
    SELECT 
        'SupplierRisk',
        'Critical',
        CONCAT('Supplier ', s.SupplierName, ' compliance score dropped to ', 
               s.ComplianceScore),
        GETDATE()
    FROM DimSupplierReliability s
    WHERE s.ComplianceScore < 75
      AND s.LastAssessmentDate >= DATEADD(DAY, -7, GETDATE());
END;
GO

-- Schedule with SQL Server Agent
EXEC msdb.dbo.sp_add_job 
    @job_name = N'LogisticsAlertMonitor';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogisticsAlertMonitor',
    @step_name = N'GenerateAlerts',
    @subsystem = N'TSQL',
    @command = N'EXEC usp_GenerateLogisticsAlerts;',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogisticsAlertMonitor',
    @schedule_name = N'Every15Minutes';
```

## Configuration Best Practices

### Performance Optimization

```sql
-- Partition fact tables by date for better query performance
CREATE PARTITION FUNCTION PF_LogiFleetDate (DATETIME2)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01', '2025-04-01', '2025-07-01', '2025-10-01',
    '2026-01-01', '2026-04-01', '2026-07-01', '2026-10-01'
);

CREATE PARTITION SCHEME PS_LogiFleetDate
AS PARTITION PF_LogiFleetDate
ALL TO ([PRIMARY]);

-- Apply to fact tables
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- Same schema as original
    OperationKey BIGINT PRIMARY KEY,
    OperationStartTime DATETIME2,
    -- ... other columns
) ON PS_LogiFleetDate(OperationStartTime);

-- Create columnstore index for analytics
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, 
    DwellTimeMinutes, PickCycleSeconds, Quantity
);
```

### Row-Level Security

```sql
-- Create security policy for multi-tenant access
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_SecurityPredicate(@LocationID VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessGranted
WHERE @LocationID IN (
    SELECT LocationID FROM dbo.UserLocationAccess
    WHERE UserName = USER_NAME()
);
GO

CREATE SECURITY POLICY SecurityPolicy
ADD FILTER PREDICATE Security.fn_SecurityPredicate(LocationID)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE Security.fn_SecurityPredicate(OriginGeographyKey)
ON dbo.FactFleetTrips
WITH (STATE = ON);
```

## Troubleshooting

### Issue: Slow Dashboard Refresh

**Solution**: Switch from DirectQuery to Import mode, or create aggregation tables:

```sql
CREATE TABLE AggWarehouseDaily AS
SELECT 
    CAST(wo.OperationStartTime AS DATE) AS OperationDate,
    wo.GeographyKey,
    wo.ProductKey,
    wo.OperationType,
    COUNT(*) AS OperationCount,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    SUM(wo.Quantity) AS TotalQuantity
FROM FactWarehouseOperations wo
GROUP BY CAST(wo.OperationStartTime AS DATE), 
         wo.GeographyKey, wo.ProductKey, wo.OperationType;
```

### Issue: Missing Time Dimension Records

**Solution**: Generate time dimension data programmatically:

```sql
DECLARE @StartDate DATETIME2 = '2025-01-01';
DECLARE @EndDate DATETIME2 = '2027-12-31';

;WITH TimeSeries AS (
    SELECT @StartDate AS FullDateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, FullDateTime)
    FROM TimeSeries
    WHERE FullDateTime < @EndDate
)
INSERT INTO DimTime (FullDateTime, DateKey, TimeOfDay, HourOfDay, DayOfWeek, DayName, IsWeekend)
SELECT 
    FullDateTime,
    CAST(FORMAT(FullDateTime, 'yyyyMMdd') AS INT),
    CAST(FullDateTime AS TIME),
    DATEPART(HOUR, FullDateTime),
    DATEPART(WEEKDAY, FullDateTime),
    DATENAME(WEEKDAY, FullDateTime),
    CASE WHEN DATEPART(WEEKDAY, FullDateTime) IN (1, 7) THEN 1 ELSE 0 END
FROM TimeSeries
OPTION (MAXRECURSION 0);
```

### Issue: Gravity Score Not Updating

**Solution**: Schedule the update procedure:

```sql
-- Verify stored procedure execution
EXEC usp_UpdateProductGravity;

-- Check last update timestamps
SELECT 
    SKU, 
    ProductName, 
    GravityScore, 
    UpdatedAt,
    DATEDIFF(HOUR, UpdatedAt, GETDATE()) AS HoursSinceUpdate
FROM DimProductGravity
ORDER BY UpdatedAt DESC;
```

### Issue: Cross-Fact Queries Timeout

**Solution**: Create indexed views for common join patterns:

```sql
CREATE VIEW vw_WarehouseFleetCrossFact
WITH SCHEMABINDING
AS
SELECT 
    wo.ProductKey,
    wo.GeographyKey,
    wo.TimeKey,
    wo.DwellTimeMinutes,
    ft.IdleTimeMinutes,
    ft.FuelConsumedLiters,
    cd.DockDwellMinutes
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.FactCrossDock cd ON wo.ProductKey = cd.ProductKey
INNER JOIN dbo.FactFleetTrips ft ON cd.OutboundTripKey = ft.TripKey;
GO

CREATE UNIQUE CLUSTERED INDEX IX_WarehouseFleetCross
ON vw_WarehouseFleetCrossFact(ProductKey, TimeKey, GeographyKey);
```

## Common Patterns

### Pattern 1: Time-Phased Simulation

```sql
-- Simulate impact of 20% capacity increase
WITH BaselineMetrics AS (
    SELECT 
        AVG(DwellTimeMinutes) AS CurrentAvgDwell,
        COUNT(*) AS CurrentOperations
    FROM FactWarehouseOperations
    WHERE OperationStartTime >= DATEADD(DAY, -30, GETDATE())
)
SELECT 
    'Current' AS Scenario,
    CurrentAvgDwell,
    CurrentOperations,
    CurrentAvgDwell * CurrentOperations AS TotalDwellMinutes
FROM BaselineMetrics
UNION ALL
SELECT 
    'Increased Capacity',
    CurrentAvgDwell * 0.83, -- Estimated 17% reduction
    CurrentOperations * 1.2,
    (CurrentAvgDwell * 0.83) * (CurrentOperations * 1.2)
FROM BaselineMetrics;
```

### Pattern 2: Collaborative Anomaly Tagging

```sql
CREATE TABLE AnomalyAnnotations (
    AnnotationID INT PRIMARY KEY IDENTITY,
    EntityType VARCHAR(50), -- 'Trip', 'Operation', 'Supplier'
    EntityID VARCHAR(100),
    AnomalyDescription VARCHAR(MAX),
    RootCause VARCHAR(500),
    AnnotatedBy VARCHAR(100),
    AnnotatedAt DATETIME2 DEFAULT GETDATE(),
    IsVerified BIT DEFAULT
