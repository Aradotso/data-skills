---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI data warehousing template for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up logistics analytics dashboard"
  - "implement supply chain data warehouse"
  - "create fleet management Power BI report"
  - "build warehouse operations analytics"
  - "configure logifleet pulse database"
  - "deploy logistics KPI tracking system"
  - "integrate supply chain data model"
  - "analyze fleet telemetry and warehouse metrics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is a comprehensive data warehousing and analytics template for logistics and supply chain operations. It provides a multi-fact star schema in MS SQL Server that integrates warehouse operations, fleet telemetry, inventory management, and external data sources. The system includes Power BI templates for real-time visualization and KPI tracking.

**Key Capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet + inventory)
- Real-time dashboard templates in Power BI
- Predictive bottleneck detection
- Warehouse gravity zone optimization
- Fleet maintenance triage engine
- Role-based access control and row-level security

## Installation

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources (WMS, TMS, telemetry APIs)

### Database Setup

1. **Clone the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the main schema creation script
:r schema/01_create_database.sql
:r schema/02_create_dimensions.sql
:r schema/03_create_facts.sql
:r schema/04_create_views.sql
:r schema/05_create_stored_procedures.sql
```

3. **Configure data source connections:**
```json
{
  "connections": {
    "wms_api": {
      "type": "REST",
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "fleet_telemetry": {
      "type": "SQL",
      "connection_string": "${FLEET_DB_CONNECTION_STRING}"
    },
    "erp_system": {
      "type": "ODBC",
      "dsn": "${ERP_DSN}",
      "credentials": "${ERP_CREDENTIALS}"
    }
  },
  "refresh_interval_minutes": 15
}
```

### Power BI Setup

1. **Open the Power BI template:**
```powershell
# Open the .pbit template file
start LogiFleet_Pulse_Master.pbit
```

2. **Configure data source connection:**
- Enter your SQL Server instance name
- Select appropriate authentication method
- Test connection and import data model

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity metrics:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    DwellTimeMinutes INT,
    QuantityHandled INT,
    OperatorKey INT,
    CycleTimeSeconds INT,
    ErrorFlag BIT DEFAULT 0,
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    FOREIGN KEY (WarehouseZoneKey) REFERENCES DimWarehouseZone(ZoneKey)
);

-- Clustered columnstore index for fast aggregation
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations;
```

**FactFleetTrips** - Fleet and route telemetry:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeStartKey INT NOT NULL,
    TimeEndKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    DistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AvgSpeedKmh DECIMAL(5,2),
    MaxSpeedKmh DECIMAL(5,2),
    WeatherConditionKey INT,
    DelayMinutes INT,
    FOREIGN KEY (TimeStartKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey),
    FOREIGN KEY (RouteKey) REFERENCES DimRoute(RouteKey)
);

CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle_Time 
ON FactFleetTrips(VehicleKey, TimeStartKey);
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfYear INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20),
    Hour INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalYear INT,
    FiscalQuarter INT,
    ShiftCode VARCHAR(10) -- 'MORNING', 'AFTERNOON', 'NIGHT'
);

-- Populate time dimension (15-minute buckets for 5 years)
DECLARE @StartDate DATETIME = '2024-01-01 00:00:00';
DECLARE @EndDate DATETIME = '2029-12-31 23:45:00';

WITH TimeSeries AS (
    SELECT @StartDate AS FullDateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, FullDateTime)
    FROM TimeSeries
    WHERE FullDateTime < @EndDate
)
INSERT INTO DimTime (TimeKey, FullDateTime, Date, Year, Quarter, Month, Week, 
                     DayOfYear, DayOfMonth, DayOfWeek, DayName, Hour, MinuteBucket,
                     IsWeekend, IsHoliday)
