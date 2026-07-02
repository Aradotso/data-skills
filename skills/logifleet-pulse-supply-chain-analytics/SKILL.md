---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics analytics platform with multi-fact star schema for warehouse, fleet, and supply chain intelligence
triggers:
  - set up logifleet pulse analytics
  - configure supply chain data warehouse
  - build logistics intelligence dashboard
  - implement multi-fact star schema for fleet tracking
  - create warehouse gravity zone model
  - deploy power bi logistics template
  - integrate fleet telemetry with warehouse data
  - query cross-modal supply chain metrics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics analytics platform built on MS SQL Server and Power BI. It provides a multi-fact star schema that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time supply chain intelligence.

**Core Capabilities:**
- Multi-fact data warehouse with time-phased dimensions
- Cross-fact KPI harmonization (warehouse + fleet metrics)
- Warehouse Gravity Zones™ spatial optimization
- Predictive bottleneck detection
- Real-time dashboard refresh (15-minute granularity)
- Role-based access control with row-level security

**Primary Language:** SQL (T-SQL for MS SQL Server)
**Visualization:** Power BI (.pbit templates)
**Supporting:** HTML/JavaScript for web components

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate permissions: db_owner for schema deployment

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Execute the main schema script
USE master;
GO

-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema creation script
-- (assuming schema.sql contains the full DDL)
:r schema.sql
GO
```

### Step 3: Configure Data Sources

Update the configuration file with your connection strings:

```json
{
  "connections": {
    "sql_server": "Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};",
    "wms_api": "${WMS_API_ENDPOINT}",
    "fleet_telemetry": "${FLEET_API_ENDPOINT}",
    "weather_api": "${WEATHER_API_KEY}"
  },
  "refresh_interval": 15,
  "timezone": "UTC"
}
```

### Step 4: Import Power BI Template

```powershell
# Open the Power BI template
# File: LogiFleet_Pulse_Master.pbit

# Update connection parameters when prompted:
# - Server: your SQL Server hostname
# - Database: LogiFleetPulse
# - Authentication: Windows or SQL Server
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity events
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'PUTAWAY','PICK','PACK','SHIP'
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    EmployeeKey INT,
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Clustered columnstore index for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouseOperations 
ON FactWarehouseOperations;
```

**FactFleetTrips** - Fleet movement and telemetry
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT IDENTITY(1,1) PRIMARY KEY,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadWeightKG DECIMAL(10,2),
    AverageSpeedKMH DECIMAL(5,2),
    CONSTRAINT FK_FT_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactFleetTrips 
ON FactFleetTrips;
```

**FactCrossDock** - Cross-dock transfer events
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    ProductKey INT NOT NULL,
    Quantity INT,
    TransferDurationMinutes INT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

### Dimension Tables

**DimTime** - 15-minute granularity time dimension
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    [Date] DATE NOT NULL,
    [Year] INT NOT NULL,
    Quarter INT NOT NULL,
    [Month] INT NOT NULL,
    [Day] INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayOfWeekName VARCHAR(20),
    [Hour] INT NOT NULL,
    MinuteBucket INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    FiscalPeriod INT
);

-- Populate time dimension
INSERT INTO DimTime (TimeKey, FullDateTime, [Date], [Year], Quarter, [Month], [Day], 
                     DayOfWeek, DayOfWeekName, [Hour], MinuteBucket, IsWeekend)
SELECT 
    CAST(FORMAT(dt, 'yyyyMMddHHmm') AS INT) as TimeKey,
    dt as FullDateTime,
    CAST(dt AS DATE) as [Date],
    YEAR(dt) as [Year],
    DATEPART(QUARTER, dt) as Quarter,
    MONTH(dt) as [Month],
    DAY(dt) as [Day],
    DATEPART(WEEKDAY, dt) as DayOfWeek,
    DATENAME(WEEKDAY, dt) as DayOfWeekName,
    DATEPART(HOUR, dt) as [Hour],
    (DATEPART(MINUTE, dt) / 15) * 15 as MinuteBucket,
    CASE WHEN DATEPART(WEEKDAY, dt) IN (1,7) THEN 1 ELSE 0 END as IsWeekend
FROM (
    SELECT DATEADD(MINUTE, (n * 15), '2025-01-01 00:00:00') as dt
    FROM (SELECT TOP 70080 ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) - 1 as n
          FROM sys.all_objects a CROSS JOIN sys.all_objects b) nums
) dates;
```

**DimProductGravity** - Product categorization with gravity score
```sql
CREATE TABLE DimProduct (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductSKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    CategoryKey INT,
    GravityScore DECIMAL(5,2), -- 0-100 based on velocity, value, fragility
    UnitWeight DECIMAL(10,3),
    IsPerishable BIT,
    OptimalTemperatureCelsius DECIMAL(4,1)
);

