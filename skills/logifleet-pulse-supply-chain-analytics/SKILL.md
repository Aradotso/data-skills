---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact star schema for warehouse, fleet, and supply chain logistics intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "create a logistics data warehouse with fleet tracking"
  - "build Power BI dashboard for warehouse operations"
  - "implement multi-fact star schema for logistics"
  - "configure cross-modal supply chain analytics"
  - "design warehouse gravity zone optimization"
  - "integrate fleet telemetry with inventory data"
  - "deploy real-time logistics intelligence platform"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that combines warehouse operations, fleet telemetry, inventory management, and external data sources into a unified semantic layer. It uses MS SQL Server for data warehousing with a custom multi-fact star schema and Power BI for visualization. The platform enables:

- Cross-fact KPI harmonization (linking warehouse metrics with fleet performance)
- Predictive bottleneck detection across supply chain touchpoints
- Warehouse gravity zone optimization (spatial dimension based on pick frequency, value, fragility)
- Real-time fleet triage with maintenance prioritization
- Temporal elasticity modeling for what-if scenarios

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQLCMD or SQL Server Management Studio (SSMS)

### Database Deployment

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```bash
sqlcmd -S localhost -d LogiFleetPulse -E -i schema/01_create_database.sql
sqlcmd -S localhost -d LogiFleetPulse -E -i schema/02_create_dimensions.sql
sqlcmd -S localhost -d LogiFleetPulse -E -i schema/03_create_facts.sql
sqlcmd -S localhost -d LogiFleetPulse -E -i schema/04_create_views.sql
sqlcmd -S localhost -d LogiFleetPulse -E -i schema/05_create_procedures.sql
```

3. **Load initial seed data:**
```bash
sqlcmd -S localhost -d LogiFleetPulse -E -i data/seed_data.sql
```

4. **Configure data source connections:**
```bash
cp config_sample.json config.json
# Edit config.json with your connection strings
```

### Power BI Template Setup

1. Open `LogiFleet_Pulse_Master.pbit` in Power BI Desktop
2. Enter your SQL Server connection string when prompted
3. Configure row-level security in Manage Roles
4. Publish to Power BI Service (optional)

## Core Schema Architecture

### Fact Tables

**FactWarehouseOperations** - Tracks all warehouse micro-operations:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    DwellTimeMinutes INT,
    PickRate DECIMAL(10,2), -- units per hour
    PackingTimeSeconds INT,
    EmployeeKey INT FOREIGN KEY REFERENCES DimEmployee(EmployeeKey),
    LoadTimestamp DATETIME2 DEFAULT GETUTCDATE()
);

-- Recommended indexes
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Product 
    ON FactWarehouseOperations(TimeKey, ProductKey) 
    INCLUDE (DwellTimeMinutes, PickRate);
```

**FactFleetTrips** - Captures fleet telemetry and route performance:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripDistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DelayMinutes INT,
    DelayReason VARCHAR(100),
    LoadTimestamp DATETIME2 DEFAULT GETUTCDATE()
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Vehicle 
    ON FactFleetTrips(TimeKey, VehicleKey) 
    INCLUDE (FuelConsumedLiters, IdleTimeMinutes);
```

### Dimension Tables

**DimTime** - 15-minute granularity time dimension:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2,
    DateKey INT,
    TimeSlot15Min VARCHAR(5), -- '00:00', '00:15', '00:30'...
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    WeekOfYear TINYINT,
    MonthOfYear TINYINT,
    MonthName VARCHAR(10),
    QuarterOfYear TINYINT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT
);
```

**DimProductGravity** - Product hierarchy with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName NVARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- pickings per day
    ValueScore DECIMAL(10,2), -- $ per unit
    FragilityScore TINYINT, -- 1-10 scale
    GravityZone VARCHAR(10), -- 'HIGH', 'MEDIUM', 'LOW'
    WeightKG DECIMAL(10,2),
    VolumeM3 DECIMAL(10,4),
    ValidFrom DATETIME2,
    ValidTo DATETIME2,
    IsCurrent BIT DEFAULT 1
);
```

