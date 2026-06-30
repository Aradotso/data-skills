---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence platform with MS SQL Server data warehousing and Power BI dashboards for fleet, warehouse, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure logistics data warehouse with SQL Server"
  - "build Power BI dashboard for fleet management"
  - "implement warehouse gravity zone optimization"
  - "create cross-fact KPI queries for logistics"
  - "deploy real-time supply chain monitoring"
  - "integrate fleet telemetry with warehouse operations"
  - "model logistics data with star schema"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and supply chain analytics into a single semantic data layer. Built on MS SQL Server with Power BI visualization, it provides:

- Multi-fact star schema for cross-domain KPI harmonization
- Real-time fleet and warehouse monitoring (15-minute refresh cycles)
- Predictive bottleneck detection and maintenance triage
- Warehouse Gravity Zones™ for spatial optimization
- Time-phased dimension modeling for temporal analysis

**Core Components:**
- SQL Server database schema (multi-fact star design)
- Power BI template (`.pbit`) with pre-built dashboards
- Stored procedures for incremental data loading
- Role-based security layer
- Automated alerting system

## Installation

### Prerequisites

- MS SQL Server 2019+ (Enterprise, Standard, or Developer edition)
- Power BI Desktop (latest version)
- Appropriate access to data sources (WMS, TMS, telemetry APIs)

### Database Setup

1. **Clone or download the repository:**
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**
```sql
-- Connect to your SQL Server instance and create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema script (typically schema.sql or setup.sql)
-- Adjust file path as needed
:r .\sql\schema.sql
```

3. **Configure connection strings:**
```json
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "SQL",
    "username": "${SQL_USERNAME}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wmsApi": "${WMS_API_ENDPOINT}",
    "fleetTelemetry": "${FLEET_TELEMETRY_ENDPOINT}",
    "weatherApi": "${WEATHER_API_KEY}"
  }
}
```

4. **Initialize dimension tables:**
```sql
-- Execute dimension seeding scripts
EXEC [dbo].[sp_InitializeDimensions];
EXEC [dbo].[sp_PopulateDimTime] @StartDate='2024-01-01', @EndDate='2027-12-31';
```

### Power BI Configuration

1. Open `LogiFleet_Pulse_Master.pbit`
2. Enter SQL Server connection details when prompted
3. Configure refresh schedule (minimum 15 minutes)
4. Publish to Power BI Service workspace (if using cloud)

## Data Model Architecture

### Core Fact Tables

**FactWarehouseOperations** - Captures all warehouse activities:
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseZoneKey INT NOT NULL,
    OperationType VARCHAR(50), -- 'Putaway', 'Pick', 'Pack', 'Ship'
    DwellTimeMinutes INT,
    PickCycleSeconds INT,
    EmployeeKey INT,
    CONSTRAINT FK_WH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WH_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Index for common query patterns
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time_Product 
ON FactWarehouseOperations(TimeKey, ProductKey) 
INCLUDE (OperationType, DwellTimeMinutes);
```

**FactFleetTrips** - Fleet telemetry and route data:
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripStartTimeKey INT NOT NULL,
    TripEndTimeKey INT,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT,
    DestinationGeographyKey INT,
    TotalDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKg DECIMAL(10,2),
    DriverKey INT,
    CONSTRAINT FK_Fleet_StartTime FOREIGN KEY (TripStartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);
```

**FactCrossDock** - Cross-docking operations:
```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTimeKey INT NOT NULL,
    OutboundTimeKey INT,
    ProductKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TransferDurationMinutes INT,
    QuantityUnits INT,
    CONSTRAINT FK_CD_Inbound FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey)
);
```

### Dimension Tables

**DimTime** - 15-minute granularity time dimension:
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    DateKey INT,
    TimeOfDay TIME,
    Hour15MinuteBucket VARCHAR(10), -- '00:00', '00:15', '00:30', '00:45'
    HourOfDay INT,
    DayOfWeek VARCHAR(10),
    IsWeekend BIT,
    FiscalPeriod VARCHAR(10),
    FiscalYear INT
);
```

**DimProductGravity** - Product classification with gravity scoring:
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    VelocityScore DECIMAL(5,2), -- Pick frequency normalized
    ValueScore DECIMAL(5,2), -- Unit price normalized
    FragilityScore DECIMAL(5,2), -- Handling risk score
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZoneType VARCHAR(50) -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
);
```

