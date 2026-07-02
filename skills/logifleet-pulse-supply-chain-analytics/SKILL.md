---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing solution for logistics, fleet management, and supply chain intelligence with multi-fact star schema.
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure warehouse and fleet data model"
  - "create Power BI logistics dashboard"
  - "build supply chain star schema in SQL Server"
  - "implement cross-fact KPI queries for logistics"
  - "deploy warehouse gravity zone analytics"
  - "configure real-time fleet telemetry tracking"
  - "set up logistics intelligence data warehouse"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

LogiFleet Pulse is a comprehensive data warehousing and business intelligence solution for logistics and supply chain management. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, inventory, and external data
- **Power BI dashboards** with real-time (15-minute refresh) logistics KPIs
- **Cross-modal analytics** that unify warehouse velocity, fleet performance, and demand patterns
- **Predictive bottleneck detection** using historical multi-fact correlations
- **Warehouse Gravity Zones** for optimal spatial allocation based on velocity and value

The system integrates data from WMS, telematics/GPS, supplier portals, weather APIs, and order management systems into a unified semantic layer.

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- Access to source systems (WMS, TMS, ERP) via SQL, REST API, or flat files

### Deployment Steps

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema**:
```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the provided schema script
-- (typically named schema_deployment.sql or similar)
```

3. **Configure data source connections**:
```json
-- Edit config_sample.json with your connection details
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "wms_api": {
    "endpoint": "${WMS_API_ENDPOINT}",
    "api_key": "${WMS_API_KEY}"
  },
  "telematics_api": {
    "endpoint": "${TELEMATICS_API_ENDPOINT}",
    "api_key": "${TELEMATICS_API_KEY}"
  }
}
```

4. **Import Power BI template**:
- Open Power BI Desktop
- File → Import → Power BI Template (.pbit)
- Select `LogiFleet_Pulse_Master.pbit`
- Enter SQL Server connection parameters when prompted

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity metrics:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimWarehouse(WarehouseKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    DwellTimeMinutes INT,
    QuantityHandled DECIMAL(18,2),
    CycleTimeSeconds INT,
    GravityZone VARCHAR(20),
    EmployeeKey INT FOREIGN KEY REFERENCES DimEmployee(EmployeeKey)
);

CREATE INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
```

**FactFleetTrips** - Fleet telemetry and trip data:
```sql
CREATE TABLE FactFleetTrips (
    TripID INT PRIMARY KEY IDENTITY(1,1),
    StartTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleKey INT FOREIGN KEY REFERENCES DimVehicle(VehicleKey),
    RouteKey INT FOREIGN KEY REFERENCES DimRoute(RouteKey),
    DriverKey INT FOREIGN KEY REFERENCES DimDriver(DriverKey),
    TotalDistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    WeatherConditionKey INT FOREIGN KEY REFERENCES DimWeather(WeatherKey),
    DelayMinutes INT
);

CREATE INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey);
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleKey);
```

**FactCrossDock** - Cross-dock transfer operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockID INT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    InboundShipmentKey INT,
    OutboundShipmentKey INT,
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    QuantityTransferred DECIMAL(18,2),
    TransferTimeMinutes INT,
    DockLocation VARCHAR(50)
);
```

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME,
    DateOnly DATE,
    TimeOnly TIME,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfWeek INT,
    DayName VARCHAR(20),
    HourOfDay INT,
    MinuteBucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    IsHoliday BIT,
    FiscalPeriod VARCHAR(20)
);

CREATE INDEX IX_DimTime_Date ON DimTime(DateOnly);
CREATE INDEX IX_DimTime_Hour ON DimTime(HourOfDay);
```

**DimProductGravity** - Product classification with gravity scoring:
```sql
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,4),
    IsFragile BIT,
    IsPerishable BIT,
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    OptimalGravityZone VARCHAR(20), -- 'High', 'Medium', 'Low'
    AverageDwellTimeHours DECIMAL(10,2),
    TurnoverRate DECIMAL(10,4)
);