**DimGeography** - Hierarchical location dimension:
```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE,
    LocationType VARCHAR(50), -- 'WAREHOUSE', 'ROUTE_NODE', 'CUSTOMER_SITE'
    LocationName NVARCHAR(200),
    Warehouse VARCHAR(100),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1
);
```

## Key Stored Procedures

### Incremental Data Loading

**usp_LoadWarehouseOperations** - Load new warehouse events:
```sql
CREATE PROCEDURE usp_LoadWarehouseOperations
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        DwellTimeMinutes, PickRate, PackingTimeSeconds, EmployeeKey
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        src.OperationType,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        src.UnitsProcessed / NULLIF(DATEDIFF(HOUR, src.StartTime, src.EndTime), 0) AS PickRate,
        DATEDIFF(SECOND, src.PackStart, src.PackEnd) AS PackingTimeSeconds,
        e.EmployeeKey
    FROM ExternalWarehouseSystem.dbo.Operations src
    INNER JOIN DimTime t ON DATEPART(HOUR, src.StartTime) = t.HourOfDay 
        AND DATEPART(MINUTE, src.StartTime) / 15 * 15 = CAST(SUBSTRING(t.TimeSlot15Min, 4, 2) AS INT)
    INNER JOIN DimGeography g ON src.WarehouseCode = g.LocationID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU AND p.IsCurrent = 1
    INNER JOIN DimEmployee e ON src.EmployeeID = e.EmployeeID
    WHERE src.OperationTimestamp BETWEEN @StartDateTime AND @EndDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.TimeKey = t.TimeKey 
                AND f.ProductKey = p.ProductKey
                AND f.EmployeeKey = e.EmployeeKey
        );
END;
```

**usp_LoadFleetTrips** - Load fleet telemetry data:
```sql
CREATE PROCEDURE usp_LoadFleetTrips
    @StartDateTime DATETIME2,
    @EndDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleKey, RouteKey, OriginGeographyKey, DestinationGeographyKey,
        TripDistanceKM, FuelConsumedLiters, IdleTimeMinutes, 
        LoadingTimeMinutes, UnloadingTimeMinutes, DelayMinutes, DelayReason
    )
    SELECT 
        t.TimeKey,
        v.VehicleKey,
        r.RouteKey,
        go.GeographyKey AS OriginKey,
        gd.GeographyKey AS DestinationKey,
        src.DistanceKM,
        src.FuelLiters,
        src.IdleMinutes,
        src.LoadingMinutes,
        src.UnloadingMinutes,
        DATEDIFF(MINUTE, src.ScheduledArrival, src.ActualArrival) AS DelayMinutes,
        src.DelayReason
    FROM TelematicsSystem.dbo.Trips src
    INNER JOIN DimTime t ON DATEPART(HOUR, src.TripStartTime) = t.HourOfDay
    INNER JOIN DimVehicle v ON src.VehicleID = v.VehicleID
    INNER JOIN DimRoute r ON src.RouteCode = r.RouteCode
    INNER JOIN DimGeography go ON src.OriginCode = go.LocationID
    INNER JOIN DimGeography gd ON src.DestinationCode = gd.LocationID
    WHERE src.TripStartTime BETWEEN @StartDateTime AND @EndDateTime;
END;
```

### Cross-Fact KPI Queries

