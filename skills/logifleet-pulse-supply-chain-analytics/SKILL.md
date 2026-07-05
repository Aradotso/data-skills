---
name: logifleet-pulse-supply-chain-analytics
description: Advanced SQL Server and Power BI logistics data warehousing platform with multi-fact star schema for supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure SQL Server logistics data warehouse"
  - "implement Power BI logistics dashboard"
  - "create multi-fact star schema for fleet management"
  - "build warehouse gravity zone analytics"
  - "deploy cross-modal supply chain reporting"
  - "integrate fleet telemetry with warehouse operations"
  - "configure logistics KPI harmonization"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, and supply chain data into a single semantic layer using MS SQL Server and Power BI. It features a multi-fact star schema architecture with time-phased dimensions, enabling cross-fact KPI analysis that links inventory turnover with fleet performance, warehouse efficiency with delivery optimization, and supplier reliability with operational bottlenecks.

**Core Capabilities:**
- Multi-fact data warehouse (warehouse ops, fleet trips, cross-dock operations)
- Time-phased dimensional modeling (15-minute granularity)
- Real-time Power BI dashboards with predictive bottleneck detection
- Warehouse Gravity Zones™ for spatial optimization
- Adaptive fleet triage engine
- Cross-fact KPI harmonization

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems: WMS, TMS, telemetry APIs, ERP

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance and run the schema script
-- Execute in SSMS or Azure Data Studio
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_bridges.sql
:r schema/05_create_views.sql
:r schema/06_create_stored_procedures.sql
```

3. **Configure connections:**
```json
// config.json (copy from config_sample.json)
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}"
  },
  "data_sources": {
    "wms_connection_string": "${WMS_CONNECTION_STRING}",
    "fleet_api_endpoint": "${FLEET_TELEMETRY_API}",
    "erp_connection_string": "${ERP_CONNECTION_STRING}"
  },
  "refresh_interval_minutes": 15
}
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Tracks warehouse micro-operations:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    GravityZoneAssignment INT,
    UnitsProcessed INT,
    OperatorID INT,
    CONSTRAINT FK_FactWarehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWarehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_FactWarehouse_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimGeography(GeographyKey)
);

-- Optimized indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey) INCLUDE (DurationMinutes, DwellTimeHours);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product_Time ON FactWarehouseOperations(ProductKey, TimeKey);
CREATE COLUMNSTORE INDEX CX_FactWarehouse ON FactWarehouseOperations;
```

**FactFleetTrips** - Captures fleet telemetry and route performance:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripDurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumptionLiters DECIMAL(10,3),
    LoadWeightKg DECIMAL(12,2),
    OnTimeDelivery BIT,
    MaintenanceScore INT, -- 1-100 proactive health score
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FactFleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey) INCLUDE (IdleTimeMinutes, FuelConsumptionLiters);
CREATE COLUMNSTORE INDEX CX_FactFleet ON FactFleetTrips;
```

**FactCrossDock** - Tracks direct transfers without long-term storage:
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    DockDurationMinutes DECIMAL(10,2),
    UnitsTransferred INT,
    CONSTRAINT FK_FactCrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_FactCrossDock_Inbound FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_FactCrossDock_Outbound FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey)
);
```

### Key Dimension Tables

**DimTime** - 15-minute grain time dimension:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT NOT NULL, -- YYYYMMDD format
    TimeOfDay TIME,
    HourOfDay INT,
    QuarterHour INT, -- 0-3 (0, 15, 30, 45 minutes)
    DayOfWeek INT,
    DayName VARCHAR(20),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    FiscalYear INT
);

-- Populate with 15-minute increments
DECLARE @StartDate DATETIME = '2025-01-01';
DECLARE @EndDate DATETIME = '2027-12-31';
DECLARE @CurrentDateTime DATETIME = @StartDate;

WHILE @CurrentDateTime <= @EndDate
BEGIN
    INSERT INTO DimTime (TimeKey, FullDateTime, DateKey, TimeOfDay, HourOfDay, QuarterHour, DayOfWeek, DayName, IsWeekend)
    VALUES (
        CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMddHHmm')),
        @CurrentDateTime,
        CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMdd')),
        CAST(@CurrentDateTime AS TIME),
        DATEPART(HOUR, @CurrentDateTime),
        DATEPART(MINUTE, @CurrentDateTime) / 15,
        DATEPART(WEEKDAY, @CurrentDateTime),
        DATENAME(WEEKDAY, @CurrentDateTime),
        CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
    );
    SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
END;
```