**DimSupplierReliability** - Supplier performance metrics:
```sql
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY,
    SupplierName VARCHAR(200),
    LeadTimeDaysAvg DECIMAL(6,2),
    LeadTimeVariance DECIMAL(6,2),
    DefectRatePercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100 scale
    ReliabilityTier VARCHAR(20) -- 'Platinum', 'Gold', 'Silver', 'Bronze'
);
```

## Key Queries and Patterns

### Cross-Fact KPI: Dwell Time vs. Fleet Idle Time

```sql
-- Correlate warehouse dwell time with downstream fleet performance
WITH WarehouseDwell AS (
    SELECT 
        dt.DateKey,
        dp.Category,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTimeMin,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(day, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY dt.DateKey, dp.Category
),
FleetIdle AS (
    SELECT 
        dt.DateKey,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTimeMin,
        AVG(CAST(fft.IdleTimeMinutes AS FLOAT) / NULLIF(DATEDIFF(MINUTE, t1.FullDateTime, t2.FullDateTime), 0) * 100) AS IdlePercentage
    FROM FactFleetTrips fft
    INNER JOIN DimTime t1 ON fft.TripStartTimeKey = t1.TimeKey
    LEFT JOIN DimTime t2 ON fft.TripEndTimeKey = t2.TimeKey
    INNER JOIN DimTime dt ON t1.DateKey = dt.DateKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(day, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY dt.DateKey
)
SELECT 
    wd.DateKey,
    wd.Category,
    wd.AvgDwellTimeMin,
    fi.AvgIdleTimeMin,
    fi.IdlePercentage,
    CASE 
        WHEN wd.AvgDwellTimeMin > 180 AND fi.IdlePercentage > 15 THEN 'Critical'
        WHEN wd.AvgDwellTimeMin > 120 OR fi.IdlePercentage > 12 THEN 'Warning'
        ELSE 'Normal'
    END AS AlertLevel
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.DateKey = fi.DateKey
ORDER BY wd.DateKey DESC, wd.Category;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in wrong gravity zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.RecommendedZoneType,
    dz.ZoneName AS CurrentZone,
    dz.ZoneType AS CurrentZoneType,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS PickCount,
    CASE 
        WHEN dp.RecommendedZoneType = 'High-Gravity' AND dz.ZoneType != 'High-Gravity' 
            THEN 'RELOCATE TO HIGH-GRAVITY'
        WHEN dp.RecommendedZoneType = 'Low-Gravity' AND dz.ZoneType = 'High-Gravity' 
            THEN 'RELOCATE TO LOW-GRAVITY'
        ELSE 'OPTIMAL'
    END AS RecommendedAction
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN DimWarehouseZone dz ON fwo.WarehouseZoneKey = dz.ZoneKey
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.DateKey >= CAST(FORMAT(DATEADD(day, -7, GETDATE()), 'yyyyMMdd') AS INT)
    AND fwo.OperationType = 'Pick'
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.RecommendedZoneType, 
         dz.ZoneName, dz.ZoneType
HAVING COUNT(*) >= 10 -- Minimum sample size
    AND dp.RecommendedZoneType != dz.ZoneType
ORDER BY dp.GravityScore DESC;
```

### Predictive Bottleneck Detection