**vw_CrossFactLogisticsKPI** - Unified view linking warehouse and fleet metrics:
```sql
CREATE VIEW vw_CrossFactLogisticsKPI AS
SELECT 
    t.DateKey,
    t.WeekOfYear,
    g.Warehouse,
    g.Region,
    p.Category,
    p.GravityZone,
    
    -- Warehouse metrics
    AVG(w.DwellTimeMinutes) AS AvgDwellTimeMin,
    SUM(CASE WHEN w.OperationType = 'PICKING' THEN 1 ELSE 0 END) AS TotalPicks,
    AVG(CASE WHEN w.OperationType = 'PICKING' THEN w.PickRate END) AS AvgPickRate,
    
    -- Fleet metrics
    AVG(f.FuelConsumedLiters) AS AvgFuelPerTrip,
    AVG(f.IdleTimeMinutes) AS AvgIdleTime,
    SUM(f.DelayMinutes) AS TotalDelayMinutes,
    
    -- Cross-fact calculations
    AVG(w.DwellTimeMinutes) * AVG(f.IdleTimeMinutes) / 60.0 AS CompoundWasteIndex,
    SUM(CASE WHEN w.DwellTimeMinutes > 72 * 60 THEN 1 ELSE 0 END) AS SlowMovingCount,
    AVG(CASE WHEN p.GravityZone = 'HIGH' THEN f.TripDistanceKM END) AS AvgDistanceHighGravity
    
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips f ON f.TimeKey = t.TimeKey 
    AND f.OriginGeographyKey = g.GeographyKey
    AND f.DestinationGeographyKey IN (
        SELECT GeographyKey FROM DimGeography 
        WHERE Country = g.Country
    )
GROUP BY t.DateKey, t.WeekOfYear, g.Warehouse, g.Region, p.Category, p.GravityZone;
```

## Automated Alerting System

**usp_CheckKPIThresholds** - Generate alerts for threshold breaches:
```sql
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertRecipient VARCHAR(255) = '${ALERT_EMAIL}';
    
    -- Check for excessive fleet idle time
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips
        WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours
        GROUP BY VehicleKey
        HAVING AVG(CAST(IdleTimeMinutes AS FLOAT) / NULLIF(
            DATEDIFF(MINUTE, MIN(LoadTimestamp), MAX(LoadTimestamp)), 0
        )) > 0.15
    )
    BEGIN
        INSERT INTO AlertLog (AlertType, Severity, Message, Timestamp)
        VALUES ('FLEET_IDLE_EXCESS', 'HIGH', 
                'Fleet idling time exceeds 15% threshold', GETUTCDATE());
    END;
    
    -- Check for slow-moving inventory in high-gravity zones
    INSERT INTO AlertLog (AlertType, Severity, Message, Timestamp)
    SELECT 
        'GRAVITY_ZONE_MISALIGNMENT',
        'MEDIUM',
        'SKU ' + p.SKU + ' has low velocity in HIGH gravity zone',
        GETUTCDATE()
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE p.GravityZone = 'HIGH'
        AND w.DwellTimeMinutes > 72 * 60
        AND w.TimeKey >= (SELECT MAX(TimeKey) - 672 FROM DimTime); -- Last week
END;
```

## Power BI DAX Measures

### Core KPIs

**Total Warehouse Operations:**
```dax
TotalOperations = COUNTROWS(FactWarehouseOperations)
```

**Average Dwell Time (Hours):**
```dax
AvgDwellTimeHours = 
DIVIDE(
    SUM(FactWarehouseOperations[DwellTimeMinutes]),
    COUNTROWS(FactWarehouseOperations)
) / 60
```

**Fleet Fuel Efficiency:**
```dax
FuelEfficiencyKMPerLiter = 
DIVIDE(
    SUM(FactFleetTrips[TripDistanceKM]),
    SUM(FactFleetTrips[FuelConsumedLiters])
)
```

**Compound Waste Index:**
```dax
CompoundWasteIndex = 
AVERAGEX(
    SUMMARIZE(
        FactWarehouseOperations,
        DimTime[DateKey],
        DimGeography[Warehouse],
        "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
    ),
    [AvgDwell]
) * 
AVERAGEX(
    SUMMARIZE(
        FactFleetTrips,
        DimTime[DateKey],
        DimVehicle[VehicleID],
        "AvgIdle", AVERAGE(FactFleetTrips[IdleTimeMinutes])
    ),
    [AvgIdle]
) / 60
```

**Gravity Zone Performance Score:**
```dax
GravityZoneScore = 
VAR HighGravityPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[GravityZone] = "HIGH",
        FactWarehouseOperations[OperationType] = "PICKING"
    )
VAR TotalPicks = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "PICKING"
    )
RETURN
DIVIDE(HighGravityPicks, TotalPicks) * 100
```