-- Calculate gravity score
UPDATE DimProduct
SET GravityScore = (
    (VelocityScore * 0.5) + 
    (ValueScore * 0.3) + 
    (FragilityScore * 0.2)
)
FROM (
    SELECT 
        ProductKey,
        NTILE(100) OVER (ORDER BY AvgDailyPicks) as VelocityScore,
        NTILE(100) OVER (ORDER BY UnitValue) as ValueScore,
        CASE WHEN IsPerishable = 1 OR IsFragile = 1 THEN 80 ELSE 20 END as FragilityScore
    FROM ProductMetrics
) scores
WHERE DimProduct.ProductKey = scores.ProductKey;
```

**DimWarehouse** - Warehouse locations with gravity zones
```sql
CREATE TABLE DimWarehouse (
    WarehouseKey INT IDENTITY(1,1) PRIMARY KEY,
    WarehouseCode VARCHAR(20) UNIQUE NOT NULL,
    WarehouseName VARCHAR(100),
    GeographyKey INT,
    ZoneType VARCHAR(20), -- 'HIGH_GRAVITY', 'MEDIUM_GRAVITY', 'LOW_GRAVITY', 'CROSSDOCK'
    DistanceToShippingDockMeters INT,
    CapacityPallets INT,
    TemperatureControlled BIT
);
```

**DimVehicle** - Fleet vehicle master
```sql
CREATE TABLE DimVehicle (
    VehicleKey INT IDENTITY(1,1) PRIMARY KEY,
    VehicleID VARCHAR(50) UNIQUE NOT NULL,
    VehicleType VARCHAR(50), -- 'TRUCK', 'VAN', 'TRAILER'
    MaxLoadKG DECIMAL(10,2),
    FuelType VARCHAR(20),
    YearManufactured INT,
    MaintenanceScore DECIMAL(5,2), -- 0-100 based on service history
    TelemetryEnabled BIT
);
```

## Key Queries & Analytics Patterns

### Cross-Fact Query: Warehouse Dwell Time vs Fleet Idle Time

```sql
-- Identify correlations between warehouse dwell and fleet delays
WITH WarehouseDwell AS (
    SELECT 
        w.WarehouseCode,
        p.ProductSKU,
        AVG(wo.DwellTimeMinutes) as AvgDwellMinutes,
        COUNT(*) as OperationCount
    FROM FactWarehouseOperations wo
    JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(DAY, -30, GETDATE())
    AND wo.OperationType = 'SHIP'
    GROUP BY w.WarehouseCode, p.ProductSKU
),
FleetIdle AS (
    SELECT 
        v.VehicleID,
        r.RouteCode,
        AVG(ft.IdleTimeMinutes) as AvgIdleMinutes,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) as FuelEfficiency
    FROM FactFleetTrips ft
    JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    JOIN DimRoute r ON ft.RouteKey = r.RouteKey
    JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    WHERE t.[Date] >= DATEADD(DAY, -30, GETDATE())
    GROUP BY v.VehicleID, r.RouteCode
)
SELECT 
    wd.WarehouseCode,
    wd.ProductSKU,
    wd.AvgDwellMinutes,
    fi.VehicleID,
    fi.AvgIdleMinutes,
    fi.FuelEfficiency,
    -- Correlation indicator
    CASE 
        WHEN wd.AvgDwellMinutes > 120 AND fi.AvgIdleMinutes > 30 
        THEN 'HIGH_RISK'
        WHEN wd.AvgDwellMinutes > 60 AND fi.AvgIdleMinutes > 15 
        THEN 'MEDIUM_RISK'
        ELSE 'LOW_RISK'
    END as BottleneckRisk
FROM WarehouseDwell wd
CROSS JOIN FleetIdle fi
WHERE wd.AvgDwellMinutes > 60
ORDER BY wd.AvgDwellMinutes DESC, fi.AvgIdleMinutes DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend product relocations based on gravity mismatch
SELECT 
    p.ProductSKU,
    p.ProductName,
    p.GravityScore,
    w.WarehouseCode,
    w.ZoneType,
    w.DistanceToShippingDockMeters,
    COUNT(*) as RecentPicks,
    AVG(wo.DwellTimeMinutes) as AvgDwellTime,
    CASE 
        WHEN p.GravityScore > 75 AND w.ZoneType != 'HIGH_GRAVITY' 
        THEN 'MOVE_TO_HIGH_GRAVITY'
        WHEN p.GravityScore < 25 AND w.ZoneType = 'HIGH_GRAVITY' 
        THEN 'MOVE_TO_LOW_GRAVITY'
        ELSE 'OPTIMAL'
    END as RecommendedAction