```sql
-- Create stored procedure for real-time bottleneck scoring
CREATE OR ALTER PROCEDURE sp_CalculateBottleneckIndex
    @ForecastHours INT = 4
AS
BEGIN
    -- Score based on multiple factors: queue depth, resource utilization, historical patterns
    WITH CurrentState AS (
        SELECT 
            dz.ZoneName,
            COUNT(DISTINCT fwo.OperationKey) AS ActiveOperations,
            AVG(fwo.DwellTimeMinutes) AS CurrentAvgDwell,
            MAX(fwo.DwellTimeMinutes) AS MaxDwell
        FROM FactWarehouseOperations fwo
        INNER JOIN DimWarehouseZone dz ON fwo.WarehouseZoneKey = dz.ZoneKey
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(MINUTE, -30, GETDATE())
            AND fwo.OperationType IN ('Putaway', 'Pick')
        GROUP BY dz.ZoneName
    ),
    HistoricalBaseline AS (
        SELECT 
            dz.ZoneName,
            AVG(fwo.DwellTimeMinutes) AS HistoricalAvgDwell,
            STDEV(fwo.DwellTimeMinutes) AS DwellStdDev
        FROM FactWarehouseOperations fwo
        INNER JOIN DimWarehouseZone dz ON fwo.WarehouseZoneKey = dz.ZoneKey
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(DAY, -30, GETDATE())
            AND dt.FullDateTime < DATEADD(DAY, -1, GETDATE())
            AND DATEPART(HOUR, dt.FullDateTime) = DATEPART(HOUR, GETDATE())
        GROUP BY dz.ZoneName
    )
    SELECT 
        cs.ZoneName,
        cs.ActiveOperations,
        cs.CurrentAvgDwell,
        hb.HistoricalAvgDwell,
        -- Bottleneck Index: weighted score 0-100
        CASE 
            WHEN hb.DwellStdDev = 0 THEN 0
            ELSE CAST(
                ((cs.CurrentAvgDwell - hb.HistoricalAvgDwell) / NULLIF(hb.DwellStdDev, 0)) * 20 +
                (cs.ActiveOperations * 2) +
                ((cs.MaxDwell / NULLIF(cs.CurrentAvgDwell, 0)) * 10)
            AS INT)
        END AS BottleneckIndex,
        CASE 
            WHEN ((cs.CurrentAvgDwell - hb.HistoricalAvgDwell) / NULLIF(hb.DwellStdDev, 0)) > 2 
                THEN 'CRITICAL - Immediate intervention required'
            WHEN ((cs.CurrentAvgDwell - hb.HistoricalAvgDwell) / NULLIF(hb.DwellStdDev, 0)) > 1.5 
                THEN 'WARNING - Monitor closely'
            ELSE 'NORMAL'
        END AS AlertStatus
    FROM CurrentState cs
    LEFT JOIN HistoricalBaseline hb ON cs.ZoneName = hb.ZoneName
    ORDER BY BottleneckIndex DESC;
END;
GO

-- Execute for real-time monitoring
EXEC sp_CalculateBottleneckIndex @ForecastHours = 4;
```

### Fleet Maintenance Triage

```sql
-- Prioritize vehicle maintenance by revenue impact
WITH VehicleHealth AS (
    SELECT 
        dv.VehicleID,
        dv.VehicleType,
        dv.TirePressurePSI,
        dv.EngineDiagnosticCode,
        dv.LastMaintenanceDate,
        DATEDIFF(DAY, dv.LastMaintenanceDate, GETDATE()) AS DaysSinceMaintenance
    FROM DimVehicle dv
    WHERE dv.IsActive = 1
),
RevenueImpact AS (
    SELECT 
        fft.VehicleKey,
        AVG(fft.LoadWeightKg * dp.ValueScore) AS AvgCargoValue,
        COUNT(*) AS TripsLast30Days,
        SUM(CASE WHEN fft.IdleTimeMinutes > 60 THEN 1 ELSE 0 END) AS HighIdleTrips
    FROM FactFleetTrips fft
    INNER JOIN FactCrossDock fcd ON fft.TripKey = fcd.OutboundTripKey
    INNER JOIN DimProductGravity dp ON fcd.ProductKey = dp.ProductKey
    INNER JOIN DimTime dt ON fft.TripStartTimeKey = dt.TimeKey
    WHERE dt.DateKey >= CAST(FORMAT(DATEADD(day, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY fft.VehicleKey
)
SELECT 
    vh.VehicleID,
    vh.VehicleType,
    vh.DaysSinceMaintenance,
    ri.AvgCargoValue,
    ri.TripsLast30Days,
    -- Triage Priority Score
    (
        (CASE WHEN vh.TirePressurePSI < 32 THEN 30 ELSE 0 END) +
        (CASE WHEN vh.EngineDiagnosticCode IS NOT NULL THEN 40 ELSE 0 END) +
        (CASE WHEN vh.DaysSinceMaintenance > 90 THEN 20 ELSE 0 END) +
        (ri.HighIdleTrips * 2)
    ) * (ri.AvgCargoValue / 10000.0) AS PriorityScore,
    CASE 
        WHEN vh.EngineDiagnosticCode IS NOT NULL AND ri.AvgCargoValue > 50000 
            THEN 'P1 - Ground vehicle immediately'
        WHEN vh.TirePressurePSI < 30 OR vh.DaysSinceMaintenance > 120 
            THEN 'P2 - Schedule within 48 hours'
        WHEN vh.TirePressurePSI < 34 OR vh.DaysSinceMaintenance > 90 
            THEN 'P3 - Schedule within 1 week'
        ELSE 'P4 - Normal schedule'
    END AS TriageCategory
FROM VehicleHealth vh
INNER JOIN RevenueImpact ri ON vh.VehicleID = ri.VehicleKey
ORDER BY PriorityScore DESC;
```

## Incremental Data Loading

### Stored Procedure Pattern