**DimProductGravity** - Warehouse gravity zone assignments:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(255),
    CategoryHierarchy VARCHAR(500), -- Level1/Level2/Level3
    GravityScore INT, -- 1-100, higher = faster moving
    VelocityClass VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueClass VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityScore INT, -- 1-10
    OptimalZoneAssignment INT, -- Zone ID in warehouse
    ReplenishmentLeadTimeDays INT,
    ValidFrom DATETIME,
    ValidTo DATETIME,
    IsCurrent BIT
);

-- Calculate gravity score
CREATE PROCEDURE usp_CalculateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        (PickFrequency / NULLIF(MaxPickFrequency, 0) * 40) +
        (UnitValue / NULLIF(MaxUnitValue, 0) * 30) +
        (10 - FragilityScore) * 3
    )
    FROM DimProductGravity p
    CROSS JOIN (
        SELECT MAX(PickFrequency) AS MaxPickFrequency, MAX(UnitValue) AS MaxUnitValue
        FROM ProductMetrics
    ) m;
END;
```

## Core Analytical Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Link warehouse dwell time to fleet idle costs for same products
WITH WarehouseDwell AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        AVG(w.DwellTimeHours) AS AvgDwellHours,
        AVG(w.DurationMinutes) AS AvgPickTimeMinutes
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateKey >= FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd')
        AND w.OperationType = 'Picking'
    GROUP BY p.ProductKey, p.SKU
),
FleetIdle AS (
    SELECT 
        p.ProductKey,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        AVG(f.FuelConsumptionLiters) AS AvgFuelLiters,
        SUM(f.IdleTimeMinutes * 0.5) AS TotalIdleCostUSD -- $0.50 per idle minute
    FROM FactFleetTrips f
    JOIN BridgeShipmentProduct bsp ON f.TripKey = bsp.TripKey
    JOIN DimProductGravity p ON bsp.ProductKey = p.ProductKey
    JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.DateKey >= FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd')
    GROUP BY p.ProductKey
)
SELECT 
    wd.SKU,
    wd.AvgDwellHours,
    wd.AvgPickTimeMinutes,
    fi.AvgIdleMinutes,
    fi.TotalIdleCostUSD,
    -- Correlation: longer dwell = more rushed loading = more idle time
    CASE 
        WHEN wd.AvgDwellHours > 72 AND fi.AvgIdleMinutes > 20 THEN 'High Risk'
        WHEN wd.AvgDwellHours > 48 AND fi.AvgIdleMinutes > 15 THEN 'Medium Risk'
        ELSE 'Normal'
    END AS RiskLevel
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.ProductKey = fi.ProductKey
ORDER BY fi.TotalIdleCostUSD DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong gravity zones
WITH CurrentPerformance AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.GravityScore,
        p.OptimalZoneAssignment AS CurrentZone,
        AVG(w.DurationMinutes) AS ActualPickTimeMinutes,
        COUNT(*) AS PickCount
    FROM DimProductGravity p
    JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateKey >= FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd')
        AND w.OperationType = 'Picking'
    GROUP BY p.ProductKey, p.SKU, p.GravityScore, p.OptimalZoneAssignment
),
BenchmarkPerformance AS (
    SELECT 
        p.OptimalZoneAssignment AS Zone,
        AVG(w.DurationMinutes) AS BenchmarkPickTime
    FROM DimProductGravity p
    JOIN FactWarehouseOperations w ON p.ProductKey = w.ProductKey
    JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.DateKey >= FORMAT(DATEADD(DAY, -90, GETDATE()), 'yyyyMMdd')
        AND w.OperationType = 'Picking'
    GROUP BY p.OptimalZoneAssignment
)
SELECT 
    cp.SKU,
    cp.GravityScore,
    cp.CurrentZone,
    cp.ActualPickTimeMinutes,
    bp.BenchmarkPickTime,
    cp.ActualPickTimeMinutes - bp.BenchmarkPickTime AS PerformanceDelta,
    CASE 
        WHEN cp.GravityScore > 70 AND cp.CurrentZone > 3 THEN 'Move to Zone 1-3'
        WHEN cp.GravityScore < 30 AND cp.CurrentZone <= 3 THEN 'Move to Zone 4+'
        ELSE 'Optimal'
    END AS Recommendation
FROM CurrentPerformance cp
JOIN BenchmarkPerformance bp ON cp.CurrentZone = bp.Zone
WHERE ABS(cp.ActualPickTimeMinutes - bp.BenchmarkPickTime) > 2 -- More than 2 minutes off benchmark
ORDER BY ABS(cp.ActualPickTimeMinutes - bp.BenchmarkPickTime) DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify probable congestion points using temporal patterns
CREATE PROCEDURE usp_PredictBottlenecks
    @ForecastHours INT = 4
AS
BEGIN
    WITH HistoricalPattern AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            w.WarehouseKey,
            AVG(w.DurationMinutes) AS AvgOperationTime,
            STDEV(w.DurationMinutes) AS StdDevOperationTime,
            COUNT(*) AS OperationCount
        FROM FactWarehouseOperations w
        JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.DateKey >= FORMAT(DATEADD(DAY, -60, GETDATE()), 'yyyyMMdd')
        GROUP BY t.HourOfDay, t.DayOfWeek, w.WarehouseKey
    ),
    CurrentLoad AS (
        SELECT 
            w.WarehouseKey,
            COUNT(*) AS ActiveOperations,
            AVG(w.DurationMinutes) AS CurrentAvgTime
        FROM FactWarehouseOperations w
        JOIN DimTime t ON w.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY w.WarehouseKey
    )
    SELECT 
        wh.WarehouseName,
        DATEPART(HOUR, GETDATE()) AS CurrentHour,
        hp.AvgOperationTime AS ExpectedTimeMinutes,
        cl.CurrentAvgTime AS ActualTimeMinutes,
        cl.ActiveOperations,
        -- Bottleneck score: higher = more congestion
        CASE 
            WHEN cl.CurrentAvgTime > (hp.AvgOperationTime + 2 * hp.StdDevOperationTime) 
            THEN 90 + (cl.CurrentAvgTime - hp.AvgOperationTime)
            WHEN cl.CurrentAvgTime > (hp.AvgOperationTime + hp.StdDevOperationTime)
            THEN 70
            ELSE 30
        END AS BottleneckScore,
        CASE 
            WHEN cl.CurrentAvgTime > (hp.AvgOperationTime + 2 * hp.StdDevOperationTime) 
            THEN 'Critical - Immediate Action Required'
            WHEN cl.CurrentAvgTime > (hp.AvgOperationTime + hp.StdDevOperationTime)
            THEN 'Warning - Monitor Closely'
            ELSE 'Normal Operations'
        END AS AlertLevel
    FROM CurrentLoad cl
    JOIN HistoricalPattern hp ON cl.WarehouseKey = hp.WarehouseKey
        AND hp.HourOfDay = DATEPART(HOUR, GETDATE())
        AND hp.DayOfWeek = DATEPART(WEEKDAY, GETDATE())
    JOIN DimGeography wh ON cl.WarehouseKey = wh.GeographyKey
    ORDER BY BottleneckScore DESC;
END;
```