SELECT 
    CAST(FORMAT(FullDateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    FullDateTime,
    CAST(FullDateTime AS DATE) AS Date,
    YEAR(FullDateTime) AS Year,
    DATEPART(QUARTER, FullDateTime) AS Quarter,
    MONTH(FullDateTime) AS Month,
    DATEPART(WEEK, FullDateTime) AS Week,
    DATEPART(DAYOFYEAR, FullDateTime) AS DayOfYear,
    DAY(FullDateTime) AS DayOfMonth,
    DATEPART(WEEKDAY, FullDateTime) AS DayOfWeek,
    DATENAME(WEEKDAY, FullDateTime) AS DayName,
    DATEPART(HOUR, FullDateTime) AS Hour,
    DATEPART(MINUTE, FullDateTime) AS MinuteBucket,
    CASE WHEN DATEPART(WEEKDAY, FullDateTime) IN (1,7) THEN 1 ELSE 0 END AS IsWeekend,
    0 AS IsHoliday -- Populate via separate holiday table join
FROM TimeSeries
OPTION (MAXRECURSION 0);
```

**DimProductGravity** - Product classification with gravity scoring:
```sql
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    CategoryL1 VARCHAR(100), -- Top-level category
    CategoryL2 VARCHAR(100), -- Sub-category
    CategoryL3 VARCHAR(100), -- Detail category
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    UnitValue DECIMAL(10,2),
    FragilityScore INT, -- 1-10 scale
    VelocityScore DECIMAL(5,2), -- Calculated from historical picks
    GravityScore DECIMAL(5,2), -- Composite metric
    OptimalZoneType VARCHAR(50), -- 'HIGH_GRAVITY', 'MEDIUM_GRAVITY', 'LOW_GRAVITY'
    IsActive BIT DEFAULT 1,
    ValidFrom DATETIME DEFAULT GETDATE(),
    ValidTo DATETIME
);

-- Stored procedure to calculate gravity scores
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    UPDATE p
    SET 
        p.VelocityScore = ISNULL(v.PicksPerDay, 0),
        p.GravityScore = (
            (ISNULL(v.PicksPerDay, 0) * 0.4) + -- Velocity weight: 40%
            ((p.UnitValue / 100) * 0.3) + -- Value weight: 30%
            (p.FragilityScore * 0.2) + -- Fragility weight: 20%
            ((1.0 / NULLIF(p.UnitVolume, 0)) * 0.1) -- Density weight: 10%
        ),
        p.OptimalZoneType = CASE
            WHEN (calculated_gravity_score) > 7.5 THEN 'HIGH_GRAVITY'
            WHEN (calculated_gravity_score) > 4.0 THEN 'MEDIUM_GRAVITY'
            ELSE 'LOW_GRAVITY'
        END
    FROM DimProduct p
    LEFT JOIN (
        SELECT 
            ProductKey,
            COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(t.Date), MAX(t.Date)), 0) AS PicksPerDay
        FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE wo.OperationType = 'PICK'
          AND t.Date >= DATEADD(DAY, -90, GETDATE())
        GROUP BY ProductKey
    ) v ON p.ProductKey = v.ProductKey;
END;
```

## Common Queries and Patterns

### Cross-Fact KPI: Dwell Time vs. Fleet Idle Time

```sql
-- Find correlation between warehouse dwell time and fleet idling
WITH WarehouseDwell AS (
    SELECT 
        t.Date,
        p.ProductSKU,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes
    FROM FactWarehouseOperations wo
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
      AND wo.OperationType = 'PUTAWAY'
    GROUP BY t.Date, p.ProductSKU
),
FleetIdle AS (
    SELECT 
        t.Date,
        AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + ft.LoadingTimeMinutes + ft.UnloadingTimeMinutes, 0)) AS IdlePercentage
    FROM FactFleetTrips ft
    JOIN DimTime t ON ft.TimeStartKey = t.TimeKey
    WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY t.Date
)
SELECT 
    wd.Date,
    wd.ProductSKU,
    wd.AvgDwellMinutes,
    fi.IdlePercentage,
    -- Simple correlation indicator
    CASE 
        WHEN wd.AvgDwellMinutes > 72 AND fi.IdlePercentage > 15 THEN 'HIGH_CORRELATION'
        WHEN wd.AvgDwellMinutes > 48 AND fi.IdlePercentage > 10 THEN 'MEDIUM_CORRELATION'
        ELSE 'LOW_CORRELATION'
    END AS CorrelationFlag
