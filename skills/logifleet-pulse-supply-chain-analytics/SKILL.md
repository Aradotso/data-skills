---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse analytics
  - deploy supply chain data warehouse
  - create logistics power bi dashboard
  - configure warehouse fleet analytics
  - implement multi-fact star schema for logistics
  - build real-time supply chain intelligence
  - integrate warehouse and fleet telemetry data
  - set up logistics KPI dashboards
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is a comprehensive logistics intelligence platform combining MS SQL Server data warehousing with Power BI visualization. It implements a multi-fact star schema to unify warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time supply chain analytics.

**Key capabilities:**
- Multi-fact star schema with time-phased dimensions
- Cross-modal KPI harmonization (warehouse + fleet + external data)
- Real-time dashboards with 15-minute refresh granularity
- Predictive bottleneck detection and fleet triage
- Warehouse gravity zone optimization
- Role-based access with row-level security

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Admin access to target SQL Server instance

### Step 1: Clone the Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance via SSMS
-- Execute the schema deployment script
:setvar DatabaseName "LogiFleetPulse"
:setvar DataPath "C:\SQLData\"
:setvar LogPath "C:\SQLLogs\"

-- Run the master deployment script
:r .\sql\00_CreateDatabase.sql
:r .\sql\01_CreateDimensions.sql
:r .\sql\02_CreateFacts.sql
:r .\sql\03_CreateViews.sql
:r .\sql\04_CreateStoredProcedures.sql
:r .\sql\05_CreateIndexes.sql
:r .\sql\06_CreateSecurity.sql
```

### Step 3: Configure Data Sources

Create a configuration file for your environment:

```json
{
  "sqlServer": {
    "server": "YOUR_SERVER_NAME",
    "database": "LogiFleetPulse",
    "authType": "WindowsAuthentication"
  },
  "dataSources": {
    "wms": {
      "type": "REST_API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}",
      "refreshInterval": 900
    },
    "fleetTelemetry": {
      "type": "ODBC",
      "connectionString": "${FLEET_CONNECTION_STRING}",
      "refreshInterval": 900
    },
    "weather": {
      "type": "REST_API",
      "endpoint": "https://api.weatherapi.com/v1",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshInterval": 3600
    }
  }
}
```

### Step 4: Import Power BI Template

```bash
# Open Power BI Desktop
# File > Import > Power BI Template
# Select: LogiFleet_Pulse_Master.pbit
# Enter SQL Server connection parameters when prompted
```

## Core Data Model

### Fact Tables

**FactWarehouseOperations**: Records all warehouse activities
```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT,
    LaborHours DECIMAL(10,2),
    ZoneKey INT,
    GravityScore DECIMAL(5,2)
);
```

**FactFleetTrips**: Fleet telemetry and route data
```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginLocationKey INT NOT NULL,
    DestinationLocationKey INT NOT NULL,
    DistanceMiles DECIMAL(10,2),
    FuelGallons DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadWeightLbs INT,
    TripDurationMinutes INT,
    DelayMinutes INT,
    DelayReasonKey INT
);
```

### Dimension Tables

**DimTime**: 15-minute granularity time dimension
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfWeek INT NOT NULL,
    Hour INT NOT NULL,
    Minute15Block INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL
);
```

