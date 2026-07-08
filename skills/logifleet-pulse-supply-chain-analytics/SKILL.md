---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "build multi-fact star schema for warehouse operations"
  - "integrate fleet telemetry with warehouse data"
  - "create supply chain KPI dashboard"
  - "deploy LogiCore analytics data warehouse"
  - "troubleshoot Power BI logistics model"
  - "optimize warehouse gravity zone queries"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced MS SQL Server data warehouse and Power BI visualization platform for logistics intelligence. It integrates warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer using a multi-fact star schema architecture.

**Core Capabilities:**
- Multi-fact data warehouse (warehouse operations, fleet trips, cross-dock transfers)
- Time-phased dimensional modeling with 15-minute granularity
- Power BI dashboards with cross-fact KPI harmonization
- Predictive bottleneck detection and fleet maintenance prioritization
- Warehouse gravity zone optimization
- Role-based access control and row-level security

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telematics APIs

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance
-- Execute the main schema script
:r schema\00_Create_Database.sql
:r schema\01_Create_Dimensions.sql
:r schema\02_Create_Facts.sql
:r schema\03_Create_Views.sql
:r schema\04_Create_Procedures.sql
:r schema\05_Create_Indexes.sql
```

3. **Configure data source connections:**
```json
{
  "connections": {
    "wms_api": "${WMS_CONNECTION_STRING}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "erp_system": "${ERP_CONNECTION_STRING}",
    "weather_api": "${WEATHER_API_ENDPOINT}"
  },
  "refresh_interval_minutes": 15,
  "security": {
    "enable_rls": true,
    "ad_group_mapping": "${AD_GROUP_JSON_PATH}"
  }
}
```

4. **Import Power BI template:**
```powershell
# Open the template file
Start-Process "LogiFleet_Pulse_Master.pbit"
# When prompted, enter your SQL Server connection details
```

## Core Data Model

### Key Tables

**Fact Tables:**
- `FactWarehouseOperations` - Receiving, putaway, picking, packing, shipping events
- `FactFleetTrips` - Route segments, fuel consumption, idle time
- `FactCrossDock` - Direct transfers between inbound/outbound
- `FactInventorySnapshot` - Point-in-time inventory levels

**Dimension Tables:**
- `DimTime` - 15-minute granularity with fiscal calendar
- `DimGeography` - Hierarchical location data
- `DimProduct` - Product hierarchy with gravity scores
- `DimVehicle` - Fleet assets with maintenance history
- `DimSupplier` - Supplier reliability metrics
- `DimWarehouse` - Storage zones and capacity

### Sample Schema Creation

```sql
-- Create time dimension (15-minute grain)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeValue DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT NOT NULL,
    QuarterHour INT NOT NULL,
    DayOfWeek INT NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthOfYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Create warehouse operations fact
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    Quantity INT NOT NULL,
    DwellTimeMinutes INT NULL,
    OperatorID INT NOT NULL,
    SourceZone VARCHAR(20) NULL,
    DestinationZone VARCHAR(20) NULL,
    ProcessingTimeMinutes INT NOT NULL,
    ErrorCount INT DEFAULT 0,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WarehouseOps_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Create fleet trips fact
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceMiles DECIMAL(10,2) NOT NULL,
    FuelConsumedGallons DECIMAL(8,2) NOT NULL,
    IdleTimeMinutes INT NOT NULL,
    LoadWeightLbs INT NOT NULL,
    DeliveryOnTime BIT NOT NULL,
    DelayReasonKey INT NULL,
    TotalCost DECIMAL(10,2) NOT NULL,
    CONSTRAINT FK_FleetTrip_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrip_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrip_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Load warehouse operations incrementally
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimeKey INT,
    @CurrentLoadTimeKey INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, OperationType,
        Quantity, DwellTimeMinutes, OperatorID, 
        SourceZone, DestinationZone, ProcessingTimeMinutes
    )
    SELECT 
        t.TimeKey,
        p.ProductKey,
        w.WarehouseKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        stg.OperatorID,
        stg.SourceZone,
        stg.DestinationZone,
        stg.ProcessingTimeMinutes
    FROM Staging.WarehouseOperations stg
    INNER JOIN DimTime t ON CAST(stg.OperationTime AS DATETIME) = t.DateTimeValue
    INNER JOIN DimProduct p ON stg.ProductSKU = p.SKU
    INNER JOIN DimWarehouse w ON stg.WarehouseCode = w.WarehouseCode
    WHERE t.TimeKey > @LastLoadTimeKey
        AND t.TimeKey <= @CurrentLoadTimeKey;
    
    -- Update control table
    UPDATE ETL.LoadControl
    SET LastLoadTimeKey = @CurrentLoadTimeKey,
        LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