```sql
-- Template for incremental ETL from external sources
CREATE OR ALTER PROCEDURE sp_IncrementalLoadFleetTrips
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last successful load timestamp
    IF @LastLoadDateTime IS NULL
        SELECT @LastLoadDateTime = MAX(LastLoadedDateTime) 
        FROM ETL_LoadLog 
        WHERE TableName = 'FactFleetTrips';
    
    -- Stage data from external table/API
    INSERT INTO FactFleetTrips_Staging (
        TripExternalID, VehicleExternalID, RouteID, 
        TripStartDateTime, TripEndDateTime,
        DistanceKm, FuelLiters, IdleMinutes, LoadKg
    )
    SELECT 
        trip_id, vehicle_id, route_id,
        start_time, end_time,
        distance_km, fuel_consumed, idle_time, load_weight
    FROM OPENROWSET(
        BULK '${FLEET_DATA_PATH}',
        DATA_SOURCE = 'ExternalFleetAPI',
        FORMAT = 'CSV',
        FIRSTROW = 2
    ) AS ExternalData
    WHERE start_time > @LastLoadDateTime;
    
    -- Transform and load with dimension lookups
    INSERT INTO FactFleetTrips (
        TripStartTimeKey, TripEndTimeKey, VehicleKey, RouteKey,
        TotalDistanceKm, FuelConsumedLiters, IdleTimeMinutes, LoadWeightKg
    )
    SELECT 
        dt_start.TimeKey,
        dt_end.TimeKey,
        dv.VehicleKey,
        dr.RouteKey,
        s.DistanceKm,
        s.FuelLiters,
        s.IdleMinutes,
        s.LoadKg
    FROM FactFleetTrips_Staging s
    INNER JOIN DimTime dt_start ON dt_start.FullDateTime = 
        DATEADD(MINUTE, (DATEPART(MINUTE, s.TripStartDateTime) / 15) * 15, 
                DATEADD(HOUR, DATEDIFF(HOUR, 0, s.TripStartDateTime), 0))
    LEFT JOIN DimTime dt_end ON dt_end.FullDateTime = 
        DATEADD(MINUTE, (DATEPART(MINUTE, s.TripEndDateTime) / 15) * 15, 
                DATEADD(HOUR, DATEDIFF(HOUR, 0, s.TripEndDateTime), 0))
    INNER JOIN DimVehicle dv ON dv.VehicleExternalID = s.VehicleExternalID
    INNER JOIN DimRoute dr ON dr.RouteExternalID = s.RouteID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactFleetTrips existing 
        WHERE existing.TripExternalID = s.TripExternalID
    );
    
    -- Log successful load
    INSERT INTO ETL_LoadLog (TableName, LastLoadedDateTime, RowsLoaded)
    VALUES ('FactFleetTrips', GETDATE(), @@ROWCOUNT);
    
    -- Clean staging table
    TRUNCATE TABLE FactFleetTrips_Staging;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact Composite KPIs

```dax
// Total Logistics Cost Per Unit Shipped
Total Logistics Cost Per Unit = 
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedLiters] * 1.5 + // Fuel cost
        FactFleetTrips[IdleTimeMinutes] * 0.8 // Idle cost per minute
    )
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * 0.3 // Storage cost per minute
    )
VAR TotalUnitsShipped = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[OperationType] = "Ship"
    )
RETURN
    DIVIDE(FleetCost + WarehouseCost, TotalUnitsShipped, 0)

// Warehouse Efficiency Index (0-100 scale)
Warehouse Efficiency Index = 
VAR ActualDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR OptimalDwellTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
        DimProductGravity[RecommendedZoneType] = "High-Gravity"
    )
VAR PickAccuracy = 
    DIVIDE(
        COUNTROWS(FILTER(FactWarehouseOperations, [PickErrors] = 0)),
        COUNTROWS(FactWarehouseOperations),
        1
    )
RETURN
    (DIVIDE(OptimalDwellTime, ActualDwellTime, 1) * 0.6 + PickAccuracy * 0.4) * 100

// Fleet Utilization Percentage
Fleet Utilization % = 
VAR ActiveTime = 
    SUMX(
        FactFleetTrips,
        DATEDIFF(
            RELATED(DimTime[FullDateTime]),
            RELATED(DimTime_End[FullDateTime]),
            MINUTE
        ) - FactFleetTrips[IdleTimeMinutes]
    )
VAR TotalAvailableTime = 
    COUNTROWS(FILTER(DimVehicle, DimVehicle[IsActive] = TRUE())) * 
    COUNTROWS(FILTER(DimTime, DimTime[DateKey] = MAX(DimTime[DateKey]))) * 15