**DimProductGravity**: Product classification with gravity scoring
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    VelocityClass VARCHAR(20), -- 'FAST', 'MEDIUM', 'SLOW'
    ValueClass VARCHAR(20), -- 'HIGH', 'MEDIUM', 'LOW'
    FragilityScore DECIMAL(3,2), -- 0.00 to 1.00
    GravityScore DECIMAL(5,2), -- Composite score
    OptimalZoneType VARCHAR(50)
);
```

## Data Loading Patterns

### Incremental Load Stored Procedure

```sql
CREATE PROCEDURE dbo.usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last load time if not provided
    IF @LastLoadDateTime IS NULL
        SELECT @LastLoadDateTime = ISNULL(MAX(LoadDateTime), '1900-01-01')
        FROM dbo.ETLLog
        WHERE TableName = 'FactWarehouseOperations';
    
    -- Load new operations
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType,
        QuantityHandled, DwellTimeMinutes, LaborHours, ZoneKey, GravityScore
    )
    SELECT 
        dt.TimeKey,
        dw.WarehouseKey,
        dp.ProductKey,
        stg.OperationType,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTime,
        stg.LaborHours,
        dz.ZoneKey,
        dp.GravityScore
    FROM staging.WarehouseOperations stg
    INNER JOIN DimTime dt ON dt.DateTime = stg.OperationDateTime
    INNER JOIN DimWarehouse dw ON dw.WarehouseCode = stg.WarehouseCode
    INNER JOIN DimProductGravity dp ON dp.SKU = stg.SKU
    LEFT JOIN DimZone dz ON dz.ZoneCode = stg.ZoneCode
    WHERE stg.LoadDateTime > @LastLoadDateTime;
    
    -- Log successful load
    INSERT INTO ETLLog (TableName, LoadDateTime, RowsInserted)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT);
END;
```

### Fleet Telemetry Ingestion

```sql
CREATE PROCEDURE dbo.usp_LoadFleetTrips
    @BatchSize INT = 10000
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Process telemetry stream in batches
    WHILE EXISTS (SELECT 1 FROM staging.FleetTelemetryStream WHERE Processed = 0)
    BEGIN
        INSERT INTO FactFleetTrips (
            TimeKey, VehicleKey, DriverKey, RouteKey,
            OriginLocationKey, DestinationLocationKey,
            DistanceMiles, FuelGallons, IdleTimeMinutes,
            LoadWeightLbs, TripDurationMinutes, DelayMinutes, DelayReasonKey
        )
        SELECT TOP (@BatchSize)
            dt.TimeKey,
            dv.VehicleKey,
            dd.DriverKey,
            dr.RouteKey,
            dloc_origin.LocationKey,
            dloc_dest.LocationKey,
            stg.Distance,
            stg.FuelConsumed,
            stg.IdleTime,
            stg.LoadWeight,
            stg.TripDuration,
            stg.DelayMinutes,
            ISNULL(delay.DelayReasonKey, 0)
        FROM staging.FleetTelemetryStream stg
        INNER JOIN DimTime dt ON dt.DateTime = stg.TripStartTime
        INNER JOIN DimVehicle dv ON dv.VehicleID = stg.VehicleID
        INNER JOIN DimDriver dd ON dd.DriverID = stg.DriverID
        INNER JOIN DimRoute dr ON dr.RouteCode = stg.RouteCode
        INNER JOIN DimLocation dloc_origin ON dloc_origin.LocationCode = stg.OriginCode
        INNER JOIN DimLocation dloc_dest ON dloc_dest.LocationCode = stg.DestinationCode
        LEFT JOIN DimDelayReason delay ON delay.ReasonCode = stg.DelayReasonCode
        WHERE stg.Processed = 0;
        
        -- Mark batch as processed
        UPDATE TOP (@BatchSize) staging.FleetTelemetryStream
        SET Processed = 1
        WHERE Processed = 0;
    END;