FROM WarehouseDwell wd
JOIN FleetIdle fi ON wd.Date = fi.Date
ORDER BY wd.Date DESC, wd.AvgDwellMinutes DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify potential bottlenecks based on trending metrics
CREATE VIEW vw_BottleneckHeatmap AS
WITH CurrentMetrics AS (
    SELECT 
        wz.ZoneName,
        COUNT(DISTINCT wo.ProductKey) AS ActiveSKUs,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        SUM(wo.QuantityHandled) AS TotalVolume,
        wz.MaxCapacity,
        (SUM(wo.QuantityHandled) * 100.0 / wz.MaxCapacity) AS UtilizationPct
    FROM FactWarehouseOperations wo
    JOIN DimWarehouseZone wz ON wo.WarehouseZoneKey = wz.ZoneKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY wz.ZoneName, wz.MaxCapacity
),
HistoricalAvg AS (
    SELECT 
        wz.ZoneName,
        AVG(wo.DwellTimeMinutes) AS HistoricalAvgDwell,
        AVG(SUM(wo.QuantityHandled) * 100.0 / wz.MaxCapacity) AS HistoricalAvgUtilization
    FROM FactWarehouseOperations wo
    JOIN DimWarehouseZone wz ON wo.WarehouseZoneKey = wz.ZoneKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Date BETWEEN DATEADD(DAY, -90, GETDATE()) AND DATEADD(DAY, -8, GETDATE())
    GROUP BY wz.ZoneName
)
SELECT 
    cm.ZoneName,
    cm.ActiveSKUs,
    cm.AvgDwellTime,
    ha.HistoricalAvgDwell,
    (cm.AvgDwellTime - ha.HistoricalAvgDwell) / NULLIF(ha.HistoricalAvgDwell, 0) AS DwellTimeGrowthPct,
    cm.UtilizationPct,
    ha.HistoricalAvgUtilization,
    (cm.UtilizationPct - ha.HistoricalAvgUtilization) AS UtilizationDelta,
    -- Bottleneck risk score
    (
        ((cm.AvgDwellTime - ha.HistoricalAvgDwell) / NULLIF(ha.HistoricalAvgDwell, 0) * 40) +
        (cm.UtilizationPct * 0.6)
    ) AS BottleneckRiskScore
FROM CurrentMetrics cm
JOIN HistoricalAvg ha ON cm.ZoneName = ha.ZoneName
WHERE (cm.AvgDwellTime - ha.HistoricalAvgDwell) / NULLIF(ha.HistoricalAvgDwell, 0) > 0.15
   OR cm.UtilizationPct > 85;
```

### Fleet Maintenance Priority Queue

```sql
-- Proactive maintenance queue based on telemetry and revenue impact
CREATE PROCEDURE usp_GenerateMaintenancePriorityQueue
AS
BEGIN
    WITH VehicleHealth AS (
        SELECT 
            v.VehicleKey,
            v.VehicleLicensePlate,
            v.VehicleType,
            AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelEfficiency,
            SUM(ft.IdleTimeMinutes) AS TotalIdleTime,
            COUNT(*) AS TripCount,
            MAX(t.FullDateTime) AS LastTripDate
        FROM FactFleetTrips ft
        JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
        JOIN DimTime t ON ft.TimeEndKey = t.TimeKey
        WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
        GROUP BY v.VehicleKey, v.VehicleLicensePlate, v.VehicleType
    ),
    RevenueImpact AS (
        SELECT 
            ft.VehicleKey,
            SUM(o.OrderValue) AS TotalRevenueHauled
        FROM FactFleetTrips ft
        JOIN BridgeTrip_Order bto ON ft.TripKey = bto.TripKey
        JOIN FactOrders o ON bto.OrderKey = o.OrderKey
        JOIN DimTime t ON ft.TimeEndKey = t.TimeKey
        WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ft.VehicleKey
    )
    SELECT 
        vh.VehicleKey,
        vh.VehicleLicensePlate,
        vh.VehicleType,
        vh.AvgFuelEfficiency,
        v.BaselineFuelEfficiency,
        ((vh.AvgFuelEfficiency - v.BaselineFuelEfficiency) / NULLIF(v.BaselineFuelEfficiency, 0) * 100) AS EfficiencyDegradationPct,
        vh.TotalIdleTime,
        vh.LastTripDate,
        ri.TotalRevenueHauled,
        -- Priority score (higher = more urgent)
        (
            (((vh.AvgFuelEfficiency - v.BaselineFuelEfficiency) / NULLIF(v.BaselineFuelEfficiency, 0)) * 30) +
            ((vh.TotalIdleTime / 60.0) * 0.5) +
            ((ri.TotalRevenueHauled / 10000.0) * 20) +
            (CASE WHEN DATEDIFF(DAY, vh.LastTripDate, GETDATE()) < 2 THEN 25 ELSE 0 END) -- Recent usage bonus
        ) AS MaintenancePriorityScore
    FROM VehicleHealth vh
    JOIN DimVehicle v ON vh.VehicleKey = v.VehicleKey
    LEFT JOIN RevenueImpact ri ON vh.VehicleKey = ri.VehicleKey
    WHERE ((vh.AvgFuelEfficiency - v.BaselineFuelEfficiency) / NULLIF(v.BaselineFuelEfficiency, 0)) > 0.10
       OR vh.TotalIdleTime > 600
    ORDER BY MaintenancePriorityScore DESC;