## Power BI Integration

### Loading the Template

1. **Open Power BI template:**
```powershell
# Navigate to repository directory
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
Start-Process "LogiFleet_Pulse_Master.pbit"
```

2. **Configure data source connection:**
```m
// Power Query M code for SQL Server connection
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER_HOST"), 
        "LogiFleetPulse",
        [
            Query = "SELECT * FROM vw_UnifiedLogisticsMetrics",
            CommandTimeout = #duration(0, 0, 10, 0)
        ]
    )
in
    Source
```

3. **Set up scheduled refresh (Power BI Service):**
```json
{
  "refreshSchedule": {
    "enabled": true,
    "frequency": "15Minutes",
    "timeZone": "UTC",
    "notifyOnFailure": true,
    "notifyEmail": "${ADMIN_EMAIL}"
  }
}
```

### Key DAX Measures

**Cross-Fact KPI: Fleet Efficiency Score**
```dax
Fleet Efficiency Score = 
VAR OnTimePercentage = 
    DIVIDE(
        CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeDelivery] = TRUE),
        COUNTROWS(FactFleetTrips)
    )
VAR IdleTimeRatio = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[TripDurationMinutes])
    )
VAR FuelEfficiency = 
    DIVIDE(
        SUM(FactFleetTrips[LoadWeightKg]),
        SUM(FactFleetTrips[FuelConsumptionLiters])
    )
RETURN
    (OnTimePercentage * 40) + 
    ((1 - IdleTimeRatio) * 30) + 
    (FuelEfficiency / 100 * 30)
```

**Warehouse Gravity Zone Adherence**
```dax
Gravity Zone Adherence % = 
VAR OptimalPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[GravityZoneAssignment] = 
            RELATED(DimProductGravity[OptimalZoneAssignment])
    )
VAR TotalPicks = COUNTROWS(FactWarehouseOperations)
RETURN
    DIVIDE(OptimalPicks, TotalPicks, 0) * 100
```