FROM FactWarehouseOperations wo
JOIN DimProduct p ON wo.ProductKey = p.ProductKey
JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
JOIN DimTime t ON wo.TimeKey = t.TimeKey
WHERE t.[Date] >= DATEADD(DAY, -7, GETDATE())
AND wo.OperationType = 'PICK'
GROUP BY p.ProductSKU, p.ProductName, p.GravityScore, 
         w.WarehouseCode, w.ZoneType, w.DistanceToShippingDockMeters
HAVING COUNT(*) > 10
ORDER BY 
    CASE 
        WHEN p.GravityScore > 75 AND w.ZoneType != 'HIGH_GRAVITY' THEN 1
        WHEN p.GravityScore < 25 AND w.ZoneType = 'HIGH_GRAVITY' THEN 2
        ELSE 3
    END,
    RecentPicks DESC;
```

### Predictive Bottleneck Detection

```sql
-- Calculate real-time bottleneck index using weighted factors
CREATE OR ALTER VIEW vw_BottleneckIndex AS
WITH CurrentMetrics AS (
    SELECT 
        w.WarehouseKey,
        w.WarehouseCode,
        COUNT(DISTINCT wo.OperationID) as CurrentOperations,
        AVG(wo.DwellTimeMinutes) as AvgDwellTime,
        w.CapacityPallets,
        SUM(CASE WHEN wo.OperationType = 'PICK' THEN 1 ELSE 0 END) as PickCount,
        SUM(CASE WHEN wo.OperationType = 'SHIP' THEN 1 ELSE 0 END) as ShipCount
    FROM FactWarehouseOperations wo
    JOIN DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
    JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MINUTE, -120, GETDATE())
    GROUP BY w.WarehouseKey, w.WarehouseCode, w.CapacityPallets
),
FleetMetrics AS (
    SELECT 
        COUNT(DISTINCT ft.TripID) as ActiveTrips,
        AVG(ft.IdleTimeMinutes) as AvgIdleTime,
        SUM(CASE WHEN ft.IdleTimeMinutes > 30 THEN 1 ELSE 0 END) as DelayedTrips
    FROM FactFleetTrips ft
    JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(MINUTE, -120, GETDATE())
)
SELECT 
    cm.WarehouseCode,
    cm.CurrentOperations,
    cm.AvgDwellTime,
    fm.ActiveTrips,
    fm.AvgIdleTime,
    -- Bottleneck Index Formula (0-100)
    CAST(
        (cm.AvgDwellTime / 10.0 * 0.4) +  -- Dwell time weight: 40%
        ((cm.CurrentOperations / CAST(cm.CapacityPallets AS FLOAT)) * 100 * 0.3) +  -- Capacity utilization: 30%
        (fm.AvgIdleTime / 5.0 * 0.2) +  -- Fleet idle: 20%
        ((cm.PickCount - cm.ShipCount) / 10.0 * 0.1)  -- Pick/Ship imbalance: 10%
    AS DECIMAL(5,2)) as BottleneckIndex,
    CASE 
        WHEN CAST((cm.AvgDwellTime / 10.0 * 0.4) +
                  ((cm.CurrentOperations / CAST(cm.CapacityPallets AS FLOAT)) * 100 * 0.3) +
                  (fm.AvgIdleTime / 5.0 * 0.2) +
                  ((cm.PickCount - cm.ShipCount) / 10.0 * 0.1) AS DECIMAL(5,2)) > 70 
        THEN 'CRITICAL'
        WHEN CAST((cm.AvgDwellTime / 10.0 * 0.4) +
                  ((cm.CurrentOperations / CAST(cm.CapacityPallets AS FLOAT)) * 100 * 0.3) +
                  (fm.AvgIdleTime / 5.0 * 0.2) +
                  ((cm.PickCount - cm.ShipCount) / 10.0 * 0.1) AS DECIMAL(5,2)) > 50 
        THEN 'WARNING'
        ELSE 'NORMAL'
    END as AlertLevel