RETURN
    DIVIDE(ActiveTime, TotalAvailableTime, 0) * 100
```

### Time Intelligence Patterns

```dax
// Rolling 7-Day Average Dwell Time
Dwell Time 7D MA = 
AVERAGEX(
    DATESINPERIOD(
        DimTime[DateKey],
        MAX(DimTime[FullDateTime]),
        -7,
        DAY
    ),
    CALCULATE(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]))
)

// Year-Over-Year Fleet Cost Comparison
Fleet Cost YoY % = 
VAR CurrentPeriodCost = [Total Fleet Cost]
VAR PriorYearCost = 
    CALCULATE(
        [Total Fleet Cost],
        SAMEPERIODLASTYEAR(DimTime[FullDateTime])
    )
RETURN
    DIVIDE(CurrentPeriodCost - PriorYearCost, PriorYearCost, 0)
```

## Automated Alerting System

### SQL Server Alert Configuration

```sql
-- Create alert threshold monitoring job
CREATE OR ALTER PROCEDURE sp_CheckAlertThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertRecipients VARCHAR(500) = '${ALERT_EMAIL_LIST}';
    
    -- Check for excessive dwell time
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations fwo
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
            AND fwo.DwellTimeMinutes > 240 -- 4 hours threshold
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Products with dwell time exceeding 4 hours detected. ' +
                            'Review warehouse operations immediately.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'LogiFleet Pulse: High Dwell Time Alert',
            @body = @AlertMessage,
            @importance = 'High';
    END;
    
    -- Check for fleet idle time percentage
    DECLARE @FleetIdlePercent DECIMAL(5,2);
    SELECT @FleetIdlePercent = 
        AVG(CAST(IdleTimeMinutes AS FLOAT) / 
            NULLIF(DATEDIFF(MINUTE, t1.FullDateTime, t2.FullDateTime), 0) * 100)
    FROM FactFleetTrips fft
    INNER JOIN DimTime t1 ON fft.TripStartTimeKey = t1.TimeKey
    LEFT JOIN DimTime t2 ON fft.TripEndTimeKey = t2.TimeKey
    WHERE t1.FullDateTime >= DATEADD(DAY, -1, GETDATE());
    
    IF @FleetIdlePercent > 15
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time at ' + 
                            CAST(@FleetIdlePercent AS VARCHAR(10)) + 
                            '%. Exceeds 15% threshold. Review route efficiency.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = @AlertRecipients,
            @subject = 'LogiFleet Pulse: High Fleet Idle Time',
            @body = @AlertMessage;
    END;
    
    -- Check for vehicle maintenance priorities
    INSERT INTO MaintenanceQueue (VehicleID, PriorityScore, AlertDateTime)
    SELECT TOP 10
        vh.VehicleID,
        (
            (CASE WHEN vh.TirePressurePSI < 32 THEN 30 ELSE 0 END) +
            (CASE WHEN vh.EngineDiagnosticCode IS NOT NULL THEN 40 ELSE 0 END)
        ) AS PriorityScore,
        GETDATE()
    FROM DimVehicle vh
    WHERE vh.IsActive = 1
        AND (vh.TirePressurePSI < 32 OR vh.EngineDiagnosticCode IS NOT NULL)
    ORDER BY PriorityScore DESC;
END;
GO

-- Schedule job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_Alert_Monitor';

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_Alert_Monitor',
    @step_name = 'Check Thresholds',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_CheckAlertThresholds',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_Alert_Monitor',
    @schedule_name = 'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_Alert_Monitor';
```

## Row-Level Security Implementation

```sql
-- Create security filter function
CREATE OR ALTER FUNCTION fn_SecurityPredicate_Warehouse(@EmployeeKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
    SELECT 1 AS SecurityCheck
    WHERE 
        @EmployeeKey = CAST(SESSION_CONTEXT(N'EmployeeKey') AS INT)
        OR IS_MEMBER('LogisticsAdmin') = 1
        OR IS_MEMBER('WarehouseManager') = 1;
GO

-- Apply security policy to fact table
CREATE SECURITY POLICY WarehouseOperationsSecurity
ADD FILTER PREDICATE dbo.fn_SecurityPredicate_Warehouse(EmployeeKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
GO

-- Set session context in application connection string
-- Application code example (C# / .NET)
/*
using (SqlConnection conn = new SqlConnection(connectionString))
{
    conn.Open();
    using (SqlCommand cmd = new SqlCommand(
        "EXEC
