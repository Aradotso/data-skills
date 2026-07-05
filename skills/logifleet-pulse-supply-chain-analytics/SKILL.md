---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics analytics platform for warehouse operations, fleet management, and cross-modal supply chain intelligence
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "configure warehouse and fleet data model"
  - "create power bi logistics dashboard"
  - "implement multi-fact star schema for supply chain"
  - "build warehouse gravity zone optimization"
  - "set up fleet telemetry analytics in sql server"
  - "deploy logifleet pulse kpi tracking"
  - "configure cross-modal logistics intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:

- **Multi-fact star schema** data warehousing in MS SQL Server
- **Power BI dashboards** for real-time warehouse and fleet analytics
- **Cross-modal KPI harmonization** linking warehouse operations with fleet telemetry
- **Predictive analytics** for bottleneck detection and maintenance triage
- **Warehouse Gravity Zones™** spatial optimization based on pick frequency and item value
- **Time-phased dimensions** for temporal elasticity modeling

Primary language: **SQL** (T-SQL for MS SQL Server) with Power BI integration.

## Installation & Prerequisites

### Requirements

- **MS SQL Server 2019+** (Express, Standard, or Enterprise)
- **Power BI Desktop** (latest version)
- **Source Systems**: WMS, TMS, or telemetry APIs (JSON/REST or SQL-accessible)

### Deployment Steps

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema**:
```sql
-- Execute the main schema script on your SQL Server instance
-- This creates the database, tables, relationships, and stored procedures
sqlcmd -S YOUR_SERVER_NAME -d master -i schema/LogiFleet_Schema.sql
```

3. **Configure data source connections**:
```json
{
  "wms_connection": "Server=${WMS_SQL_SERVER};Database=${WMS_DB};User Id=${WMS_USER};Password=${WMS_PASSWORD};",
  "telemetry_api": "${FLEET_TELEMETRY_API_URL}",
  "telemetry_api_key": "${FLEET_API_KEY}",
  "refresh_interval_minutes": 15
}
```

4. **Open Power BI template**:
   - Open `LogiFleet_Pulse_Master.pbit`
   - Connect to your SQL Server instance
   - Configure row-level security based on user roles

## Core Data Model Architecture

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity tracking:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    QuantityHandled INT,
    OperatorKey INT,
    CONSTRAINT FK_WHOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WHOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WHOps_Zone FOREIGN KEY (WarehouseZoneKey) REFERENCES DimWarehouseZone(ZoneKey)
);

-- Recommended indexes for cross-fact queries
CREATE NONCLUSTERED INDEX IX_WHOps_Time_Product ON FactWarehouseOperations(TimeKey, ProductKey) INCLUDE (DwellTimeMinutes, QuantityHandled);
CREATE NONCLUSTERED INDEX IX_WHOps_Zone_OpType ON FactWarehouseOperations(WarehouseZoneKey, OperationType);
```

**FactFleetTrips** - Fleet telemetry and route tracking:
```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKG INT,
    OnTimeDelivery BIT,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_Fleet_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_Fleet_Time_Vehicle ON FactFleetTrips(TimeKey, VehicleKey) INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

**FactCrossDock** - Cross-docking operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    InboundShipmentKey INT NOT NULL,
    OutboundShipmentKey INT NOT NULL,
    ProductKey INT NOT NULL,
    QuantityTransferred INT,
    DwellTimeMinutes INT, -- Time between inbound receipt and outbound dispatch
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfYear INT,
    DayOfMonth INT,
    DayOfWeek INT,
    HourOfDay INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod VARCHAR(20),
    INDEX IX_Time_DateTime NONCLUSTERED (FullDateTime)
);

-- Populate time dimension
DECLARE @StartDate DATETIME2 = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME2 = '2027-12-31 23:45:00';