FROM CurrentMetrics cm
CROSS JOIN FleetMetrics fm;
```

### Incremental Data Loading Stored Procedure

```sql
-- Stored procedure for incremental fact table loading
CREATE OR ALTER PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Default to last hour if no parameter provided
    IF @LastLoadDateTime IS NULL
        SET @LastLoadDateTime = DATEADD(HOUR, -1, GETDATE());
    
    -- Insert new records from staging
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        Quantity, DwellTimeMinutes, EmployeeKey
    )
    SELECT 
        CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT) as TimeKey,
        w.WarehouseKey,
        p.ProductKey,
        s.OperationType,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartDateTime, s.EndDateTime) as DwellTimeMinutes,
        e.EmployeeKey
    FROM staging.WarehouseOperations s
    JOIN DimWarehouse w ON s.WarehouseCode = w.WarehouseCode
    JOIN DimProduct p ON s.ProductSKU = p.ProductSKU
    LEFT JOIN DimEmployee e ON s.EmployeeID = e.EmployeeID
    WHERE s.OperationDateTime > @LastLoadDateTime
    AND NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f
        WHERE f.TimeKey = CAST(FORMAT(s.OperationDateTime, 'yyyyMMddHHmm') AS INT)
        AND f.ProductKey = p.ProductKey
        AND f.WarehouseKey = w.WarehouseKey
    );
    
    -- Log load completion
    INSERT INTO etl.LoadLog (TableName, LoadDateTime, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
GO
```

## Power BI Integration

### DAX Measures for Cross-Fact KPIs

```dax
// Total Fleet Fuel Cost
TotalFuelCost = 
SUMX(
    FactFleetTrips,
    FactFleetTrips[FuelConsumedLiters] * RELATED(DimFuelPrice[PricePerLiter])
)

// Warehouse Utilization %
WarehouseUtilization = 
DIVIDE(
    COUNTROWS(FILTER(FactWarehouseOperations, FactWarehouseOperations[OperationType] = "SHIP")),
    RELATED(DimWarehouse[CapacityPallets]),
    0
) * 100

// Composite Efficiency Score
LogisticsEfficiencyScore = 
VAR DwellScore = 100 - MIN(AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 2, 100)
VAR IdleScore = 100 - MIN(AVERAGE(FactFleetTrips[IdleTimeMinutes]) * 2, 100)
VAR UtilScore = [WarehouseUtilization]
RETURN
    (DwellScore * 0.4) + (IdleScore * 0.3) + (UtilScore * 0.3)

// Time-Phased Comparison (vs Previous Period)
DwellTime_PP = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATEADD(DimTime[Date], -1, MONTH)
)

DwellTime_Variance = 
VAR CurrentDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR PreviousDwell = [DwellTime_PP]
RETURN
    DIVIDE(CurrentDwell - PreviousDwell, PreviousDwell, 0)
```

### Row-Level Security Implementation

```dax
// RLS for Warehouse Managers (can only see their assigned warehouse)
// Table: DimWarehouse
[WarehouseCode] IN 
    SELECTCOLUMNS(
        FILTER(
            UserWarehouseAccess,
            UserWarehouseAccess[UserEmail] = USERPRINCIPALNAME()
        ),
        "WarehouseCode", UserWarehouseAccess[WarehouseCode]
    )

// RLS for Regional Managers (can see all warehouses in their region)
// Table: DimGeography
[RegionCode] IN
    SELECTCOLUMNS(
        FILTER(
            UserRegionAccess,
            UserRegionAccess[UserEmail] = USERPRINCIPALNAME()
        ),
        "RegionCode", UserRegionAccess[RegionCode]
    )
```

## Automated Alerting

### SQL Server Agent Job for KPI Threshold Monitoring

```sql
-- Create alert stored procedure
CREATE OR ALTER PROCEDURE sp_MonitorKPIThresholds
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for high bottleneck index
    IF EXISTS (
        SELECT 1 FROM vw_BottleneckIndex
        WHERE BottleneckIndex > 70
    )
    BEGIN
        SELECT @AlertMessage = STRING_AGG(
            'CRITICAL: ' + WarehouseCode + ' bottleneck index: ' + 
            CAST(BottleneckIndex AS VARCHAR(10)),
            '; '
        )
        FROM vw_BottleneckIndex
        WHERE BottleneckIndex > 70;
        
        -- Send email (requires Database Mail configuration)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Critical Bottleneck Alert',
            @body = @AlertMessage;
    END;
    
    -- Check for fleet idle time exceeding threshold
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips ft
        JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
        AND ft.IdleTimeMinutes > (ft.DistanceKM / ft.AverageSpeedKMH * 60 * 0.15) -- idle > 15% of trip time
    )
    BEGIN
        SELECT @AlertMessage = 'WARNING: ' + CAST(COUNT(*) AS VARCHAR(10)) + 
                               ' fleet trips have excessive idle time in the last 2 hours.'
        FROM FactFleetTrips ft
        JOIN DimTime t ON ft.StartTimeKey = t.TimeKey
        WHERE t.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
        AND ft.IdleTimeMinutes > (ft.DistanceKM / ft.AverageSpeedKMH * 60 * 0.15);
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Fleet Idle Time Warning',
            @body = @AlertMessage;
    END;