END;
```

## Key Analytics Queries

### Cross-Fact KPI: Dwell Time vs Fleet Idle Cost

```sql
-- Calculate correlation between warehouse dwell time and fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        dt.Date,
        dw.WarehouseName,
        dp.Category,
        AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
        SUM(fwo.QuantityHandled) AS TotalUnits
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dt.Date >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY dt.Date, dw.WarehouseName, dp.Category
),
FleetIdle AS (
    SELECT 
        dt.Date,
        dloc.WarehouseName,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
        AVG(fft.IdleTimeMinutes * dv.HourlyOperatingCost / 60.0) AS AvgIdleCost
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    INNER JOIN DimLocation dloc ON fft.OriginLocationKey = dloc.LocationKey
    INNER JOIN DimVehicle dv ON fft.VehicleKey = dv.VehicleKey
    WHERE dt.Date >= DATEADD(MONTH, -3, GETDATE())
        AND dloc.LocationType = 'WAREHOUSE'
    GROUP BY dt.Date, dloc.WarehouseName
)
SELECT 
    wd.Date,
    wd.WarehouseName,
    wd.Category,
    wd.AvgDwellTime,
    fi.AvgIdleTime,
    fi.AvgIdleCost,
    CASE 
        WHEN wd.AvgDwellTime > 120 AND fi.AvgIdleTime > 30 
        THEN 'HIGH_RISK'
        WHEN wd.AvgDwellTime > 60 AND fi.AvgIdleTime > 15
        THEN 'MEDIUM_RISK'
        ELSE 'LOW_RISK'
    END AS BottleneckRisk
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi ON wd.Date = fi.Date AND wd.WarehouseName = fi.WarehouseName
ORDER BY wd.Date DESC, BottleneckRisk DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Identify products in suboptimal zones
CREATE VIEW vw_GravityZoneMismatch AS
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.OptimalZoneType,
    dz.ZoneType AS CurrentZoneType,
    dw.WarehouseName,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTime,
    COUNT(*) AS OperationCount,
    CASE 
        WHEN dp.OptimalZoneType != dz.ZoneType THEN 'REASSIGN'
        ELSE 'OPTIMAL'
    END AS RecommendedAction
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN DimZone dz ON fwo.ZoneKey = dz.ZoneKey
INNER JOIN DimWarehouse dw ON dz.WarehouseKey = dw.WarehouseKey
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
WHERE dt.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY 
    dp.SKU, dp.ProductName, dp.GravityScore, 
    dp.OptimalZoneType, dz.ZoneType, dw.WarehouseName
HAVING AVG(fwo.DwellTimeMinutes) > 60;
```

### Fleet Triage Engine

```sql
-- Prioritize fleet maintenance based on revenue impact
CREATE PROCEDURE dbo.usp_GenerateFleetTriagePriority
AS
BEGIN
    WITH VehicleHealth AS (
        SELECT 
            dv.VehicleKey,
            dv.VehicleID,
            dv.VehicleType,
            -- Health score based on telemetry
            CASE 
                WHEN dv.TirePressureStatus = 'LOW' THEN 0.7
                WHEN dv.EngineStatus = 'WARNING' THEN 0.6
                WHEN dv.LastMaintenanceDays > 90 THEN 0.5
                ELSE 1.0
            END AS HealthScore
        FROM DimVehicle dv
    ),
    VehicleRevenue AS (
        SELECT 
            fft.VehicleKey,
            SUM(fft.LoadWeightLbs * dp.AvgValuePerLb) AS TotalCargoValue,
            COUNT(DISTINCT fft.TripKey) AS TripCount,
            AVG(dp.GravityScore) AS AvgCargoGravity
        FROM FactFleetTrips fft
        INNER JOIN FactWarehouseOperations fwo 
            ON fft.TimeKey = fwo.TimeKey 
            AND fft.OriginLocationKey = fwo.WarehouseKey
        INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
        WHERE fft.TimeKey >= (SELECT TimeKey FROM DimTime WHERE Date = DATEADD(DAY, -7, GETDATE()))
        GROUP BY fft.VehicleKey
    )
    SELECT 
        vh.VehicleID,
        vh.VehicleType,
        vh.HealthScore,
        ISNULL(vr.TotalCargoValue, 0) AS RevenueAtRisk,
        ISNULL(vr.AvgCargoGravity, 0) AS CargoGravity,
        -- Priority score: lower health + higher revenue = higher priority
        CAST((1 - vh.HealthScore) * ISNULL(vr.TotalCargoValue, 0) AS DECIMAL(18,2)) AS TriagePriority
    FROM VehicleHealth vh
    LEFT JOIN VehicleRevenue vr ON vh.VehicleKey = vr.VehicleKey
    WHERE vh.HealthScore < 1.0
    ORDER BY TriagePriority DESC;