END;
```

## Power BI Integration

### DAX Measures

**Cross-Fact KPI: Cost Per Unit Moved**
```dax
CostPerUnitMoved = 
VAR TotalFleetCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedLiters] * 1.5 + -- Fuel cost
        FactFleetTrips[IdleTimeMinutes] * 0.8 + -- Idle cost
        FactFleetTrips[DistanceKm] * 0.2 -- Depreciation cost
    )
VAR TotalUnitsHandled = 
    SUM(FactWarehouseOperations[QuantityHandled])
RETURN
    DIVIDE(TotalFleetCost, TotalUnitsHandled, 0)
```

**Warehouse Gravity Zone Compliance**
```dax
GravityZoneCompliance = 
VAR TotalSKUs = DISTINCTCOUNT(FactWarehouseOperations[ProductKey])
VAR CompliantSKUs = 
    CALCULATE(
        DISTINCTCOUNT(FactWarehouseOperations[ProductKey]),
        FILTER(
            DimProduct,
            DimProduct[OptimalZoneType] = RELATED(DimWarehouseZone[ZoneType])
        )
    )
RETURN
    DIVIDE(CompliantSKUs, TotalSKUs, 0)
```

**Temporal Elasticity: Projected Capacity Stress**
```dax
ProjectedCapacityStress = 
VAR CurrentUtilization = 
    DIVIDE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        SUM(DimWarehouseZone[MaxCapacity])
    )
VAR GrowthRate = 
    CALCULATE(
        DIVIDE(
            SUM(FactWarehouseOperations[QuantityHandled]),
            CALCULATE(
                SUM(FactWarehouseOperations[QuantityHandled]),
                DATEADD(DimTime[Date], -30, DAY)
            )
        ) - 1,
        DATESBETWEEN(DimTime[Date], TODAY()-60, TODAY())
    )
VAR ProjectedUtilization30Days = CurrentUtilization * (1 + GrowthRate)
RETURN
    IF(ProjectedUtilization30Days > 0.95, "CRITICAL",
        IF(ProjectedUtilization30Days > 0.85, "WARNING", "NORMAL")
    )
