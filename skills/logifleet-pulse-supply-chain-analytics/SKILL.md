---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for logistics intelligence with multi-fact star schema, fleet optimization, and warehouse analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure logistics data warehouse with Power BI"
  - "implement multi-fact star schema for fleet management"
  - "deploy warehouse gravity zone optimization"
  - "create logistics intelligence dashboard"
  - "configure cross-modal supply chain KPIs"
  - "set up real-time fleet telemetry tracking"
  - "implement predictive bottleneck detection for logistics"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to implement and configure LogiFleet Pulse, a comprehensive logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. The system unifies warehouse operations, fleet telemetry, and supply chain metrics through a multi-fact star schema architecture.

## What LogiFleet Pulse Does

LogiFleet Pulse is a logistics data platform that:

- **Unifies disparate data sources**: Warehouse Management Systems (WMS), telematics/GPS feeds, supplier portals, weather APIs, and order history
- **Multi-fact star schema**: Links warehouse operations, fleet trips, and cross-dock activities through shared dimensions
- **Real-time dashboarding**: Power BI dashboards with 15-minute refresh intervals
- **Predictive analytics**: Bottleneck detection, fleet maintenance prioritization, warehouse gravity zones
- **Cross-fact KPI harmonization**: Mathematically links inventory turnover with fuel consumption and delivery metrics

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Power BI Pro/Premium license (for sharing and scheduled refresh)
- SQL Server Management Studio (SSMS)

### Database Schema Deployment

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema**:
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the master schema script
:r "sql/01_create_database.sql"
:r "sql/02_create_tables.sql"
:r "sql/03_create_relationships.sql"
:r "sql/04_create_views.sql"
:r "sql/05_create_stored_procedures.sql"
```

3. **Configure data source connections**:
```json
// config.json (do not commit with real credentials)
{
  "sql_server": {
    "host": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "data_sources": {
    "wms_api": "${WMS_API_ENDPOINT}",
    "telematics_api": "${TELEMATICS_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  }
}
```

### Power BI Setup

1. **Open the template**:
```
File → Open → LogiFleet_Pulse_Master.pbit
```

2. **Configure data source**:
- Enter SQL Server host and database name
- Choose authentication method (Windows or SQL Server)
- Power BI will auto-detect fact tables and relationships

3. **Set up scheduled refresh** (Power BI Service):
```
Dataset Settings → Scheduled Refresh → Configure credentials
Set refresh frequency: Every 15 minutes (requires Premium)
```

## Core Data Model Architecture

### Fact Tables

**FactWarehouseOperations**:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingTimeMinutes INT,
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Indexing strategy for performance
CREATE NONCLUSTERED INDEX IX_WO_Time ON FactWarehouseOperations(TimeKey) INCLUDE (DwellTimeMinutes);
CREATE NONCLUSTERED INDEX IX_WO_Product ON FactWarehouseOperations(ProductKey, OperationType);
```

**FactFleetTrips**:
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    FuelConsumptionLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    DistanceKm DECIMAL(10,2),
    LoadWeightKg DECIMAL(10,2),
    OnTimeDelivery BIT,
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    CONSTRAINT FK_FT_Route FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FT_Time_Vehicle ON FactFleetTrips(TimeKey, VehicleKey);
CREATE NONCLUSTERED INDEX IX_FT_Route ON FactFleetTrips(RouteKey) INCLUDE (OnTimeDelivery);
```

**FactCrossDock**:
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID BIGINT,
    OutboundTripID BIGINT,
    TransferTimeMinutes INT,
    QuantityUnits INT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Inbound FOREIGN KEY (InboundTripID) REFERENCES FactFleetTrips(TripID),
    CONSTRAINT FK_CD_Outbound FOREIGN KEY (OutboundTripID) REFERENCES FactFleetTrips(TripID)
);
```

### Dimension Tables

**DimTime** (15-minute granularity):
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfMonth INT,
    DayOfWeek INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10)
);