END;
```

## Power BI Configuration

### Connection Setup (M Query)

```powerquery
let
    Source = Sql.Database(
        Environment.GetEnvironmentVariable("SQL_SERVER"), 
        "LogiFleetPulse"
    ),
    FactWarehouseOperations = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleetTrips = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    
    // Apply incremental refresh
    FilteredOperations = Table.SelectRows(
        FactWarehouseOperations, 
        each [TimeKey] >= Int32.From(
            DateTime.ToText([RangeStart], "yyyyMMddHHmm")
        )
    )
in
    FilteredOperations
```

### DAX Measures for Cross-Fact Analytics

```dax
// Total Dwell Cost (combines warehouse time with labor cost)
TotalDwellCost = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[DwellTimeMinutes] / 60 * 
    RELATED(DimWarehouse[HourlyLaborCost])
)

// Fleet Efficiency Ratio
FleetEfficiencyRatio = 
DIVIDE(
    SUM(FactFleetTrips[DistanceMiles]),
    SUM(FactFleetTrips[FuelGallons]) + 
    SUM(FactFleetTrips[IdleTimeMinutes]) / 60 * 
    AVERAGE(DimVehicle[HourlyOperatingCost]),
    0
)

// Composite Supply Chain Health Score
SupplyChainHealthScore = 
VAR WarehouseScore = 1 - (AVERAGE(FactWarehouseOperations[DwellTimeMinutes]) / 480)
VAR FleetScore = 1 - (AVERAGE(FactFleetTrips[DelayMinutes]) / 120)
VAR InventoryScore = 1 - (COUNTROWS(FILTER(DimProductGravity, [VelocityClass] = "SLOW")) / COUNTROWS(DimProductGravity))
RETURN (WarehouseScore * 0.4) + (FleetScore * 0.4) + (InventoryScore * 0.2)

// Time Intelligence: YoY Dwell Time Change
DwellTimeYoY = 
VAR CurrentDwell = [TotalDwellCost]
VAR PriorYearDwell = CALCULATE([TotalDwellCost], SAMEPERIODLASTYEAR(DimTime[Date]))
RETURN DIVIDE(CurrentDwell - PriorYearDwell, PriorYearDwell, 0)
```

### Row-Level Security (RLS)

```dax
// DimWarehouse table RLS expression
[WarehouseRegion] = USERPRINCIPALNAME() || 
[WarehouseRegion] IN (
    LOOKUPVALUE(
        SecurityUserRegions[Region],
        SecurityUserRegions[UserEmail],
        USERPRINCIPALNAME()
    )
)
```

## Automated Alerting

### Scheduled Alert Stored Procedure

```sql
CREATE PROCEDURE dbo.usp_SendSupplyChainAlerts
AS
BEGIN
    DECLARE @AlertSubject NVARCHAR(200);
    DECLARE @AlertBody NVARCHAR(MAX);
    DECLARE @Recipients NVARCHAR(500) = '${ALERT_EMAIL_RECIPIENTS}';
    
    -- Alert 1: High dwell time in premium zones
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations fwo
        INNER JOIN DimZone dz ON fwo.ZoneKey = dz.ZoneKey
        INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
        WHERE dt.Date = CAST(GETDATE() AS DATE)
            AND dz.ZoneType = 'PREMIUM'
            AND fwo.DwellTimeMinutes > 180
    )
    BEGIN
        SET @AlertSubject = 'ALERT: High Dwell Time in Premium Zones';
        SET @AlertBody = 'Multiple SKUs have exceeded 3-hour dwell time in premium warehouse zones today.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetPulse',
            @recipients = @Recipients,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END;
    
    -- Alert 2: Fleet idle cost threshold
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips fft
        INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
        WHERE dt.Date = CAST(GETDATE() AS DATE)
        GROUP BY fft.VehicleKey
        HAVING SUM(fft.IdleTimeMinutes) > 120
    )
    BEGIN
        SET @AlertSubject = 'ALERT: Excessive Fleet Idle Time';
        SET @AlertBody = 'One or more vehicles have exceeded 2 hours of idle time today.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetPulse',
            @recipients = @Recipients,
            @subject = @AlertSubject,
            @body = @AlertBody;
    END;