GO
```

### Cross-Fact KPI Calculation

```sql
-- Calculate composite KPI: Warehouse efficiency impact on fleet utilization
CREATE PROCEDURE sp_CalculateWarehouseFleetKPI
    @StartDateKey INT,
    @EndDateKey INT
AS
BEGIN
    SELECT 
        w.WarehouseName,
        d.FiscalWeek,
        
        -- Warehouse metrics
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        SUM(wo.Quantity) AS TotalPickedUnits,
        SUM(CASE WHEN wo.ErrorCount > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS ErrorRate,
        
        -- Fleet metrics
        AVG(ft.IdleTimeMinutes) AS AvgFleetIdleTime,
        SUM(ft.FuelConsumedGallons) AS TotalFuelConsumption,
        SUM(CASE WHEN ft.DeliveryOnTime = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(ft.TripKey) AS OnTimeDeliveryRate,
        
        -- Cross-fact KPI: Dwell impact score
        (AVG(wo.DwellTimeMinutes) * 0.6 + AVG(ft.IdleTimeMinutes) * 0.4) AS CompositeDelayScore
        
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime d ON wo.TimeKey = d.TimeKey
    INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    LEFT JOIN FactFleetTrips ft ON ft.OriginGeographyKey = w.GeographyKey
        AND ft.StartTimeKey BETWEEN d.TimeKey - 96 AND d.TimeKey + 96 -- +/- 24 hours
    WHERE d.DateKey BETWEEN @StartDateKey AND @EndDateKey
        AND wo.OperationType = 'Shipping'
    GROUP BY w.WarehouseName, d.FiscalWeek
    ORDER BY CompositeDelayScore DESC;
END;
GO
```

### Warehouse Gravity Zone Analysis

```sql
-- Calculate and update product gravity scores
CREATE PROCEDURE sp_UpdateProductGravityScores
    @AnalysisWindowDays INT = 90
AS
BEGIN
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            COUNT(*) AS PickFrequency,
            AVG(wo.ProcessingTimeMinutes) AS AvgPickTime,
            SUM(wo.Quantity) AS TotalQuantityPicked,
            p.UnitValue,
            p.FragilityScore
        FROM FactWarehouseOperations wo
        INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateKey >= DATEADD(DAY, -@AnalysisWindowDays, GETDATE())
            AND wo.OperationType = 'Picking'
        GROUP BY p.ProductKey, p.SKU, p.UnitValue, p.FragilityScore
    )
    UPDATE p
    SET 
        p.GravityScore = (
            (pm.PickFrequency * 0.4) +
            (pm.UnitValue / 100.0 * 0.3) +
            (pm.FragilityScore * 0.2) +
            ((1.0 / NULLIF(pm.AvgPickTime, 0)) * 100 * 0.1)
        ),
        p.RecommendedZone = CASE
            WHEN (pm.PickFrequency * 0.4 + pm.UnitValue / 100.0 * 0.3) > 75 THEN 'HIGH_GRAVITY'
            WHEN (pm.PickFrequency * 0.4 + pm.UnitValue / 100.0 * 0.3) > 40 THEN 'MEDIUM_GRAVITY'
            ELSE 'LOW_GRAVITY'
        END,
        p.LastGravityUpdate = GETDATE()
    FROM DimProduct p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO
```

## Power BI Configuration

### Connection Setup

```powerquery
// In Power BI Desktop, Power Query M code
let
    Source = Sql.Database(
        #"SQL_SERVER_HOST" meta [EncryptConnection = true],
        "LogiFleetPulse",
        [
            Query = "EXEC sp_CalculateWarehouseFleetKPI 
                     @StartDateKey = " & Text.From(StartDateKey) & ",
                     @EndDateKey = " & Text.From(EndDateKey)
        ]
    ),
    #"Changed Type" = Table.TransformColumnTypes(Source,{
        {"AvgDwellTime", type number},
        {"AvgFleetIdleTime", type number},
        {"CompositeDelayScore", type number}
    })