## Configuration & Environment Variables

### config.json Structure

```json
{
  "sqlServer": {
    "host": "${SQL_SERVER_HOST}",
    "port": 1433,
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}",
    "trustedConnection": false
  },
  "dataSources": {
    "warehouseSystem": {
      "type": "SQL",
      "connectionString": "${WMS_CONNECTION_STRING}"
    },
    "telematicsAPI": {
      "type": "REST",
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "apiKey": "${TELEMATICS_API_KEY}"
    },
    "weatherAPI": {
      "type": "REST",
      "endpoint": "https://api.weather.com/v3/",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refreshSchedule": {
    "incrementalLoadMinutes": 15,
    "fullRefreshHours": 24
  },
  "alerting": {
    "enabled": true,
    "recipients": ["${ALERT_EMAIL}"],
    "smtpServer": "${SMTP_SERVER}",
    "smtpPort": 587,
    "smtpUsername": "${SMTP_USERNAME}",
    "smtpPassword": "${SMTP_PASSWORD}"
  }
}
```

## Common Patterns & Workflows

### Daily Data Refresh Pattern

```sql
-- Execute daily at 00:15 via SQL Agent Job
DECLARE @Yesterday DATETIME2 = DATEADD(DAY, -1, CAST(GETDATE() AS DATE));
DECLARE @Today DATETIME2 = CAST(GETDATE() AS DATE);

EXEC usp_LoadWarehouseOperations @Yesterday, @Today;
EXEC usp_LoadFleetTrips @Yesterday, @Today;
EXEC usp_CheckKPIThresholds;

-- Rebuild indexes weekly
IF DATEPART(WEEKDAY, GETDATE()) = 1 -- Sunday
BEGIN
    ALTER INDEX ALL ON FactWarehouseOperations REBUILD;
    ALTER INDEX ALL ON FactFleetTrips REBUILD;
END;
```

### Product Gravity Zone Recalculation

```sql
-- Run monthly to update gravity zones based on velocity trends
UPDATE DimProductGravity
SET 
    VelocityScore = velocity.PicksPerDay,
    GravityZone = CASE
        WHEN velocity.PicksPerDay >= 50 AND ValueScore >= 100 THEN 'HIGH'
        WHEN velocity.PicksPerDay >= 20 OR ValueScore >= 50 THEN 'MEDIUM'
        ELSE 'LOW'
    END,
    ValidTo = GETUTCDATE(),
    IsCurrent = 0
FROM (
    SELECT 
        p.ProductKey,
        COUNT(*) / 30.0 AS PicksPerDay
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType = 'PICKING'
        AND w.LoadTimestamp >= DATEADD(MONTH, -1, GETUTCDATE())
    GROUP BY p.ProductKey
) velocity
WHERE DimProductGravity.ProductKey = velocity.ProductKey
    AND DimProductGravity.IsCurrent = 1;

-- Insert new current records
INSERT INTO DimProductGravity (SKU, ProductName, Category, Subcategory, 
    VelocityScore, ValueScore, FragilityScore, GravityZone, 
    WeightKG, VolumeM3, ValidFrom, IsCurrent)
SELECT 
    SKU, ProductName, Category, Subcategory,
    VelocityScore, ValueScore, FragilityScore, GravityZone,
    WeightKG, VolumeM3, GETUTCDATE(), 1
FROM DimProductGravity
WHERE IsCurrent = 0 
    AND ValidTo >= DATEADD(MINUTE, -5, GETUTCDATE());
```

### Cross-Fact Analysis Query Pattern