CREATE INDEX IX_DimProduct_Gravity ON DimProduct(GravityScore DESC);
CREATE INDEX IX_DimProduct_Category ON DimProduct(Category);
```

**DimWarehouse** - Warehouse locations with gravity zones:
```sql
CREATE TABLE DimWarehouse (
    WarehouseKey INT PRIMARY KEY IDENTITY(1,1),
    WarehouseCode VARCHAR(20) UNIQUE,
    WarehouseName VARCHAR(100),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TotalCapacityCubicMeters DECIMAL(18,2),
    CurrentUtilizationPercent DECIMAL(5,2),
    HighGravityZoneCapacity DECIMAL(18,2),
    MediumGravityZoneCapacity DECIMAL(18,2),
    LowGravityZoneCapacity DECIMAL(18,2)
);
```

## Key SQL Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Correlate warehouse dwell time with fleet idle costs per route
WITH WarehouseDwell AS (
    SELECT 
        p.Category,
        AVG(wo.DwellTimeMinutes) AS AvgDwellMinutes,
        SUM(wo.QuantityHandled) AS TotalVolume
    FROM FactWarehouseOperations wo
    JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateOnly >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.Category
),
FleetIdle AS (
    SELECT 
        r.RouteType,
        AVG(ft.IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(ft.FuelConsumedLiters * 1.50) AS TotalFuelCost -- Assuming $1.50/liter
    FROM FactFleetTrips ft
    JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    WHERE t.DateOnly >= DATEADD(DAY, -30, GETDATE())
    GROUP BY r.RouteType
)
SELECT 
    wd.Category,
    wd.AvgDwellMinutes,
    fi.RouteType,
    fi.AvgIdleMinutes,
    fi.TotalFuelCost,
    -- Composite metric: waste ratio
    (wd.AvgDwellMinutes * fi.AvgIdleMinutes) / 
    NULLIF(wd.TotalVolume, 0) AS WasteRatio
FROM WarehouseDwell wd
CROSS JOIN FleetIdle fi
ORDER BY WasteRatio DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong gravity zones based on velocity changes
SELECT 
    p.SKU,
    p.ProductName,
    p.OptimalGravityZone AS CurrentZone,
    wo.GravityZone AS ActualZone,
    COUNT(*) AS OperationCount,
    AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
    CASE 
        WHEN p.GravityScore >= 80 THEN 'High'
        WHEN p.GravityScore >= 40 THEN 'Medium'
        ELSE 'Low'
    END AS RecommendedZone,
    CASE 
        WHEN p.OptimalGravityZone <> wo.GravityZone THEN 'MISMATCH'
        ELSE 'CORRECT'
    END AS Status
FROM FactWarehouseOperations wo
JOIN DimProduct p ON wo.ProductKey = p.ProductKey
JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.DateOnly >= DATEADD(DAY, -7, GETDATE())
    AND wo.OperationType = 'Picking'
GROUP BY p.SKU, p.ProductName, p.OptimalGravityZone, 
         wo.GravityZone, p.GravityScore
HAVING p.OptimalGravityZone <> wo.GravityZone
ORDER BY AvgCycleTime DESC;
```

### Predictive Bottleneck Index

```sql
-- Calculate bottleneck probability based on historical patterns
CREATE PROCEDURE sp_CalculateBottleneckIndex
AS
BEGIN
    WITH TimePatterns AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            COUNT(DISTINCT wo.OperationID) AS OperationVolume,
            AVG(wo.CycleTimeSeconds) AS AvgCycleTime,
            STDEV(wo.CycleTimeSeconds) AS CycleTimeVariance
        FROM FactWarehouseOperations wo
        JOIN DimTime t ON wo.TimeKey = t.TimeKey
        WHERE t.DateOnly >= DATEADD(DAY, -90, GETDATE())
        GROUP BY t.HourOfDay, t.DayOfWeek
    ),
    FleetPatterns AS (
        SELECT 
            t.HourOfDay,
            t.DayOfWeek,
            COUNT(DISTINCT ft.TripID) AS TripCount,
            AVG(ft.DelayMinutes) AS AvgDelay,
            SUM(CASE WHEN ft.DelayMinutes > 30 THEN 1 ELSE 0 END) AS SevereDelayCount
        FROM FactFleetTrips ft
        JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
        WHERE t.DateOnly >= DATEADD(DAY, -90, GETDATE())
        GROUP BY t.HourOfDay, t.DayOfWeek
    )
    SELECT 
        tp.HourOfDay,
        tp.DayOfWeek,
        tp.OperationVolume,
        tp.AvgCycleTime,
        fp.TripCount,
        fp.AvgDelay,
        -- Bottleneck Index: weighted score 0-100
        (
            (tp.CycleTimeVariance / NULLIF(tp.AvgCycleTime, 0) * 30) +
            (fp.AvgDelay / 60.0 * 40) +
            (CAST(fp.SevereDelayCount AS FLOAT) / NULLIF(fp.TripCount, 0) * 30)
        ) AS BottleneckIndex,
        CASE 
            WHEN (
                (tp.CycleTimeVariance / NULLIF(tp.AvgCycleTime, 0) * 30) +
                (fp.AvgDelay / 60.0 * 40) +
                (CAST(fp.SevereDelayCount AS FLOAT) / NULLIF(fp.TripCount, 0) * 30)
            ) >= 70 THEN 'HIGH RISK'
            WHEN (
                (tp.CycleTimeVariance / NULLIF(tp.AvgCycleTime, 0) * 30) +
                (fp.AvgDelay / 60.0 * 40) +
                (CAST(fp.SevereDelayCount AS FLOAT) / NULLIF(fp.TripCount, 0) * 30)
            ) >= 40 THEN 'MODERATE RISK'
            ELSE 'LOW RISK'
        END AS RiskLevel
    FROM TimePatterns tp
    JOIN FleetPatterns fp ON tp.HourOfDay = fp.HourOfDay 
                          AND tp.DayOfWeek = fp.DayOfWeek
    ORDER BY BottleneckIndex DESC;
END;
GO
```