in
    #"Changed Type"
```

### DAX Measures for Cross-Fact Analysis

```dax
// Total Warehouse Throughput
Total Throughput = 
SUM(FactWarehouseOperations[Quantity])

// Fleet Efficiency Ratio
Fleet Efficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceMiles]),
    SUM(FactFleetTrips[IdleTimeMinutes]) / 60,
    0
)

// Cross-Fact: Warehouse Impact on Delivery
Warehouse Delivery Impact = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR OnTimeRate = DIVIDE(
    CALCULATE(COUNT(FactFleetTrips[TripKey]), FactFleetTrips[DeliveryOnTime] = TRUE()),
    COUNT(FactFleetTrips[TripKey])
)
RETURN
    IF(AvgDwell > 120, OnTimeRate * 0.85, OnTimeRate) // Penalty for high dwell time

// Predictive Bottleneck Score
Bottleneck Risk Score = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR HistoricalAvg = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -90, DAY)
)
VAR DwellVariance = DIVIDE(CurrentDwell - HistoricalAvg, HistoricalAvg, 0)
VAR FleetIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN
    (DwellVariance * 0.6 + (FleetIdle / 60) * 0.4) * 100
```

## Common Usage Patterns

### Pattern 1: Daily Operations Dashboard

```sql
-- Query for real-time operations view
SELECT 
    w.WarehouseName,
    t.DateValue AS OperationDate,
    t.HourOfDay,
    wo.OperationType,
    COUNT(*) AS OperationCount,
    AVG(wo.ProcessingTimeMinutes) AS AvgProcessingTime,
    SUM(wo.Quantity) AS TotalQuantity,
    SUM(wo.ErrorCount) AS TotalErrors
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
WHERE t.DateKey = CAST(CONVERT(VARCHAR, GETDATE(), 112) AS INT)
GROUP BY w.WarehouseName, t.DateValue, t.HourOfDay, wo.OperationType
ORDER BY t.HourOfDay DESC;
```

### Pattern 2: Fleet Maintenance Prioritization

```sql
-- Identify vehicles needing maintenance based on operational impact
WITH VehicleMetrics AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.MaintenanceStatusScore,
        COUNT(ft.TripKey) AS TripCount,
        AVG(ft.FuelConsumedGallons / NULLIF(ft.DistanceMiles, 0)) AS AvgFuelEfficiency,
        SUM(CASE WHEN ft.DeliveryOnTime = 0 THEN 1 ELSE 0 END) AS LateDeliveries,
        SUM(ft.LoadWeightLbs * ft.DistanceMiles) AS TotalRevenueMiles
    FROM DimVehicle v
    INNER JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
    INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    WHERE t.DateKey >= CAST(CONVERT(VARCHAR, DATEADD(DAY, -30, GETDATE()), 112) AS INT)
    GROUP BY v.VehicleKey, v.VehicleID, v.MaintenanceStatusScore
)
SELECT 
    VehicleID,
    MaintenanceStatusScore,
    TripCount,
    AvgFuelEfficiency,
    LateDeliveries,
    -- Weighted priority score
    (
        (100 - MaintenanceStatusScore) * 0.4 +
        (LateDeliveries * 10) * 0.3 +
        ((1.0 / NULLIF(AvgFuelEfficiency, 0)) * 10) * 0.3
    ) AS MaintenancePriorityScore,
    TotalRevenueMiles
FROM VehicleMetrics
WHERE MaintenanceStatusScore < 80
ORDER BY MaintenancePriorityScore DESC;
```

### Pattern 3: Gravity Zone Optimization

```sql
-- Recommend product relocations based on gravity analysis
WITH CurrentZones AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.CurrentStorageZone,
        p.RecommendedZone,
        wz.DistanceFromDock AS CurrentDistance,
        COUNT(wo.OperationKey) AS RecentPicks
    FROM DimProduct p
    INNER JOIN DimWarehouse w ON p.WarehouseKey = w.WarehouseKey
    INNER JOIN DimWarehouseZone wz ON p.CurrentStorageZone = wz.ZoneCode
    LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
        AND wo.OperationType = 'Picking'
        AND wo.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime) -- Last 7 days
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, 
             p.CurrentStorageZone, p.RecommendedZone, wz.DistanceFromDock
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    CurrentStorageZone,
    RecommendedZone,
    CurrentDistance,
    RecentPicks,
    (GravityScore * RecentPicks * CurrentDistance) AS RelocationBenefit