WITH TimeCTE AS (
    SELECT @StartDate AS TimeStamp
    UNION ALL
    SELECT DATEADD(MINUTE, 15, TimeStamp)
    FROM TimeCTE
    WHERE TimeStamp < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Week, DayOfYear, DayOfMonth, DayOfWeek, HourOfDay, MinuteBucket, IsWeekend)
SELECT 
    CAST(FORMAT(TimeStamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    TimeStamp,
    YEAR(TimeStamp),
    DATEPART(QUARTER, TimeStamp),
    MONTH(TimeStamp),
    DATEPART(WEEK, TimeStamp),
    DATEPART(DAYOFYEAR, TimeStamp),
    DAY(TimeStamp),
    DATEPART(WEEKDAY, TimeStamp),
    DATEPART(HOUR, TimeStamp),
    (DATEPART(MINUTE, TimeStamp) / 15) * 15,
    CASE WHEN DATEPART(WEEKDAY, TimeStamp) IN (1, 7) THEN 1 ELSE 0 END
FROM TimeCTE
OPTION (MAXRECURSION 0);
```

**DimProductGravity** - Product hierarchy with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    CategoryL1 VARCHAR(100),
    CategoryL2 VARCHAR(100),
    UnitValue DECIMAL(10,2),
    Fragility TINYINT, -- 1-10 scale
    PickFrequencyScore DECIMAL(5,2), -- Calculated from historical picks
    ReplenishmentLeadTimeDays INT,
    GravityScore AS (PickFrequencyScore * 0.4 + UnitValue/100 * 0.3 + Fragility * 0.3) PERSISTED, -- Composite score
    RecommendedZoneType VARCHAR(50) -- 'HIGH_GRAVITY', 'MEDIUM_GRAVITY', 'LOW_GRAVITY'
);
```

**DimWarehouseZone** - Spatial warehouse zones:
```sql
CREATE TABLE DimWarehouseZone (
    ZoneKey INT PRIMARY KEY IDENTITY(1,1),
    ZoneCode VARCHAR(20) UNIQUE NOT NULL,
    ZoneName VARCHAR(100),
    WarehouseID VARCHAR(20),
    ZoneType VARCHAR(50), -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING', 'COLD_STORAGE'
    GravityZoneRating VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW' based on proximity to shipping
    CapacityPallets INT,
    CurrentUtilizationPct DECIMAL(5,2)
);
```

## Key Stored Procedures

### Incremental Data Loading

**Load Warehouse Operations**:
```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from external WMS system via linked server or staging table
    INSERT INTO FactWarehouseOperations (TimeKey, ProductKey, WarehouseZoneKey, OperationType, DwellTimeMinutes, QuantityHandled, OperatorKey)
    SELECT 
        CAST(FORMAT(wms.OperationTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        p.ProductKey,
        z.ZoneKey,
        wms.OperationType,
        DATEDIFF(MINUTE, wms.StartTime, wms.EndTime) AS DwellTimeMinutes,
        wms.Quantity,
        o.OperatorKey
    FROM StagingWMS.dbo.Operations wms
    INNER JOIN DimProductGravity p ON wms.SKU = p.SKU
    INNER JOIN DimWarehouseZone z ON wms.ZoneCode = z.ZoneCode
    INNER JOIN DimOperator o ON wms.OperatorID = o.OperatorID
    WHERE wms.OperationTimestamp > @LastLoadTimestamp;
    
    -- Update last load timestamp
    UPDATE ETL_ControlTable SET LastLoadTime = GETDATE() WHERE TableName = 'FactWarehouseOperations';
END;
```

**Load Fleet Trips**:
```sql
CREATE PROCEDURE sp_LoadFleetTrips
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (TimeKey, VehicleKey, RouteKey, DriverKey, TripStartTime, TripEndTime, DistanceKM, FuelConsumedLiters, IdleTimeMinutes, LoadWeightKG, OnTimeDelivery)
    SELECT 
        CAST(FORMAT(t.TripStartTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        v.VehicleKey,
        r.RouteKey,
        d.DriverKey,
        t.TripStartTime,
        t.TripEndTime,
        t.DistanceKM,
        t.FuelLiters,
        t.IdleMinutes,
        t.LoadWeight,
        CASE WHEN t.ActualDeliveryTime <= t.ScheduledDeliveryTime THEN 1 ELSE 0 END
    FROM StagingTelemetry.dbo.Trips t
    INNER JOIN DimVehicle v ON t.VehicleID = v.VehicleID
    INNER JOIN DimRoute r ON t.RouteID = r.RouteID
    INNER JOIN DimDriver d ON t.DriverID = d.DriverID
    WHERE t.TripStartTime > @LastLoadTimestamp;
    
    UPDATE ETL_ControlTable SET LastLoadTime = GETDATE() WHERE TableName = 'FactFleetTrips';
END;
```

### Analytical Queries

**Cross-Fact KPI: Dwell Time vs Fleet Idling**:
```sql
-- Find products with high warehouse dwell time that correlate with fleet idle time
WITH WarehouseDwell AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeMinutes) AS AvgDwellMinutes,
        t.DayOfWeek
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY p.SKU, p.ProductName, t.DayOfWeek
    HAVING AVG(w.DwellTimeMinutes) > 120 -- More than 2 hours
),
FleetIdle AS (
    SELECT 
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes,
        t.DayOfWeek
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.DayOfWeek
)
SELECT 
    wd.SKU,
    wd.ProductName,
    wd.DayOfWeek,
    wd.AvgDwellMinutes,
    fi.AvgIdleMinutes,
    (wd.AvgDwellMinutes * fi.AvgIdleMinutes) AS CompositeFrictionScore
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.DayOfWeek = fi.DayOfWeek
ORDER BY CompositeFrictionScore DESC;
```

**Warehouse Gravity Zone Optimization**:
```sql
-- Identify products in wrong gravity zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.RecommendedZoneType,
    z.ZoneType AS CurrentZoneType,
    z.ZoneName,
    COUNT(w.OperationID) AS PickOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime
FROM FactWarehouseOperations w
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
INNER JOIN DimWarehouseZone z ON w.WarehouseZoneKey = z.ZoneKey
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE 
    w.OperationType = 'PICK'
    AND t.FullDateTime >= DATEADD(MONTH, -1, GETDATE())
    AND p.RecommendedZoneType != z.GravityZoneRating
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.RecommendedZoneType, z.ZoneType, z.ZoneName
HAVING COUNT(w.OperationID) > 50 -- Frequent picks
ORDER BY p.GravityScore DESC;
```

**Predictive Fleet Maintenance Triage**:
```sql
-- Score vehicles by maintenance urgency weighted by revenue impact
WITH VehicleHealth AS (
    SELECT 
        v.VehicleKey,
        v.VehicleID,
        v.TirePressurePSI,
        v.EngineDiagnosticCode,
        v.LastMaintenanceDate,
        DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceService
    FROM DimVehicle v
),
RevenueImpact AS (
    SELECT 
        f.VehicleKey,
        SUM(l.LoadValue) AS TotalLoadValue,
        AVG(f.OnTimeDelivery) AS OnTimeRate
    FROM FactFleetTrips f
    INNER JOIN FactShipmentLoads l ON f.TripID = l.TripID
    WHERE f.TripStartTime >= DATEADD(MONTH, -1, GETDATE())
    GROUP BY f.VehicleKey
)
SELECT 
    vh.VehicleID,
    vh.DaysSinceService,
    vh.TirePressurePSI,
    vh.EngineDiagnosticCode,
    ri.TotalLoadValue,
    ri.OnTimeRate,
    -- Urgency score: service overdue + low tire pressure + high load value
    (CASE WHEN vh.DaysSinceService > 90 THEN 50 ELSE 0 END +
     CASE WHEN vh.TirePressurePSI < 30 THEN 30 ELSE 0 END +
     CASE WHEN vh.EngineDiagnosticCode IS NOT NULL THEN 40 ELSE 0 END) * 
    (ri.TotalLoadValue / 10000) AS MaintenanceUrgencyScore
FROM VehicleHealth vh
LEFT JOIN RevenueImpact ri ON vh.VehicleKey = ri.VehicleKey
ORDER BY MaintenanceUrgencyScore DESC;
```

## Power BI Integration

### Connection Setup

Connect Power BI to SQL Server using DirectQuery for real-time dashboards:

```powerquery
// In Power BI Advanced Editor
let
    Source = Sql.Database(
        "${SQL_SERVER_NAME}", 
        "${DATABASE_NAME}",
        [
            Query = "SELECT * FROM vw_LogiFleet_Unified_KPI", // Use pre-built view for performance
            CommandTimeout = #duration(0, 0, 5, 0)
        ]
    )
in
    Source
```

### DAX Measures for Cross-Fact KPIs

**Composite Dwell-to-Idle Ratio**:
```dax
// Measure in Power BI
Dwell_to_Idle_Ratio = 
DIVIDE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    AVERAGE(FactFleetTrips[IdleTimeMinutes]),
    0
)
```

**Weighted Gravity Zone Efficiency**:
```dax
Gravity_Zone_Efficiency = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[QuantityHandled] * 
    RELATED(DimProductGravity[GravityScore]) / 
    RELATED(DimWarehouseZone[CurrentUtilizationPct])
)
```

**Fleet Fuel Efficiency with Load Adjustment**:
```dax
Fuel_Efficiency_Per_Ton = 
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLiters]),
    SUM(FactFleetTrips[LoadWeightKG]) / 1000, // Convert to tons
    BLANK()
)
```

### Row-Level Security

Configure RLS in Power BI based on user roles:

```dax
// Create role "Regional_Manager"
[DimGeography[Region]] = USERNAME()

// Create role "Warehouse_Supervisor"
[DimWarehouse[WarehouseID]] IN 
    LOOKUPVALUE(
        UserWarehouseAccess[WarehouseList],
        UserWarehouseAccess[UserEmail],
        USERPRINCIPALNAME()
    )
```

## Automated Alerting

### SQL Server Agent Job for KPI Breach Detection

```sql
CREATE PROCEDURE sp_AlertCriticalKPIs
AS
BEGIN
    -- Check for excessive fleet idle time
    DECLARE @ExcessiveIdleVehicles TABLE (VehicleID VARCHAR(20), AvgIdleMinutes INT);
    
    INSERT INTO @ExcessiveIdleVehicles
    SELECT 
        v.VehicleID,
        AVG(f.IdleTimeMinutes) AS AvgIdleMinutes
    FROM FactFleetTrips f
    INNER JOIN DimVehicle v ON f.VehicleKey = v.VehicleKey
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -1, GETDATE())
    GROUP BY v.VehicleID
    HAVING AVG(f.IdleTimeMinutes) > (SELECT AVG(IdleTimeMinutes) * 1.5 FROM FactFleetTrips);
    
    IF EXISTS (SELECT 1 FROM @ExcessiveIdleVehicles)
    BEGIN
        DECLARE @AlertBody NVARCHAR(MAX);
        SET @AlertBody = (
            SELECT VehicleID, AvgIdleMinutes
            FROM @ExcessiveIdleVehicles
            FOR JSON PATH
        );
        
        -- Send alert via Database Mail or external webhook
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'ALERT: Excessive Fleet Idle Time Detected',
            @body = @AlertBody,
            @body_format = 'HTML';
    END;
    
    -- Check for warehouse capacity approaching limit
    IF EXISTS (
        SELECT 1 FROM DimWarehouseZone 
        WHERE CurrentUtilizationPct > 95
    )
    BEGIN
        -- Send capacity alert
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'ALERT: Warehouse Zones Near Capacity',
            @body = 'One or more warehouse zones exceed 95% utilization.',
            @body_format = 'TEXT';
    END;
END;

-- Schedule to run every hour
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_KPI_Alerts';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Alerts',
    @step_name = 'Check_KPIs',
    @command = 'EXEC sp_AlertCriticalKPIs';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Hourly',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hours
    @freq_subday_interval = 1;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleet_KPI_Alerts',
    @schedule_name = 'Hourly';
```

## Common Patterns

### Pattern 1: Time-Phased Dimensional Drill-Down

```sql
-- Analyze performance trends across multiple time hierarchies
SELECT 
    t.Year,
    t.Quarter,
    t.Month,
    t.DayOfWeek,
    COUNT(DISTINCT w.OperationID) AS TotalOperations,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    SUM(w.QuantityHandled) AS TotalQuantity
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
WHERE t.FullDateTime >= DATEADD(YEAR, -1, GETDATE())
GROUP BY ROLLUP(t.Year, t.Quarter, t.Month, t.DayOfWeek)
ORDER BY t.Year, t.Quarter, t.Month, t.DayOfWeek;
```

### Pattern 2: Bridge Table for Many-to-Many Relationships

```sql
-- Handle routes that serve multiple warehouse zones
CREATE TABLE BridgeRouteZone (
    RouteZoneBridgeKey INT PRIMARY KEY IDENTITY(1,1),
    RouteKey INT NOT NULL,
    ZoneKey INT NOT NULL,
    AllocationPercentage DECIMAL(5,2), -- % of route dedicated to this zone
    CONSTRAINT FK_Bridge_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey),
    CONSTRAINT FK_Bridge_Zone FOREIGN KEY (ZoneKey) REFERENCES DimWarehouseZone(ZoneKey)
);

-- Query using bridge table
SELECT 
    r.RouteName,
    z.ZoneName,
    brz.AllocationPercentage,
    SUM(f.DistanceKM * brz.AllocationPercentage / 100) AS AllocatedDistanceKM
FROM FactFleetTrips f
INNER JOIN BridgeRouteZone brz ON f.RouteKey = brz.RouteKey
INNER JOIN DimRoute r ON brz.RouteKey = r.RouteKey
INNER JOIN DimWarehouseZone z ON brz.ZoneKey = z.ZoneKey
GROUP BY r.RouteName, z.ZoneName, brz.AllocationPercentage;
```

### Pattern 3: Slowly Changing Dimensions (SCD Type 2)

```sql
-- Track historical changes in product gravity scores
ALTER TABLE DimProductGravity ADD 
    ValidFrom DATETIME2 DEFAULT GETDATE(),
    ValidTo DATETIME2 DEFAULT '9999-12-31',
    IsCurrent BIT DEFAULT 1;

CREATE PROCEDURE sp_UpdateProductGravity
    @SKU VARCHAR(50),
    @NewGravityScore DECIMAL(5,2)
AS
BEGIN
    -- Expire current record
    UPDATE DimProductGravity
    SET ValidTo = GETDATE(), IsCurrent = 0
    WHERE SKU = @SKU AND IsCurrent = 1;
    
    -- Insert new record
    INSERT INTO DimProductGravity (SKU, ProductName, GravityScore, ValidFrom, IsCurrent)
    SELECT SKU, ProductName, @NewGravityScore, GETDATE(), 1
    FROM DimProductGravity
    WHERE SKU = @SKU AND ValidTo = GETDATE();
END;
```

## Configuration Files

### ETL Configuration (JSON)

```json
{
  "etl_config": {
    "sources": {
      "wms": {
        "type": "sql_server",
        "connection_string": "${WMS_CONNECTION_STRING}",
        "tables": ["Operations", "Inventory", "Zones"]
      },
      "telemetry": {
        "type": "rest_api",
        "endpoint": "${TELEMETRY_API_URL}",
        "auth": {
          "type": "bearer",
          "token": "${TELEMETRY_API_TOKEN}"
        },
        "polling_interval_minutes": 15
      },
      "weather": {
        "type": "rest_api",
        "endpoint": "https://api.openweathermap.org/data/2.5/",
        "api_key": "${WEATHER_API_KEY}"
      }
    },
    "target": {
      "server": "${SQL_SERVER_NAME}",
      "database": "LogiFleetPulse",
      "schema": "dbo"
    },
    "schedule": {
      "warehouse_ops": "*/15 * * * *",
      "fleet_trips": "*/15 * * * *",
      "gravity_recalc": "0 2 * * *"
    }
  }
}
```

### Power BI Refresh Schedule (via PowerShell)

```powershell
# Automated Power BI dataset refresh using REST API
$workspaceId = $env:POWERBI_WORKSPACE_ID
$datasetId = $env:POWERBI_DATASET_ID
$accessToken = $env:POWERBI_ACCESS_TOKEN

$headers = @{
    "Authorization" = "Bearer $accessToken"
    "Content-Type" = "application/json"
}

$body = @{
    "notifyOption" = "MailOnFailure"
} | ConvertTo-Json

Invoke-RestMethod -Method Post `
    -Uri "https://api.powerbi.com/v1.0/myorg/groups/$workspaceId/datasets/$datasetId/refreshes" `
    -Headers $headers `
    -Body $body
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom**: Queries joining FactWarehouseOperations and FactFleetTrips take >30 seconds.

**Solution**: 
1. Ensure proper indexing on TimeKey and foreign keys
2. Use materialized views for common cross-fact aggregations:

```sql
-- Create indexed view for pre-aggregated cross-fact KPIs
CREATE VIEW vw_CrossFactKPI_Hourly
WITH SCHEMABINDING
AS
SELECT 
    t.Year,
    t.Month,
    t.DayOfMonth,
    t.HourOfDay,
    COUNT_BIG(*) AS RecordCount,
    AVG(w.DwellTimeMinutes) AS AvgWarehouseDwell,
    AVG(f.IdleTimeMinutes) AS AvgFleetIdle
FROM dbo.FactWarehouseOperations w
INNER JOIN dbo.DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN dbo.FactFleetTrips f ON w.TimeKey = f.TimeKey
GROUP BY t.Year, t.Month, t.DayOfMonth, t.HourOfDay;

CREATE UNIQUE CLUSTERED INDEX IX_CrossFactKPI 
ON vw_CrossFactKPI_Hourly(Year, Month, DayOfMonth, HourOfDay);
```

### Issue: Power BI Dashboard Not Refreshing

**Symptom**: Data in Power BI is stale despite refresh attempts.

**Solution**:
1. Check DirectQuery connection timeout settings
2. Verify SQL Server firewall allows Power BI Service IP ranges
3. Test connection using Power BI Gateway diagnostic:

```powershell
# Test gateway connection
Test-NetConnection -ComputerName $env:SQL_SERVER_NAME -Port 1433
```

### Issue: ETL Load Failures

**Symptom**: sp_LoadWarehouseOperations fails with timeout errors.

**Solution**:
1. Implement batch loading with transaction management:

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations_Batch
    @LastLoadTimestamp DATETIME2,
    @BatchSize INT = 10000
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @RowsProcessed INT = 0;
    DECLARE @TotalRows INT;
    
    SELECT @TotalRows = COUNT(*) 
    FROM StagingWMS.dbo.Operations 
    WHERE OperationTimestamp > @LastLoadTimestamp;
    
    WHILE @RowsProcessed < @TotalRows
    BEGIN
        BEGIN TRANSACTION;
        
        INSERT INTO FactWarehouseOperations (TimeKey, ProductKey, WarehouseZoneKey, OperationType, DwellTimeMinutes, QuantityHandled, Oper