END;
GO

-- Schedule the alert job (run every 15 minutes)
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_KPI_Monitoring',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_KPI_Monitoring',
    @step_name = 'Check_Thresholds',
    @subsystem = 'TSQL',
    @command = 'EXEC sp_MonitorKPIThresholds',
    @database_name = 'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Every_15_Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_KPI_Monitoring',
    @schedule_name = 'Every_15_Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_KPI_Monitoring';
```

## Configuration Best Practices

### Indexing Strategy

```sql
-- Create non-clustered indexes for frequently filtered columns
CREATE NONCLUSTERED INDEX IX_FactWarehouseOps_Time_Product
ON FactWarehouseOperations (TimeKey, ProductKey)
INCLUDE (Quantity, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_FactFleetTrips_StartTime_Vehicle
ON FactFleetTrips (StartTimeKey, VehicleKey)
INCLUDE (DistanceKM, FuelConsumedLiters, IdleTimeMinutes);

-- Partition fact tables by date for performance
CREATE PARTITION FUNCTION pf_LogiFleetDate (DATE)
AS RANGE RIGHT FOR VALUES 
    ('2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01');

CREATE PARTITION SCHEME ps_LogiFleetDate
AS PARTITION pf_LogiFleetDate
TO ([PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY], [PRIMARY]);

-- Apply partitioning to fact tables
CREATE TABLE FactWarehouseOperations_Partitioned (
    -- columns as before
) ON ps_LogiFleetDate([Date]);
```

### Performance Tuning

```sql
-- Update statistics regularly
CREATE OR ALTER PROCEDURE sp_UpdateStatistics
AS
BEGIN
    UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN;
    UPDATE STATISTICS FactFleetTrips WITH FULLSCAN;
    UPDATE STATISTICS FactCrossDock WITH FULLSCAN;
END;
GO

-- Schedule weekly statistics update
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_Stats_Update';
EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_Stats_Update',
    @step_name = 'Update_All_Stats',
    @command = 'EXEC sp_UpdateStatistics',
    @database_name = 'LogiFleetPulse';
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom:** Power BI dataset refresh fails with timeout error

**Solution:**
```sql
-- Optimize with indexed views for slow aggregations
CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    t.[Date],
    w.WarehouseKey,
    wo.OperationType,
    COUNT_BIG(*) as OperationCount,
    AVG(CAST(wo.DwellTimeMinutes AS BIGINT)) as AvgDwellTime
FROM dbo.FactWarehouseOperations wo
JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
JOIN dbo.DimWarehouse w ON wo.WarehouseKey = w.WarehouseKey
GROUP BY t.[Date], w.WarehouseKey, wo.OperationType;
GO

CREATE UNIQUE CLUSTERED INDEX IX_vw_DailyWarehouseMetrics
ON vw_DailyWarehouseMetrics ([Date], WarehouseKey, OperationType);
```

### Issue: Incorrect Time Zone Handling

**Symptom:** Time-based aggregations show wrong periods

**Solution:**
```sql
-- Add UTC offset columns to DimTime
ALTER TABLE DimTime ADD UTCOffset INT;

UPDATE DimTime
SET UTCOffset = CASE 
    WHEN [Date] BETWEEN '2025-03-09' AND '2025-11-02' THEN -4 -- EDT
    ELSE -5 -- EST
END;

-- Use in queries
SELECT 
    DATEADD(HOUR, t.UTCOffset, t.FullDateTime) as LocalDateTime,
    COUNT(*) as Operations
FROM FactWarehouseOperations wo
JOIN DimTime t ON wo.TimeKey = t.TimeKey
GROUP BY DATEADD(HOUR, t.UTCOffset, t.FullDateTime);
```

### Issue: Row-Level Security Not Working

**Symptom:** Users see data they shouldn't have access to

**Solution:**
```sql
-- Verify RLS is enabled
SELECT 