FROM CurrentZones
WHERE CurrentStorageZone != RecommendedZone
    AND RecentPicks > 10
ORDER BY RelocationBenefit DESC;
```

## Alerting Configuration

### SQL Server Agent Job for Threshold Alerts

```sql
-- Create alert stored procedure
CREATE PROCEDURE sp_MonitorLogisticsAlerts
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @Recipients NVARCHAR(500) = '${ALERT_EMAIL_RECIPIENTS}';
    
    -- Check 1: High dwell time alert
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations wo
        INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateKey = CAST(CONVERT(VARCHAR, GETDATE(), 112) AS INT)
            AND wo.DwellTimeMinutes > 240
        GROUP BY wo.WarehouseKey
        HAVING COUNT(*) > 5
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Multiple items with dwell time > 4 hours detected';
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @Recipients,
            @subject = 'LogiFleet: High Dwell Time Alert',
            @body = @AlertMessage;
    END
    
    -- Check 2: Fleet idle time exceeds threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips ft
        INNER JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
        WHERE t.DateKey = CAST(CONVERT(VARCHAR, GETDATE(), 112) AS INT)
            AND (ft.IdleTimeMinutes * 100.0 / NULLIF(DATEDIFF(MINUTE, t.DateTimeValue, 
                (SELECT DateTimeValue FROM DimTime WHERE TimeKey = ft.EndTimeKey)), 0)) > 15
        GROUP BY ft.VehicleKey
        HAVING COUNT(*) > 3
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet vehicles showing excessive idle time (>15% of trip duration)';
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = @Recipients,
            @subject = 'LogiFleet: High Fleet Idle Time Alert',
            @body = @AlertMessage;
    END
END;
GO

-- Schedule the job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_Alert_Monitor';
    
EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_Alert_Monitor',
    @step_name = N'Check_Thresholds',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_MonitorLogisticsAlerts';
    
EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every_15_Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15;
```

## Row-Level Security Setup

```sql
-- Create security table
CREATE TABLE Security.UserAccess (
    UserID INT PRIMARY KEY,
    UserEmail NVARCHAR(255) NOT NULL,
    WarehouseAccess NVARCHAR(MAX), -- JSON array of warehouse keys
    FleetAccess NVARCHAR(MAX), -- JSON array of fleet regions
    RoleLevel VARCHAR(20) NOT NULL -- 'Operator', 'Supervisor', 'Executive'
);

-- Create security predicate function
CREATE FUNCTION Security.fn_WarehouseAccessPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    FROM Security.UserAccess
    WHERE UserEmail = USER_NAME()
        AND (
            RoleLevel = 'Executive'
            OR @WarehouseKey IN (
                SELECT value FROM OPENJSON(WarehouseAccess)
            )
        )
);
GO

-- Apply security policy
CREATE SECURITY POLICY Security.WarehouseAccessPolicy
ADD FILTER PREDICATE Security.fn_WarehouseAccessPredicate(WarehouseKey)
ON FactWarehouseOperations
WITH (STATE = ON);
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Solution:** Add composite indexes on join keys and filter columns.

```sql
-- Index for time-based joins
CREATE INDEX IX_WarehouseOps_Time_Warehouse 
ON FactWarehouseOperations(TimeKey, WarehouseKey)
INCLUDE (Quantity, DwellTimeMinutes);

CREATE INDEX IX_FleetTrips_StartTime_Vehicle
ON FactFleetTrips(StartTimeKey, VehicleKey)
INCLUDE (IdleTimeMinutes, FuelConsumedGallons);

-- Index for geography-based joins
CREATE INDEX IX_FleetTrips_Origin_Destination
ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey)
INCLUDE (StartTimeKey, DeliveryOnTime);
```

### Issue: Power BI Refresh Timeout

**Solution:** Implement incremental refresh with partitioning.