END;

-- Schedule via SQL Agent Job
EXEC msdb.dbo.sp_add_job @job_name = 'LogiFleet_Alerts';
EXEC msdb.dbo.sp_add_jobstep 
    @job_name = 'LogiFleet_Alerts',
    @step_name = 'Check_Thresholds',
    @command = 'EXEC dbo.usp_SendSupplyChainAlerts';
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = 'Every_15_Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;
```

## Common Patterns

### Pattern 1: Adding New Fact Table

```sql
-- Step 1: Create fact table with proper foreign keys
CREATE TABLE FactInventorySnapshot (
    SnapshotKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    WarehouseKey INT NOT NULL FOREIGN KEY REFERENCES DimWarehouse(WarehouseKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    QuantityOnHand INT NOT NULL,
    QuantityReserved INT NOT NULL,
    QuantityAvailable AS (QuantityOnHand - QuantityReserved) PERSISTED,
    UnitCost DECIMAL(10,2),
    TotalValue AS (QuantityOnHand * UnitCost) PERSISTED
);

-- Step 2: Create clustered columnstore index for analytics
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactInventorySnapshot 
ON FactInventorySnapshot;

-- Step 3: Create load procedure
CREATE PROCEDURE dbo.usp_LoadInventorySnapshot
AS
BEGIN
    INSERT INTO FactInventorySnapshot (TimeKey, WarehouseKey, ProductKey, 
        QuantityOnHand, QuantityReserved, UnitCost)
    SELECT 
        dt.TimeKey,
        dw.WarehouseKey,
        dp.ProductKey,
        stg.QtyOnHand,
        stg.QtyReserved,
        stg.UnitCost
    FROM staging.InventorySnapshot stg
    INNER JOIN DimTime dt ON dt.DateTime = stg.SnapshotDateTime
    INNER JOIN DimWarehouse dw ON dw.WarehouseCode = stg.WarehouseCode
    INNER JOIN DimProductGravity dp ON dp.SKU = stg.SKU;
END;
```

### Pattern 2: External Data Integration (Weather API)

```sql
-- Create staging table for external data
CREATE TABLE staging.WeatherData (
    LocationCode VARCHAR(50),
    RecordedDateTime DATETIME,
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2),
    WindSpeed DECIMAL(5,2),
    Conditions VARCHAR(100),
    LoadDateTime DATETIME DEFAULT GETDATE()
);

-- Python script to populate (run via SQL Agent or external scheduler)
-- Save as: scripts/load_weather_data.py
/*
import pyodbc
import requests
import os
from datetime import datetime

conn_str = os.environ['SQL_CONNECTION_STRING']
api_key = os.environ['WEATHER_API_KEY']

conn = pyodbc.connect(conn_str)
cursor = conn.cursor()

# Get locations to monitor
cursor.execute("SELECT LocationCode, Latitude, Longitude FROM DimLocation WHERE MonitorWeather = 1")
locations = cursor.fetchall()

for loc in locations:
    response = requests.get(
        f"https://api.weatherapi.com/v1/current.json",
        params={'key': api_key, 'q': f"{loc.Latitude},{loc.Longitude}"}
    )
    data = response.json()['current']
    
    cursor.execute("""
        INSERT INTO staging.WeatherData 
        (LocationCode, RecordedDateTime, Temperature, Precipitation, WindSpeed, Conditions)
        VALUES (?, ?, ?, ?, ?, ?)
    """, loc.LocationCode, datetime.now(), data['temp_f'], 
         data['precip_in'], data['wind_mph'], data['condition']['text'])

conn.commit()
*/