-- Populate with time dimension generator
EXEC sp_GenerateTimeDimension @StartDate = '2024-01-01', @EndDate = '2027-12-31';
```

**DimProductGravity** (Warehouse Gravity Zones™):
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    GravityScore DECIMAL(5,2), -- Calculated: velocity × value × (1/fragility)
    VelocityClass VARCHAR(20), -- 'Fast-Mover', 'Medium-Mover', 'Slow-Mover'
    ValueTier VARCHAR(20), -- 'High', 'Medium', 'Low'
    FragilityIndex DECIMAL(3,2),
    RecommendedZone VARCHAR(50) -- Derived from GravityScore
);

-- Update gravity scores based on operational data
CREATE PROCEDURE sp_UpdateProductGravity
AS
BEGIN
    UPDATE DimProductGravity
    SET GravityScore = (
        SELECT (COUNT(*) / 90.0) * AVG(ProductValue) / ISNULL(NULLIF(FragilityIndex, 0), 1)
        FROM FactWarehouseOperations wo
        WHERE wo.ProductKey = DimProductGravity.ProductKey
          AND wo.TimeKey >= (SELECT MAX(TimeKey) - 90 FROM DimTime)
    ),
    VelocityClass = CASE
        WHEN GravityScore > 75 THEN 'Fast-Mover'
        WHEN GravityScore > 30 THEN 'Medium-Mover'
        ELSE 'Slow-Mover'
    END;
END;
```

**DimVehicle**:
```sql
CREATE TABLE DimVehicle (
    VehicleKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL,
    VehicleType VARCHAR(50), -- 'Truck', 'Van', 'Refrigerated'
    Capacity DECIMAL(10,2),
    RegistrationDate DATE,
    LastMaintenanceDate DATE,
    MaintenancePriorityScore DECIMAL(5,2) -- Updated by adaptive triage engine
);
```

## Key Stored Procedures & Views

### Cross-Fact KPI Harmonization View

```sql
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    t.FullDateTime,
    t.Year,
    t.Month,
    w.WarehouseName,
    p.SKU,
    p.GravityScore,
    -- Warehouse metrics
    AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
    AVG(wo.PickRateUnitsPerHour) AS AvgPickRate,
    -- Fleet metrics
    AVG(ft.FuelConsumptionLiters) AS AvgFuelConsumption,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    SUM(CASE WHEN ft.OnTimeDelivery = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS OnTimeDeliveryPct,
    -- Cross-dock efficiency
    AVG(cd.TransferTimeMinutes) AS AvgCrossDockTime
FROM DimTime t
LEFT JOIN FactWarehouseOperations wo ON t.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
LEFT JOIN FactCrossDock cd ON t.TimeKey = cd.TimeKey
LEFT JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
LEFT JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE t.FullDateTime >= DATEADD(MONTH, -3, GETDATE())
GROUP BY t.FullDateTime, t.Year, t.Month, w.WarehouseName, p.SKU, p.GravityScore;
```

### Predictive Bottleneck Detection

```sql
CREATE PROCEDURE sp_DetectBottlenecks
    @ThresholdDwellTime INT = 72, -- hours
    @ThresholdIdleTime DECIMAL(5,2) = 15.0 -- percentage
AS
BEGIN
    -- Warehouse bottlenecks
    SELECT 
        'Warehouse' AS BottleneckType,
        w.WarehouseName AS Location,
        p.SKU,
        AVG(wo.DwellTimeMinutes) / 60.0 AS AvgDwellHours,
        'Dwell time exceeds threshold' AS Issue,
        p.GravityScore,
        'Consider moving to higher gravity zone' AS Recommendation
    FROM FactWarehouseOperations wo
    JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY w.WarehouseName, p.SKU, p.GravityScore
    HAVING AVG(wo.DwellTimeMinutes) / 60.0 > @ThresholdDwellTime

    UNION ALL

    -- Fleet bottlenecks
    SELECT 
        'Fleet' AS BottleneckType,
        v.VehicleID AS Location,
        r.RouteName AS SKU,
        AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + (ft.DistanceKm / 60.0 * 60), 0)) AS IdleTimePct,
        'Idle time percentage exceeds threshold' AS Issue,
        NULL AS GravityScore,
        'Review route efficiency and driver behavior' AS Recommendation
    FROM FactFleetTrips ft
    JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY v.VehicleID, r.RouteName
    HAVING AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + (ft.DistanceKm / 60.0 * 60), 0)) > @ThresholdIdleTime;
END;
```

### Adaptive Fleet Triage Engine