**Dynamic Time Intelligence: Rolling 30-Day Average**
```dax
Dwell Time - 30 Day Avg = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        MAX(DimTime[FullDateTime]),
        -30,
        DAY
    )
)
```

## Data Loading Patterns

### Incremental Load with Change Tracking

```sql
-- Enable change tracking on fact tables
ALTER DATABASE LogiFleetPulse SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 7 DAYS, AUTO_CLEANUP = ON);

ALTER TABLE FactWarehouseOperations 
ENABLE CHANGE_TRACKING WITH (TRACK_COLUMNS_UPDATED = OFF);

-- Incremental load stored procedure
CREATE PROCEDURE usp_IncrementalLoadWarehouseOps
    @LastLoadVersion BIGINT OUTPUT
AS
BEGIN
    DECLARE @CurrentVersion BIGINT = CHANGE_TRACKING_CURRENT_VERSION();
    
    -- Insert new and updated records
    MERGE FactWarehouseOperations AS target
    USING (
        SELECT 
            ops.OperationID,
            t.TimeKey,
            p.ProductKey,
            wh.GeographyKey AS WarehouseKey,
            ops.OperationType,
            ops.DurationMinutes,
            ops.DwellTimeHours,
            ops.GravityZoneAssignment,
            ops.UnitsProcessed,
            ops.OperatorID
        FROM StagingWarehouseOperations ops
        JOIN DimTime t ON CAST(ops.OperationTimestamp AS DATETIME) = t.FullDateTime
        JOIN DimProductGravity p ON ops.SKU = p.SKU AND p.IsCurrent = 1
        JOIN DimGeography wh ON ops.WarehouseCode = wh.LocationCode
        JOIN CHANGETABLE(CHANGES StagingWarehouseOperations, @LastLoadVersion) ct
            ON ops.OperationID = ct.OperationID
    ) AS source
    ON target.OperationKey = source.OperationID
    WHEN MATCHED THEN UPDATE SET
        target.DurationMinutes = source.DurationMinutes,
        target.DwellTimeHours = source.DwellTimeHours
    WHEN NOT MATCHED THEN INSERT VALUES (
        source.TimeKey,
        source.ProductKey,
        source.WarehouseKey,
        source.OperationType,
        source.DurationMinutes,
        source.DwellTimeHours,
        source.GravityZoneAssignment,
        source.UnitsProcessed,
        source.OperatorID
    );
    
    SET @LastLoadVersion = @CurrentVersion;
END;
```

### External API Integration (Fleet Telemetry)

```sql
-- Create external data source for REST API
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = REST_API,
    LOCATION = '${FLEET_TELEMETRY_API}',
    CREDENTIAL = FleetAPICredential
);

-- Stored procedure to ingest fleet data
CREATE PROCEDURE usp_IngestFleetTelemetry
AS
BEGIN
    DECLARE @json NVARCHAR(MAX);
    
    -- Call external API (using SQL Server 2022+ OPENROWSET with REST)
    SELECT @json = BulkColumn
    FROM OPENROWSET(
        BULK '/api/v1/trips/recent',
        DATA_SOURCE = 'FleetTelemetryAPI',
        SINGLE_CLOB
    ) AS j;
    
    -- Parse JSON and insert into staging
    INSERT INTO StagingFleetTrips (TripID, VehicleID, StartTime, EndTime, IdleMinutes, FuelLiters)
    SELECT 
        TripID,
        VehicleID,
        StartTime,
        EndTime,
        IdleTimeMinutes,
        FuelConsumptionLiters
    FROM OPENJSON(@json, '$.trips')
    WITH (
        TripID INT '$.trip_id',
        VehicleID VARCHAR(50) '$.vehicle_id',
        StartTime DATETIME '$.start_time',
        EndTime DATETIME '$.end_time',
        IdleTimeMinutes DECIMAL(10,2) '$.idle_time_minutes',
        FuelConsumptionLiters DECIMAL(10,3) '$.fuel_consumption_liters'
    );
END;
```

## Configuration Management

### Role-Based Access Control