```

### Power BI Report Setup

1. **Create main dashboard pages:**
   - Executive Overview (high-level KPIs)
   - Warehouse Operations (zone heatmaps, dwell time trends)
   - Fleet Performance (fuel efficiency, route optimization)
   - Predictive Alerts (bottleneck forecasts, maintenance queue)

2. **Configure automatic refresh:**
```powerquery
// In Power Query Editor, set up incremental refresh
Table.SelectRows(
    FactWarehouseOperations,
    each [TimeKey] >= #"RangeStart" and [TimeKey] < #"RangeEnd"
)
```

3. **Implement row-level security:**
```dax
// In Manage Roles, create role for regional managers
[GeographyKey] = LOOKUPVALUE(
    DimUserAccess[GeographyKey],
    DimUserAccess[UserEmail], USERPRINCIPALNAME()
)
```

## Data Loading and ETL

### Incremental Load Pattern

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadFactWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Stage data from source system
    INSERT INTO Staging_WarehouseOperations (
        OperationTimestamp,
        ProductSKU,
        ZoneCode,
        OperationType,
        DwellTimeMinutes,
        QuantityHandled,
        OperatorID
    )
    SELECT 
        OperationTimestamp,
        ProductSKU,
        ZoneCode,
        OperationType,
        DwellTimeMinutes,
        QuantityHandled,
        OperatorID
    FROM OPENROWSET(
        'MSDASQL',
        '${WMS_ODBC_CONNECTION}',
        'SELECT * FROM wms.operations WHERE last_modified > ?',
        @LastLoadDateTime
    );
    
    -- Transform and load into fact table
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        ProductKey,
        WarehouseZoneKey,
        OperationType,
        DwellTimeMinutes,
        QuantityHandled,
        OperatorKey,
        CycleTimeSeconds
    )
    SELECT 
        CAST(FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        p.ProductKey,
        z.ZoneKey,
        s.OperationType,
        s.DwellTimeMinutes,
        s.QuantityHandled,
        op.OperatorKey,
        DATEDIFF(SECOND, s.OperationStartTime, s.OperationTimestamp) AS CycleTimeSeconds
    FROM Staging_WarehouseOperations s
    JOIN DimProduct p ON s.ProductSKU = p.ProductSKU
    JOIN DimWarehouseZone z ON s.ZoneCode = z.ZoneCode
    JOIN DimOperator op ON s.OperatorID = op.OperatorID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = CAST(FORMAT(s.OperationTimestamp, 'yyyyMMddHHmm') AS INT)
          AND f.ProductKey = p.ProductKey
          AND f.OperatorKey = op.OperatorKey
    );
    
    -- Clean up staging
    TRUNCATE TABLE Staging_WarehouseOperations;
    
    -- Update last load timestamp
    UPDATE ETL_Control
    SET LastLoadDateTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### External API Integration (Weather Data)

```sql
-- Populate weather dimension from external API
CREATE PROCEDURE usp_LoadWeatherData
AS
BEGIN
    DECLARE @JSON NVARCHAR(MAX);
    
    -- Call external weather API
    DECLARE @URL NVARCHAR(500) = '${WEATHER_API_ENDPOINT}/forecast?key=${WEATHER_API_KEY}&days=7';
    
    EXEC sp_OACreate 'MSXML2.ServerXMLHTTP', @Object OUT;
    EXEC sp_OAMethod @Object, 'open', NULL, 'GET', @URL, 'false';
    EXEC sp_OAMethod @Object, 'send';
    EXEC sp_OAMethod @Object, 'responseText', @JSON OUTPUT;
    
    -- Parse JSON and insert into dimension
    INSERT INTO DimWeatherCondition (Date, LocationKey, ConditionCode, Temperature, Precipitation, WindSpeed)
    SELECT 
        CAST(JSON_VALUE(value, '$.date') AS DATE),
        JSON_VALUE(value, '$.location_id'),
        JSON_VALUE(value, '$.condition'),
        CAST(JSON_VALUE(value, '$.temp_avg') AS DECIMAL(5,2)),
        CAST(JSON_VALUE(value, '$.precipitation_mm') AS DECIMAL(5,2)),
        CAST(JSON_VALUE(value, '$.wind_speed_kmh') AS DECIMAL(5,2))
    FROM OPENJSON(@JSON, '$.forecast') 
    WHERE NOT EXISTS (
        SELECT 1 FROM DimWeatherCondition w
        WHERE w.Date = CAST(JSON_VALUE(value, '$.date') AS DATE)
          AND w.LocationKey = JSON_VALUE(value, '$.location_id')
    );
END;
```

## Alerting and Notifications

### SQL Server Agent Job for Threshold Alerts

```sql
-- Create alert job for KPI breaches
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertSubject NVARCHAR(200);
    
    -- Check for excessive dwell time
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.Date = CAST(GETDATE() AS DATE)
          AND wo.DwellTimeMinutes > 72
          AND wo.OperationType = 'PUTAWAY'
        HAVING COUNT(*) > 10
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Excessive Warehouse Dwell Time Detected';
        SET @AlertMessage = 'More than 10 SKUs have exceeded 72-hour dwell time today. Review warehouse gravity zone assignments.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END;
    
    -- Check for fleet idle time threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips ft
        JOIN DimTime t ON ft.TimeEndKey = t.TimeKey
        WHERE t.Date >= DATEADD(DAY, -1, GETDATE())
        HAVING AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.IdleTimeMinutes + DATEDIFF(MINUTE, t.FullDateTime, GETDATE()), 0)) > 15
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Fleet Idle Time Exceeds Threshold';
        SET @AlertMessage = 'Average fleet idle time has exceeded 15% in the last 24 hours. Review route planning.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END;
END;

-- Schedule the job to run every 15 minutes
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_KPI_Monitor';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Monitor',
    @step_name = 'Check Thresholds',
    @command = 'EXEC usp_CheckKPIThresholds',
    @database_name = 'LogiFleetPulse';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
EXEC msdb.dbo.sp_attach_schedule 
    @job_name = 'LogiFleet_KPI_Monitor',
    @schedule_name = 'Every15Minutes';
EXEC msdb.dbo.sp_add_jobserver 
    @job_name = 'LogiFleet_KPI_Monitor';
```

## Troubleshooting

### Performance Optimization

**Issue: Slow cross-fact queries**
```sql
-- Add covering indexes on frequently joined columns
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (DwellTimeMinutes, QuantityHandled);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Time_Vehicle 
ON FactFleetTrips(TimeStartKey, VehicleKey) 
INCLUDE (IdleTimeMinutes, FuelConsumedLi