### Adaptive Fleet Triage

```sql
-- Generate maintenance priority queue weighted by revenue impact
SELECT 
    v.VehicleCode,
    v.VehicleMake,
    v.VehicleModel,
    v.LastMaintenanceDate,
    DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) AS DaysSinceMaintenance,
    SUM(ft.TotalDistanceKM) AS TotalDistanceLast30Days,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    COUNT(CASE WHEN ft.DelayMinutes > 30 THEN 1 END) AS DelayIncidents,
    -- Calculate revenue at risk (assuming avg margin per km)
    SUM(ft.TotalDistanceKM) * 2.50 AS EstimatedRevenue,
    -- Priority score
    (
        (DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) / 30.0 * 30) +
        (AVG(ft.IdleTimeMinutes) / 60.0 * 20) +
        (COUNT(CASE WHEN ft.DelayMinutes > 30 THEN 1 END) * 5) +
        (CASE WHEN v.CargoType = 'Perishable' THEN 25 ELSE 0 END)
    ) AS MaintenancePriorityScore
FROM DimVehicle v
JOIN FactFleetTrips ft ON v.VehicleKey = ft.VehicleKey
JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
WHERE t.DateOnly >= DATEADD(DAY, -30, GETDATE())
GROUP BY v.VehicleCode, v.VehicleMake, v.VehicleModel, 
         v.LastMaintenanceDate, v.CargoType
HAVING DATEDIFF(DAY, v.LastMaintenanceDate, GETDATE()) > 60
    OR AVG(ft.IdleTimeMinutes) > 30
ORDER BY MaintenancePriorityScore DESC;
```

## Data Loading Procedures

### Incremental ETL for Warehouse Operations

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDate DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last successful load or 1 day ago
    IF @LastLoadDate IS NULL
        SELECT @LastLoadDate = ISNULL(MAX(LoadDate), DATEADD(DAY, -1, GETDATE()))
        FROM ETL_LoadLog 
        WHERE TableName = 'FactWarehouseOperations' AND Status = 'SUCCESS';
    
    -- Stage data from WMS source
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        DwellTimeMinutes, QuantityHandled, CycleTimeSeconds,
        GravityZone, EmployeeKey
    )
    SELECT 
        dt.TimeKey,
        dw.WarehouseKey,
        dp.ProductKey,
        src.OperationType,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        src.Quantity,
        DATEDIFF(SECOND, src.StartTime, src.EndTime) AS CycleTimeSeconds,
        dp.OptimalGravityZone,
        de.EmployeeKey
    FROM WMS_Operations src -- External table or linked server
    JOIN DimTime dt ON CAST(src.StartTime AS DATE) = dt.DateOnly
                    AND DATEPART(HOUR, src.StartTime) = dt.HourOfDay
                    AND (DATEPART(MINUTE, src.StartTime) / 15) * 15 = dt.MinuteBucket
    JOIN DimWarehouse dw ON src.WarehouseCode = dw.WarehouseCode
    JOIN DimProduct dp ON src.SKU = dp.SKU
    LEFT JOIN DimEmployee de ON src.EmployeeID = de.EmployeeID
    WHERE src.StartTime > @LastLoadDate
        AND src.EndTime IS NOT NULL;
    
    -- Log successful load
    INSERT INTO ETL_LoadLog (TableName, LoadDate, RowsInserted, Status)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT, 'SUCCESS');