```powerquery
// In Power BI, set up incremental refresh parameters
let
    RangeStart = #datetime(2024, 1, 1, 0, 0, 0) meta [IsParameterQuery=true, Type="DateTime"],
    RangeEnd = #datetime(2026, 12, 31, 23, 59, 59) meta [IsParameterQuery=true, Type="DateTime"],
    
    Source = Sql.Database("YOUR_SERVER", "LogiFleetPulse"),
    FactTable = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FilteredRows = Table.SelectRows(FactTable, each 
        [OperationDateTime] >= RangeStart and [OperationDateTime] < RangeEnd
    )
in
    FilteredRows
```

### Issue: Gravity Score Not Updating

**Diagnostic query:**

```sql
-- Check when gravity scores were last updated
SELECT 
    COUNT(*) AS TotalProducts,
    COUNT(CASE WHEN LastGravityUpdate >= DATEADD(DAY, -1, GETDATE()) THEN 1 END) AS UpdatedLast24h,
    MIN(LastGravityUpdate) AS OldestUpdate,
    MAX(LastGravityUpdate) AS NewestUpdate
FROM DimProduct;

-- Check for products with no recent picking activity
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.LastGravityUpdate,
    COUNT(wo.OperationKey) AS RecentPickCount
FROM DimProduct p
LEFT JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
    AND wo.OperationType = 'Picking'
    AND wo.TimeKey >= (SELECT MAX(TimeKey) - 8640 FROM DimTime) -- Last 90 days
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.LastGravityUpdate
HAVING COUNT(wo.OperationKey) = 0
    AND p.LastGravityUpdate < DATEADD(DAY, -7, GETDATE());
```

**Solution:** Schedule the gravity update procedure more frequently or investigate data pipeline issues.

```sql
-- Force manual gravity update
EXEC sp_UpdateProductGravityScores @AnalysisWindowDays = 90;
```

## Performance Optimization

### Columnstore Indexes for Large Fact Tables

```sql
-- Create columnstore index for analytical queries
CREATE COLUMNSTORE INDEX CCI_FactWarehouseOperations
ON FactWarehouseOperations (
    TimeKey, ProductKey, WarehouseKey, OperationType,
    Quantity, DwellTimeMinutes, ProcessingTimeMinutes
);

CREATE COLUMNSTORE INDEX CCI_FactFleetTrips
ON FactFleetTrips (
    StartTimeKey, VehicleKey, RouteKey,
    DistanceMiles, FuelConsumedGallons, IdleTimeMinutes
);
```

### Partitioning Strategy

```sql
-- Partition fact tables by month
CREATE PARTITION FUNCTION PF_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601,
    20260701, 20260801, 20260901, 20261001, 20261101, 20261201
);

CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly ALL TO ([PRIMARY]);

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- other columns...
    DateKey INT NOT NULL,
    CONSTRAINT PK_WarehouseOps_Part PRIMARY KEY (DateKey, OperationKey)
) ON PS_Monthly(DateKey);
```

## Environment Variables Reference

```bash
# SQL Server Connection
SQL_SERVER_HOST="your-server.database.windows.net"
SQL_SERVER_DATABASE="LogiFleetPulse"
SQL_SERVER_USER="${SQL_SERVER_USER}"
SQL_SERVER_PASSWORD="${SQL_SERVER_PASSWORD}"

# Data Source APIs
WMS_CONNECTION_STRING="${WMS_CONNECTION_STRING}"
FLEET_API_ENDPOINT="${FLEET_API_ENDPOINT}"
FLEET_API_KEY="${FLEET_API_KEY}"
ERP_CONNECTION_STRING="${ERP_CONNECTION_STRING}"
WEATHER_API_ENDPOINT="${WEATHER_API_ENDPOINT}"
WEATHER_API_KEY="${WEATHER_API_KEY}"

# Alerting
ALERT_EMAIL_RECIPIENTS="logistics-team@company.com"
SMTP_SERVER="${SMTP_SERVER}"
SMTP_PORT="587"

# Power BI Service
POWERBI_WORKSPACE_ID="${POWERBI_WORKSPACE_ID}"
POWERBI_SERVICE_PRINCIPAL="${POWERBI_SERVICE_PRINCIPAL}"
POWERBI_CLIENT_SECRET="${POWERBI_CLIENT_SECRET}"

# Active Directory (for RLS)
AD_GROUP_JSON_PATH="/config/ad_group_mapping.json"
```

This skill provides comprehensive guidance for deploying and operating the LogiFleet Pulse supply chain analytics platform, covering database schema, ETL processes, Power BI integration, and operational best practices.