```sql
-- Find shipments delayed due to weather that originated from 
-- cold-storage zones with high dwell time
SELECT 
    t.FullDateTime AS ShipmentTime,
    go.LocationName AS OriginWarehouse,
    gd.LocationName AS Destination,
    p.SKU,
    p.Category,
    w.DwellTimeMinutes / 60.0 AS DwellHours,
    f.DelayMinutes,
    f.DelayReason
FROM FactFleetTrips f
INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
INNER JOIN DimGeography go ON f.OriginGeographyKey = go.GeographyKey
INNER JOIN DimGeography gd ON f.DestinationGeographyKey = gd.GeographyKey
INNER JOIN FactWarehouseOperations w ON w.WarehouseKey = go.GeographyKey
    AND w.TimeKey BETWEEN f.TimeKey - 96 AND f.TimeKey -- Within 24 hours before trip
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE f.DelayReason LIKE '%weather%'
    AND go.LocationName LIKE '%Cold Storage%'
    AND w.DwellTimeMinutes > 72 * 60
    AND t.DateKey >= DATEADD(QUARTER, -1, GETDATE())
ORDER BY f.DelayMinutes DESC;
```

## Troubleshooting

### Issue: Power BI refresh fails with timeout

**Solution:** Increase command timeout in Power BI connection settings:
```m
// In Power BI Advanced Editor
let
    Source = Sql.Database(
        "${SQL_SERVER_HOST}", 
        "LogiFleetPulse",
        [CommandTimeout=#duration(0, 0, 30, 0)] // 30 minutes
    )
in
    Source
```

### Issue: Cross-fact queries running slowly

**Diagnosis:**
```sql
-- Check missing indexes
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    s.avg_fragmentation_in_percent,
    s.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') s
INNER JOIN sys.indexes i ON s.object_id = i.object_id 
    AND s.index_id = i.index_id
WHERE s.avg_fragmentation_in_percent > 30
    AND s.page_count > 1000;
```

**Solution:** Add covering indexes on join columns:
```sql
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time_Warehouse_Product
    ON FactWarehouseOperations(TimeKey, WarehouseKey, ProductKey)
    INCLUDE (DwellTimeMinutes, PickRate, OperationType);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time_Origin_Destination
    ON FactFleetTrips(TimeKey, OriginGeographyKey, DestinationGeographyKey)
    INCLUDE (DelayMinutes, FuelConsumedLiters);
```

### Issue: Gravity zone updates not reflecting in Power BI

**Cause:** Row-level security filtering on `IsCurrent = 1`

**Solution:** Ensure Power BI relationship uses current records:
```m
// In Power BI relationship settings
= Table.SelectRows(
    DimProductGravity, 
    each [IsCurrent] = true
)
```

### Issue: Alert emails not sending

**Check SMTP configuration:**
```sql
-- Test Database Mail
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'LogiFleetPulse',
    @recipients = '${ALERT_EMAIL}',
    @subject = 'Test Email',
    @body = 'Database Mail is working';

-- View mail log
SELECT * FROM msdb.dbo.sysmail_log
ORDER BY log_date DESC;
```

## Best Practices

1. **Partition large fact tables** by date for better query performance:
```sql
ALTER DATABASE LogiFleetPulse ADD FILEGROUP FG_2025;
ALTER DATABASE LogiFleetPulse ADD FILEGROUP FG_2026;

CREATE PARTITION FUNCTION PF_DateKey (INT)
AS RANGE RIGHT FOR VALUES (20250101, 20260101);

CREATE PARTITION SCHEME PS_DateKey
AS PARTITION PF_DateKey TO (FG_2024, FG_2025, FG_2026);

-- Rebuild table on partition scheme
CREATE TABLE FactWarehouseOperations_New (...)
ON PS_DateKey(TimeKey);
```

2. **Implement incremental refresh** in Power BI for large datasets

3. **Use columnstore indexes** for analytical workloads:
```sql
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouse
    ON FactWarehouseOperations (
        TimeKey, ProductKey, WarehouseKey, 
        DwellTimeMinutes, PickRate
    );
```

4. **Archive historical data** older than 2 years to separate tables

5. **Monitor query performance** with Extended Events:
```sql
CREATE EVENT SESSION LogiFleetPulse_Monitoring ON SERVER
ADD EVENT sqlserver.sql_batch_completed (
    WHERE database_name = 'LogiFleetPulse'
        AND duration > 5000000 -- 5 seconds
);
```