END;
GO
```

### Real-Time Telematics Ingestion

```sql
CREATE PROCEDURE sp_LoadFleetTelemetry
AS
BEGIN
    -- Insert from external telematics API (via external table or staging)
    INSERT INTO FactFleetTrips (
        StartTimeKey, EndTimeKey, VehicleKey, RouteKey, DriverKey,
        TotalDistanceKM, FuelConsumedLiters, IdleTimeMinutes,
        LoadingTimeMinutes, UnloadingTimeMinutes, WeatherConditionKey, DelayMinutes
    )
    SELECT 
        tStart.TimeKey AS StartTimeKey,
        tEnd.TimeKey AS EndTimeKey,
        dv.VehicleKey,
        dr.RouteKey,
        dd.DriverKey,
        tel.TotalDistance,
        tel.FuelUsed,
        tel.IdleTime,
        tel.LoadTime,
        tel.UnloadTime,
        dw.WeatherKey,
        DATEDIFF(MINUTE, tel.ScheduledArrival, tel.ActualArrival) AS DelayMinutes
    FROM Telematics_Staging tel
    JOIN DimVehicle dv ON tel.VehicleID = dv.VehicleCode
    JOIN DimRoute dr ON tel.RouteID = dr.RouteCode
    JOIN DimDriver dd ON tel.DriverID = dd.DriverCode
    JOIN DimTime tStart ON CAST(tel.TripStart AS DATE) = tStart.DateOnly
                        AND DATEPART(HOUR, tel.TripStart) = tStart.HourOfDay
    JOIN DimTime tEnd ON CAST(tel.TripEnd AS DATE) = tEnd.DateOnly
                      AND DATEPART(HOUR, tel.TripEnd) = tEnd.HourOfDay
    LEFT JOIN DimWeather dw ON tel.WeatherCondition = dw.WeatherDescription
    WHERE tel.Processed = 0;
    
    -- Mark staging records as processed
    UPDATE Telematics_Staging SET Processed = 1
    WHERE Processed = 0;
END;
GO
```

## Power BI Configuration

### DAX Measures for Cross-Fact Analysis

**Total Warehouse Efficiency Score**:
```dax
Warehouse Efficiency Score = 
VAR AvgCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeSeconds])
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR BaselineEfficiency = 100
VAR CycleTimePenalty = (AvgCycleTime - 300) / 10  // Baseline 300 seconds
VAR DwellTimePenalty = (AvgDwellTime - 120) / 30  // Baseline 120 minutes
RETURN 
    MAX(0, BaselineEfficiency - CycleTimePenalty - DwellTimePenalty)
```

**Fleet Cost per Delivery**:
```dax
Cost Per Delivery = 
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLiters]) * 1.50 +  // Fuel cost
    SUM(FactFleetTrips[TotalDistanceKM]) * 0.35 +     // Maintenance cost per km
    SUM(FactFleetTrips[IdleTimeMinutes]) * 0.80,      // Idle cost per minute
    COUNTROWS(FactFleetTrips),
    0
)
```

**Cross-Fact Correlation: Gravity Zone Impact**:
```dax
Gravity Zone ROI = 
VAR HighGravityPickTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[CycleTimeSeconds]),
        DimProduct[OptimalGravityZone] = "High"
    )
VAR LowGravityPickTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[CycleTimeSeconds]),
        DimProduct[OptimalGravityZone] = "Low"
    )
VAR TimeSavingsPercent = 
    DIVIDE(LowGravityPickTime - HighGravityPickTime, LowGravityPickTime, 0)
RETURN 
    TimeSavingsPercent * 100
```

### Power BI Report Structure

1. **Executive Dashboard**:
   - KPI cards: Total shipments, on-time delivery %, warehouse utilization
   - Trend lines: Daily operations volume, fuel costs, delays
   - Heatmap: Bottleneck index by hour and day of week

2. **Warehouse Operations**:
   - Bar chart: Operations by gravity zone
   - Scatter plot: Dwell time vs. cycle time by product category
   - Table: Top 20 products by operation volume with gravity recommendations

3. **Fleet Performance**:
   - Map visual: Real-time vehicle locations (via DirectQuery to live telemetry)
   - Line chart: Fuel consumption trend by route type
   - Matrix: Maintenance priority queue

4. **Cross-Modal Analysis**:
   - Combined area chart: Warehouse operations (left axis) vs. fleet trips (right axis)
   - Drill-through: Click any product → see full warehouse-to-delivery journey

### Scheduled Refresh Configuration

```powershell
# Power BI Service API - Schedule dataset refresh
$apiUrl = "https://api.powerbi.com/v1.0/myorg/groups/${WORKSPACE_ID}/datasets/${DATASET_ID}/refreshes"
$headers = @{
    "Authorization" = "Bearer ${POWERBI_ACCESS_TOKEN}"
    "Content-Type" = "application/json"
}