```sql
-- Create security roles
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY,
    RoleName VARCHAR(50) NOT NULL UNIQUE,
    CanViewWarehouse BIT DEFAULT 0,
    CanViewFleet BIT DEFAULT 0,
    CanViewFinancials BIT DEFAULT 0,
    CanEditConfiguration BIT DEFAULT 0
);

INSERT INTO SecurityRoles (RoleName, CanViewWarehouse, CanViewFleet, CanViewFinancials, CanEditConfiguration)
VALUES 
    ('Operator', 1, 0, 0, 0),
    ('Supervisor', 1, 1, 0, 0),
    ('Executive', 1, 1, 1, 1);

-- Row-level security function
CREATE FUNCTION fn_WarehouseSecurityPredicate(@UserRole VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN 
    SELECT 1 AS AccessGranted
    FROM dbo.FactWarehouseOperations fwo
    JOIN dbo.DimGeography g ON fwo.WarehouseKey = g.GeographyKey
    WHERE g.Region IN (
        SELECT AllowedRegion 
        FROM dbo.UserRegionAccess 
        WHERE UserName = USER_NAME() AND RoleName = @UserRole
    );
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseAccessPolicy
ADD FILTER PREDICATE dbo.fn_WarehouseSecurityPredicate('Operator') ON dbo.FactWarehouseOperations,
ADD BLOCK PREDICATE dbo.fn_WarehouseSecurityPredicate('Operator') ON dbo.FactWarehouseOperations;
```

### Automated Alerting

```sql
-- Alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT PRIMARY KEY IDENTITY,
    AlertName VARCHAR(100) NOT NULL,
    MetricType VARCHAR(50), -- 'DwellTime', 'IdleTime', 'Bottleneck'
    ThresholdValue DECIMAL(10,2),
    ComparisonOperator VARCHAR(10), -- '>', '<', '='
    NotificationMethod VARCHAR(50), -- 'Email', 'SMS', 'Teams'
    RecipientList VARCHAR(MAX),
    IsActive BIT DEFAULT 1
);

-- Alert evaluation and dispatch
CREATE PROCEDURE usp_EvaluateAlerts
AS
BEGIN
    DECLARE @AlertID INT, @MetricType VARCHAR(50), @Threshold DECIMAL(10,2), 
            @Operator VARCHAR(10), @Recipients VARCHAR(MAX);
    
    DECLARE alert_cursor CURSOR FOR
    SELECT AlertID, MetricType, ThresholdValue, ComparisonOperator, RecipientList
    FROM AlertConfiguration WHERE IsActive = 1;
    
    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @Threshold, @Operator, @Recipients;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        DECLARE @CurrentValue DECIMAL(10,2);
        
        -- Evaluate metric
        IF @MetricType = 'DwellTime'
        BEGIN
            SELECT @CurrentValue = AVG(DwellTimeHours)
            FROM FactWarehouseOperations fwo
            JOIN DimTime t ON fwo.TimeKey = t.TimeKey
            WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE());
        END
        ELSE IF @MetricType = 'FleetIdle'
        BEGIN
            SELECT @CurrentValue = AVG(IdleTimeMinutes) / AVG(TripDurationMinutes) * 100
            FROM FactFleetTrips fft
            JOIN DimTime t ON fft.TimeKey = t.TimeKey
            WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE());
        END
        
        -- Check threshold breach
        DECLARE @Breached BIT = 0;
        IF (@Operator = '>' AND @CurrentValue > @Threshold) SET @Breached = 1;
        IF (@Operator = '<' AND @CurrentValue < @Threshold) SET @Breached = 1;
        
        -- Send notification if breached
        IF @Breached = 1
        BEGIN
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = 'LogiFleet Alerts',
                @recipients = @Recipients,
                @subject = 'LogiFleet Alert: ' + @MetricType + ' Threshold Breached',
                @body = 'Current value: ' + CAST(@CurrentValue AS VARCHAR(20)) + 
                        ' | Threshold: ' + CAST(@Threshold AS VARCHAR(20));
                        
            -- Log alert
            INSERT INTO AlertLog (AlertID, TriggerTime, MetricValue, Recipients)
            VALUES (@AlertID, GETDATE(), @CurrentValue, @Recipients);
        END
        
        FETCH NEXT FROM alert_cursor INTO @AlertID, @MetricType, @Threshold, @Operator, @Recipients;
    END
    
    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
```

## Common Troubleshooting

### Performance Optimization

**Issue: Slow cross-fact queries**
```sql
-- Verify columnstore indexes exist
SELECT 
    OBJECT_NAME(object_id) AS TableName,
    name AS IndexName,
    type_desc
FROM sys.indexes
WHERE type_desc = 'CLUSTERED COLUMNSTORE'
    AND OBJECT_NAME(object_id) IN ('FactWarehouseOperations', 'FactFleetTrips', 'FactCrossDock');

-- Rebuild if fragmented
ALTER INDEX CX_FactWarehouse ON FactWarehouseOperations REBUILD;

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
```

**Issue: Power BI refresh timeouts**
```m
// Increase timeout in Power Query
=