```sql
CREATE PROCEDURE sp_FleetMaintenanceTriage
AS
BEGIN
    UPDATE DimVehicle
    SET MaintenancePriorityScore = (
        SELECT 
            -- Base score from last maintenance age
            (DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) / 30.0) * 10 +
            -- Weighted by recent trip volume
            (SELECT COUNT(*) FROM FactFleetTrips ft 
             WHERE ft.VehicleKey = DimVehicle.VehicleKey 
               AND ft.TimeKey >= (SELECT MAX(TimeKey) - 30 FROM DimTime)) * 2 +
            -- Penalty for high idle time
            (SELECT AVG(IdleTimeMinutes) FROM FactFleetTrips ft
             WHERE ft.VehicleKey = DimVehicle.VehicleKey
               AND ft.TimeKey >= (SELECT MAX(TimeKey) - 30 FROM DimTime)) / 10.0 +
            -- Penalty for refrigerated units (higher criticality)
            CASE WHEN VehicleType = 'Refrigerated' THEN 20 ELSE 0 END
    );

    -- Return prioritized maintenance queue
    SELECT TOP 20
        VehicleID,
        VehicleType,
        LastMaintenanceDate,
        DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
        MaintenancePriorityScore,
        'Schedule maintenance within ' + 
            CASE 
                WHEN MaintenancePriorityScore > 80 THEN '24 hours'
                WHEN MaintenancePriorityScore > 50 THEN '3 days'
                ELSE '1 week'
            END AS RecommendedAction
    FROM DimVehicle
    ORDER BY MaintenancePriorityScore DESC;
END;
```

## Data Loading & ETL Patterns

### Incremental Load Pattern

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET @LastLoadDateTime = ISNULL(@LastLoadDateTime, 
        (SELECT MAX(LoadDateTime) FROM FactWarehouseOperations_Staging));

    -- Load from staging to fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        DwellTimeMinutes, PickRateUnitsPerHour, PackingTimeMinutes
    )
    SELECT 
        t.TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        s.OperationType,
        s.DwellTimeMinutes,
        s.PickRateUnitsPerHour,
        s.PackingTimeMinutes
    FROM FactWarehouseOperations_Staging s
    JOIN DimTime t ON DATEADD(MINUTE, (s.EventMinute / 15) * 15, 
        CAST(CAST(s.EventDateTime AS DATE) AS DATETIME) + 
        CAST(DATEPART(HOUR, s.EventDateTime) AS VARCHAR(2)) + ':00') = t.FullDateTime
    JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode
    JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE s.LoadDateTime > @LastLoadDateTime;

    -- Update gravity scores after load
    EXEC sp_UpdateProductGravity;
END;
```

### External Data Source Configuration (Weather API)

```sql
-- Create external data source for weather API enrichment
CREATE EXTERNAL DATA SOURCE WeatherAPISource
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = '${WEATHER_API_ENDPOINT}'
);

-- Create external table for weather data
CREATE EXTERNAL TABLE ExtWeatherData (
    WeatherTimestamp DATETIME,
    LocationCode VARCHAR(20),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    DelayRiskScore DECIMAL(3,2)
)
WITH (
    LOCATION = '/weather_feeds/',
    DATA_SOURCE = WeatherAPISource,
    FILE_FORMAT = JSONFormat
);

-- Join weather data with fleet trips
CREATE VIEW vw_FleetTripsWithWeather AS
SELECT 
    ft.*,
    w.Temperature,
    w.Precipitation,
    w.DelayRiskScore,
    CASE 
        WHEN w.DelayRiskScore > 0.7 AND ft.OnTimeDelivery = 0 THEN 1
        ELSE 0
    END AS WeatherRelatedDelay
FROM FactFleetTrips ft
JOIN DimRoute r ON ft.RouteKey = r.RouteKey
LEFT JOIN ExtWeatherData w ON 
    DATEADD(MINUTE, (DATEPART(MINUTE, ft.TripStartTime) / 15) * 15, 
        CAST(CAST(ft.TripStartTime AS DATE) AS DATETIME) + 
        CAST(DATEPART(HOUR, ft.TripStartTime) AS VARCHAR(2)) + ':00') = w.WeatherTimestamp
    AND r.OriginLocationCode = w.LocationCode;
```

## Power BI DAX Measures

### Core KPI Measures

```dax
// Total Dwell Time
Total Dwell Time = 
SUM(FactWarehouseOperations[DwellTimeMinutes])

// Average Dwell Time by Gravity Zone
Avg Dwell Time by Zone = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[RecommendedZone])
)