$body = @{
    "notifyOption" = "MailOnFailure"
} | ConvertTo-Json

Invoke-RestMethod -Uri $apiUrl -Method Post -Headers $headers -Body $body
```

## Alerting & Monitoring

### SQL Agent Job for KPI Threshold Alerts

```sql
CREATE PROCEDURE sp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check fleet idle time threshold (>15% of trip duration)
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips ft
        JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
        WHERE t.DateOnly = CAST(GETDATE() AS DATE)
            AND (CAST(ft.IdleTimeMinutes AS FLOAT) / 
                 NULLIF(ft.IdleTimeMinutes + ft.LoadingTimeMinutes + ft.UnloadingTimeMinutes, 0)) > 0.15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% threshold today.';
        
        -- Send email via Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Pulse - Idle Time Alert',
            @body = @AlertMessage;
    END;
    
    -- Check warehouse utilization threshold (>85%)
    IF EXISTS (
        SELECT 1 
        FROM DimWarehouse 
        WHERE CurrentUtilizationPercent > 85
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse capacity exceeded 85% threshold.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL}',
            @subject = 'LogiFleet Pulse - Capacity Alert',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule the job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_KPI_Monitor',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_KPI_Monitor',
    @step_name = 'CheckThresholds',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_MonitorKPIThresholds;',
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

## Common Patterns

### Pattern: Time-Intelligence Analysis

```sql
-- Compare current period vs. same period last year
WITH CurrentPeriod AS (
    SELECT 
        SUM(QuantityHandled) AS TotalVolume,
        AVG(CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations wo
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Year = YEAR(GETDATE()) 
        AND t.Month = MONTH(GETDATE())
),
PriorYearPeriod AS (
    SELECT 
        SUM(QuantityHandled) AS TotalVolume,
        AVG(CycleTimeSeconds) AS AvgCycleTime
    FROM FactWarehouseOperations wo
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.Year = YEAR(DATEADD(YEAR, -1, GETDATE()))
        AND t.Month = MONTH(GETDATE())
)
SELECT 
    cp.TotalVolume AS CurrentVolume,
    py.TotalVolume AS PriorYearVolume,
    ((cp.TotalVolume - py.TotalVolume) / NULLIF(py.TotalVolume, 0.0)) * 100 AS VolumeGrowthPercent,
    cp.AvgCycleTime AS CurrentCycleTime,
    py.AvgCycleTime AS PriorYearCycleTime,
    ((cp.AvgCycleTime - py.AvgCycleTime) / NULLIF(py.AvgCycleTime, 0.0)) * 100 AS CycleTimeChangePercent
FROM CurrentPeriod cp
CROSS JOIN PriorYearPeriod py;
```

### Pattern: Role-Playing Dimension (Time)

```sql
-- Use DimTime for both trip start and trip end
SELECT 
    tStart.DateOnly AS TripDate,
    tStart.HourOfDay AS DepartureHour,
    tEnd.HourOfDay AS ArrivalHour,
    COUNT(ft.TripID) AS TripCount,
    AVG(ft.TotalDistanceKM) AS AvgDistance
FROM FactFleetTrips ft
JOIN DimTime tStart ON ft.StartTimeKey = tStart.TimeKey
JOIN DimTime tEnd ON ft.EndTimeKey = tEnd.TimeKey
WHERE tStart.DateOnly >= DATEADD(DAY, -7, GETDATE())
GROUP BY tStart.DateOnly, tStart.HourOfDay, tEnd.HourOfDay
ORDER BY tStart.DateOnly, tStart.HourOfDay;
```

### Pattern: Bridge Table for Many-to-Many

```sql
-- Products can move through multiple warehouses
CREATE TABLE BridgeProductWarehouse (
    ProductKey INT FOREIGN KEY REFERENCES DimProduct(ProductKey),
    WarehouseKey INT FOREIGN KEY REFERENCES DimWarehouse(WarehouseKey),
    AllocationPercent DECIMAL(5,2),
    EffectiveDate DATE,
    ExpirationDate DATE
);

-- Query using bridge
SELECT 
    p.SKU,
    w.WarehouseName,
    b.AllocationPercent,
    SUM(wo.