-- Merge weather data into dimension
CREATE PROCEDURE dbo.usp_ProcessWeatherData
AS
BEGIN
    MERGE DimLocation AS target
    USING (
        SELECT LocationCode, 
               AVG(Temperature) AS AvgTemp,
               SUM(Precipitation) AS TotalPrecip,
               MAX(CASE WHEN Conditions LIKE '%rain%' OR Conditions LIKE '%snow%' THEN 1 ELSE 0 END) AS HasAdverseWeather
        FROM staging.WeatherData
        WHERE LoadDateTime >= DATEADD(HOUR, -1, GETDATE())
        GROUP BY LocationCode
    ) AS source
    ON target.LocationCode = source.LocationCode
    WHEN MATCHED THEN
        UPDATE SET 
            CurrentTemperature = source.AvgTemp,
            WeatherRiskFlag = source.HasAdverseWeather,
            LastWeatherUpdate = GETDATE();
END;
```

### Pattern 3: Calculating Gravity Score

```sql
CREATE PROCEDURE dbo.usp_RecalculateGravityScores
AS
BEGIN
    -- Update gravity scores based on recent activity
    UPDATE dp
    SET dp.GravityScore = (
        -- Velocity component (40% weight)
        (CASE dp.VelocityClass
            WHEN 'FAST' THEN 1.0
            WHEN 'MEDIUM' THEN 0.5
            WHEN 'SLOW' THEN 0.1
        END * 0.4) +
        
        -- Value component (35% weight)
        (CASE dp.ValueClass
            WHEN 'HIGH' THEN 1.0
            WHEN 'MEDIUM' THEN 0.5
            WHEN 'LOW' THEN 0.2
        END * 0.35) +
        
        -- Fragility component (25% weight, inverse)
        ((1.0 - dp.FragilityScore) * 0.25)
    ),
    dp.OptimalZoneType = CASE
        WHEN (CASE dp.VelocityClass WHEN 'FAST' THEN 1.0 WHEN 'MEDIUM' THEN 0.5 ELSE 0.1 END * 0.4) +
             (CASE dp.ValueClass WHEN 'HIGH' THEN 1.0 WHEN 'MEDIUM' THEN 0.5 ELSE 0.2 END * 0.35) +
             ((1.0 - dp.FragilityScore) * 0.25) > 0.7 THEN 'PREMIUM'
        WHEN (CASE dp.VelocityClass WHEN 'FAST' THEN 1.0 WHEN 'MEDIUM' THEN 0.5 ELSE 0.1 END * 0.4) +
             (CASE dp.ValueClass WHEN 'HIGH' THEN 1.0 WHEN 'MEDIUM' THEN 0.5 ELSE 0.2 END * 0.35) +
             ((1.0 - dp.FragilityScore) * 0.25) > 0.4 THEN 'STANDARD'
        ELSE 'BULK'
    END
    FROM DimProductGravity dp;
END;
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom**: Dashboard refresh fails with timeout error

**Solution**:
```sql
-- Enable incremental refresh in Power BI
-- Add RangeStart and RangeEnd parameters in Power Query
-- In SQL, create indexed views for expensive aggregations

CREATE VIEW vw_DailyWarehouseMetrics
WITH SCHEMABINDING
AS
SELECT 
    dt.Date,
    fwo.WarehouseKey,
    fwo.ProductKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(fwo.QuantityHandled) AS TotalQuantity,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTime
FROM dbo.FactWarehouseOperations fwo
INNER JOIN dbo.DimTime dt ON fwo.TimeKey = dt.TimeKey
GROUP BY dt.Date, fwo.WarehouseKey, fwo.ProductKey;

CREATE UNIQUE CLUSTERED INDEX IDX_DailyWarehouseMetrics 
ON vw_DailyWarehouseMetrics (Date, WarehouseKey, ProductKey);
```

### Issue: High Query Latency on Cross-Fact Queries

**Symptom**: Queries joining multiple fact tables run slowly

**Solution**:
```sql
-- Add covering indexes on common join patterns
CREATE NONCLUSTERED INDEX