// Fleet Utilization %
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[DistanceKm]),
    SUM(FactFleetTrips[DistanceKm]) + 
    (SUM(FactFleetTrips[IdleTimeMinutes]) / 60 * 60), // Assume 60 km/h average
    0
) * 100

// On-Time Delivery Rate
OTD Rate = 
DIVIDE(
    CALCULATE(COUNT(FactFleetTrips[TripID]), FactFleetTrips[OnTimeDelivery] = TRUE()),
    COUNT(FactFleetTrips[TripID]),
    0
) * 100

// Cost per Delivery (fuel + idle time cost)
Cost per Delivery = 
VAR FuelCostPerLiter = 1.50 // Update with actual rate or parameter
VAR IdleCostPerHour = 25.00
RETURN
DIVIDE(
    SUM(FactFleetTrips[FuelConsumptionLiters]) * FuelCostPerLiter +
    (SUM(FactFleetTrips[IdleTimeMinutes]) / 60) * IdleCostPerHour,
    COUNT(FactFleetTrips[TripID]),
    0
)

// Cross-Fact: Dwell Impact on Fuel
Dwell-Fuel Correlation = 
VAR DwellTable = 
    SUMMARIZE(
        FactWarehouseOperations,
        FactWarehouseOperations[ProductKey],
        "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
    )
VAR FleetTable = 
    SUMMARIZE(
        FactFleetTrips,
        FactFleetTrips[VehicleKey],
        "AvgFuel", AVERAGE(FactFleetTrips[FuelConsumptionLiters])
    )
RETURN
// Simplified correlation placeholder - actual implementation uses CORREL function
AVERAGEX(DwellTable, [AvgDwell]) / AVERAGEX(FleetTable, [AvgFuel])
```

### Time Intelligence Measures

```dax
// Dwell Time vs. Previous Period
Dwell Time vs. Prev Period = 
VAR CurrentPeriod = [Total Dwell Time]
VAR PreviousPeriod = 
    CALCULATE(
        [Total Dwell Time],
        DATEADD(DimTime[FullDateTime], -1, MONTH)
    )
RETURN
DIVIDE(CurrentPeriod - PreviousPeriod, PreviousPeriod, 0) * 100

// Rolling 7-Day Average Fleet Idle Time
Rolling 7D Avg Idle = 
CALCULATE(
    AVERAGE(FactFleetTrips[IdleTimeMinutes]),
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -7,
        DAY
    )
)
```

## Alerting Configuration

### SQL Server Alert Job

```sql
-- Create alert stored procedure
CREATE PROCEDURE sp_SendAlerts
AS
BEGIN
    DECLARE @AlertSubject VARCHAR(200);
    DECLARE @AlertBody VARCHAR(MAX);

    -- Check for high dwell time alerts
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
          AND wo.DwellTimeMinutes > 4320 -- 72 hours
    )
    BEGIN
        SET @AlertSubject = 'ALERT: High Dwell Time Detected';
        SET @AlertBody = 'One or more SKUs have exceeded 72-hour dwell time. Execute sp_DetectBottlenecks for details.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = @AlertSubject,
            @body = @AlertBody;
    END;

    -- Check for fleet idle time alerts
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips ft
        JOIN DimTime t ON ft.TimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
          AND (ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + (ft.DistanceKm / 60.0 * 60), 0)) > 15
    )
    BEGIN
        SET @AlertSubject = 'ALERT: High Fleet Idle Time';
        SET @AlertBody = 'Fleet idle time exceeds 15% threshold. Review route efficiency.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = @AlertSubject,
            @body = @AlertBody;
    END;
END;

-- Schedule as SQL Server Agent Job (every 15 minutes)
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_AlertMonitor',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_AlertMonitor',
    @step_name = 'Check_Alerts',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_SendAlerts',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_AlertMonitor',
    @schedule_name = 'Every15Minutes';
```

## Common Patterns & Best Practices

### Row-Level Security Implementation

```sql
-- Create security table
CREATE TABLE SecurityUserWarehouse (
    Username VARCHAR(100),
    WarehouseKey INT,
    CONSTRAINT FK_Security_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey)
);

-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@WarehouseKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS SecurityCheck
WHERE @WarehouseKey IN (
    SELECT WarehouseKey 
    FROM dbo.SecurityUserWarehouse 
    WHERE Username = USER_NAME()
)
OR IS_MEMBER('LogisticsAdmin') = 1;

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(WarehouseKey)
ON dbo.FactCrossDock
WITH (STATE = ON);
```

### Performance Optimization - Columnstore Index

```sql
-- Create columnstore index for large fact tables
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (
    TimeKey, WarehouseKey, ProductKey, DwellTimeMinutes, PickRateUnitsPerHour
)
WHERE TimeKey < (SELECT MAX(TimeKey) - 90 FROM DimTime); -- Exclude recent hot data

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON FactFleetTrips (
    TimeKey, VehicleKey, RouteKey, FuelConsumptionLiters, IdleTimeMinutes, OnTimeDelivery
)
WHERE TimeKey < (SELECT MAX(TimeKey) - 90 FROM DimTime);
```

### Partitioning Strategy for Scale

```sql
-- Create partition function (monthly partitions)
CREATE PARTITION FUNCTION pf_Monthly (INT)
AS RANGE RIGHT FOR VALUES (
    20240101, 20240201, 20240301, 20240401, 20240501, 20240601,
    20240701, 20240801, 20240901, 20241001, 20241101, 20241201
);

-- Create partition scheme
CREATE PARTITION SCHEME ps_Monthly
AS PARTITION pf_Monthly
ALL TO ([PRIMARY]);

-- Recreate fact table with partitioning
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationID BIGINT IDENTITY(1,1),
    TimeKey INT NOT NULL,
    -- ... other columns ...
    CONSTRAINT PK_WO_Part PRIMARY KEY (TimeKey, OperationID)
) ON ps_Monthly(TimeKey);
```

## Troubleshooting

### Issue: Power BI Refresh Failing

**Symptoms**: Scheduled refresh times out or returns "data source error"

**Solutions**:
1. Check SQL Server firewall rules allow Power BI Service IP ranges
2. Verify credentials are stored correctly in dataset settings
3. Reduce query complexity by creating pre-aggregated views:

```sql
CREATE VIEW vw_PreAggregatedDaily AS
SELECT 
    CAST(t.FullDateTime AS DATE) AS Date,
    w.WarehouseKey,
    p.ProductKey,
    SUM(wo.DwellTimeMinutes) AS TotalDwellTime,
    AVG(wo.PickRateUnitsPerHour) AS AvgPickRate,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
JOIN DimTime t ON wo.TimeKey = t.TimeKey
JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
GROUP BY CAST(t.FullDateTime AS DATE), w.WarehouseKey, p.ProductKey;
```

### Issue: Slow Cross-Fact Queries

**Symptoms**: Queries joining multiple fact tables take >30 seconds

**Solutions**:
1. Ensure proper indexing on join keys:
```sql
CREATE INDEX IX_Fleet_TimeKey ON FactFleetTrips(TimeKey) INCLUDE (VehicleKey, FuelConsumptionLiters);
CREATE INDEX IX_Warehouse_TimeKey ON FactWarehouseOperations(TimeKey) INCLUDE (ProductKey, DwellTimeMinutes);
```

2. Use query hints for complex joins:
```sql
SELECT /* ... */
FROM FactWarehouseOperations wo WITH (NOLOCK)
JOIN FactFleetTrips ft WITH (NOLOCK) ON wo.TimeKey = ft.TimeKey
OPTION (HASH JOIN, MAXDOP 4);
```

### Issue: Gravity Score Not Updating

**Symptoms**: Product gravity scores remain static despite operational changes

**Solutions**:
1. Schedule `sp_UpdateProductGravity` as daily SQL Agent job
2. Verify data is loading into `FactWarehouseOperations`:
```sql
SELECT COUNT(*), MAX(TimeKey) 
FROM FactWarehouseOperations
WHERE TimeKey >= (SELECT MAX(TimeKey) - 90 FROM DimTime);
```

3. Check for divide-by-zero issues:
```sql
SELECT ProductKey, SKU, FragilityIndex
FROM DimProductGravity
WHERE FragilityIndex = 0 OR FragilityIndex IS NULL;
```

### Issue: External Weather Data Not Loading

**Symptoms**: `ExtWeatherData` returns no rows or stale data

**Solutions**:
1. Verify external data source connectivity:
```sql
SELECT * FROM sys.external_data_sources WHERE name = 'WeatherAPISource';
```

2. Test API endpoint separately with `curl`:
```bash
curl
