---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing template for real-time logistics, fleet management, and supply chain intelligence dashboards
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy the logistics data warehouse schema"
  - "configure Power BI for fleet and warehouse KPIs"
  - "implement multi-fact star schema for logistics"
  - "create supply chain intelligence dashboards"
  - "integrate warehouse and fleet telemetry data"
  - "optimize logistics data model with cross-fact queries"
  - "build real-time supply chain monitoring system"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to deploy and configure LogiFleet Pulse, an advanced MS SQL Server and Power BI data warehousing solution for logistics, fleet management, and supply chain analytics. The project provides a multi-fact star schema that harmonizes warehouse operations, fleet telemetry, inventory management, and external data sources into unified semantic layers for real-time business intelligence.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **logistics intelligence engine** that combines:
- Warehouse operations tracking (receiving, putaway, picking, packing, shipping)
- Fleet telemetry and GPS data (routes, fuel consumption, idle time, vehicle diagnostics)
- Inventory management (aging, dwell time, turnover rates)
- Supplier reliability metrics (lead times, defect rates)
- External data integration (weather, traffic APIs)

The platform uses a **time-aware dimensional model** with cross-fact KPI harmonization, enabling queries like "Show warehouse dwell time correlated with fleet fuel costs by route and product category."

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telematics APIs)

### Step 1: Deploy SQL Schema

Clone the repository and navigate to the SQL scripts:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse/sql
```

Deploy the base schema to SQL Server:

```sql
-- Connect to your SQL Server instance via SSMS or sqlcmd
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Execute the schema creation script
-- This creates dimension and fact tables
:r CreateDimensions.sql
:r CreateFactTables.sql
:r CreateBridgeTables.sql
:r CreateIndexes.sql
:r CreateViews.sql
:r CreateStoredProcedures.sql
```

### Step 2: Configure Data Sources

Create a configuration file for connection strings:

```json
{
  "dataSources": {
    "wms": {
      "connectionString": "${WMS_CONNECTION_STRING}",
      "refreshInterval": 900
    },
    "telemetry": {
      "apiEndpoint": "${FLEET_TELEMETRY_API_URL}",
      "apiKey": "${FLEET_API_KEY}",
      "refreshInterval": 900
    },
    "erp": {
      "connectionString": "${ERP_CONNECTION_STRING}",
      "refreshInterval": 3600
    },
    "weather": {
      "apiEndpoint": "${WEATHER_API_URL}",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshInterval": 1800
    }
  },
  "sqlServer": {
    "server": "${SQL_SERVER_HOSTNAME}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectory"
  }
}
```

### Step 3: Load Power BI Template

Open Power BI Desktop and load the template:

```powershell
# Open the .pbit template file
start LogiFleet_Pulse_Master.pbit
```

When prompted, enter your SQL Server connection details:
- Server: `${SQL_SERVER_HOSTNAME}`
- Database: `LogiFleetPulse`

## Core Data Model Components

### Dimension Tables

**DimTime** - Time-phased dimension with 15-minute granularity:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2 NOT NULL,
    Hour TINYINT,
    DayOfWeek VARCHAR(10),
    DayOfMonth TINYINT,
    Month TINYINT,
    Quarter TINYINT,
    FiscalYear INT,
    IsWeekend BIT,
    IsHoliday BIT,
    ShiftNumber TINYINT,
    INDEX IX_DateTime (DateTime)
)
```

**DimGeography** - Hierarchical location dimension:

```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY,
    LocationID VARCHAR(50) NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, Distribution Center, Route Node
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10, 7),
    Longitude DECIMAL(10, 7),
    TimezoneOffset INT,
    INDEX IX_LocationID (LocationID),
    INDEX IX_City_Country (City, Country)
)
```

**DimProductGravity** - Product hierarchy with velocity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY,
    SKU VARCHAR(100) NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5, 2), -- Calculated from velocity, value, fragility
    VelocityClass VARCHAR(20), -- Fast, Medium, Slow
    FragilityRating TINYINT,
    UnitValue DECIMAL(18, 2),
    UnitWeight DECIMAL(10, 3),
    RequiresColdStorage BIT,
    RequiresHazmatHandling BIT,
    INDEX IX_SKU (SKU),
    INDEX IX_GravityScore (GravityScore DESC)
)
```

### Fact Tables

**FactWarehouseOperations** - Core warehouse activity tracking:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationStartTime DATETIME2,
    OperationEndTime DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    QuantityHandled INT,
    StorageZone VARCHAR(50),
    WorkerID VARCHAR(50),
    EquipmentID VARCHAR(50),
    DwellTimeHours DECIMAL(10, 2), -- Time in storage before next operation
    CycleTimeSeconds INT,
    ErrorFlag BIT,
    ErrorCode VARCHAR(50),
    INDEX IX_TimeProduct (TimeKey, ProductKey),
    INDEX IX_OperationType (OperationType)
)
```

**FactFleetTrips** - Fleet and route performance:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TripStartTime DATETIME2,
    TripEndTime DATETIME2,
    DistanceMiles DECIMAL(10, 2),
    FuelConsumedGallons DECIMAL(8, 3),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightLbs DECIMAL(12, 2),
    AverageSpeedMPH DECIMAL(5, 2),
    HarshBrakingEvents INT,
    HarshAccelerationEvents INT,
    WeatherDelayMinutes INT,
    TrafficDelayMinutes INT,
    OnTimeDelivery BIT,
    FuelEfficiencyMPG AS (DistanceMiles / NULLIF(FuelConsumedGallons, 0)),
    INDEX IX_TimeVehicle (TimeKey, VehicleID),
    INDEX IX_Routes (OriginGeographyKey, DestinationGeographyKey)
)
```

**FactCrossDock** - Cross-docking operations:

```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY,
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ArrivalTime DATETIME2,
    DepartureTime DATETIME2,
    DockDwellMinutes AS DATEDIFF(MINUTE, ArrivalTime, DepartureTime),
    QuantityTransferred INT,
    TransferWorkerID VARCHAR(50),
    TemperatureCompliance BIT,
    INDEX IX_TimeProduct (TimeKey, ProductKey)
)
```

## Key SQL Queries & Patterns

### Cross-Fact KPI: Warehouse Dwell Time vs Fleet Idle Cost

```sql
-- Calculate correlation between product dwell time and fleet idle costs
WITH WarehouseDwell AS (
    SELECT 
        dt.FiscalYear,
        dt.Quarter,
        dpg.Category,
        AVG(fwo.DwellTimeHours) AS AvgDwellHours,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimProductGravity dpg ON fwo.ProductKey = dpg.ProductKey
    WHERE fwo.OperationType = 'Picking'
        AND dt.FiscalYear = 2026
    GROUP BY dt.FiscalYear, dt.Quarter, dpg.Category
),
FleetIdle AS (
    SELECT 
        dt.FiscalYear,
        dt.Quarter,
        dpg.Category,
        AVG(fft.IdleTimeMinutes) AS AvgIdleMinutes,
        SUM(fft.IdleTimeMinutes * 0.5) AS TotalIdleCostUSD -- $0.50/min idle cost
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    INNER JOIN FactWarehouseOperations fwo ON fwo.GeographyKey = fft.OriginGeographyKey
    INNER JOIN DimProductGravity dpg ON fwo.ProductKey = dpg.ProductKey
    WHERE dt.FiscalYear = 2026
    GROUP BY dt.FiscalYear, dt.Quarter, dpg.Category
)
SELECT 
    wd.FiscalYear,
    wd.Quarter,
    wd.Category,
    wd.AvgDwellHours,
    fi.AvgIdleMinutes,
    fi.TotalIdleCostUSD,
    (wd.AvgDwellHours * fi.AvgIdleMinutes) AS DwellIdleCorrelation
FROM WarehouseDwell wd
INNER JOIN FleetIdle fi 
    ON wd.FiscalYear = fi.FiscalYear 
    AND wd.Quarter = fi.Quarter 
    AND wd.Category = fi.Category
ORDER BY DwellIdleCorrelation DESC
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouse zones with increasing dwell time trends
WITH DailyDwell AS (
    SELECT 
        dt.DateTime,
        CAST(dt.DateTime AS DATE) AS OperationDate,
        fwo.StorageZone,
        AVG(fwo.DwellTimeHours) AS AvgDailyDwell
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE dt.DateTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY dt.DateTime, CAST(dt.DateTime AS DATE), fwo.StorageZone
),
TrendAnalysis AS (
    SELECT 
        StorageZone,
        OperationDate,
        AvgDailyDwell,
        AVG(AvgDailyDwell) OVER (
            PARTITION BY StorageZone 
            ORDER BY OperationDate 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS SevenDayMovingAvg,
        AVG(AvgDailyDwell) OVER (
            PARTITION BY StorageZone 
            ORDER BY OperationDate 
            ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
        ) AS ThirtyDayMovingAvg
    FROM DailyDwell
)
SELECT 
    StorageZone,
    OperationDate,
    AvgDailyDwell,
    SevenDayMovingAvg,
    ThirtyDayMovingAvg,
    CASE 
        WHEN SevenDayMovingAvg > ThirtyDayMovingAvg * 1.2 THEN 'High Risk'
        WHEN SevenDayMovingAvg > ThirtyDayMovingAvg * 1.1 THEN 'Medium Risk'
        ELSE 'Normal'
    END AS BottleneckRisk
FROM TrendAnalysis
WHERE OperationDate = CAST(GETDATE() AS DATE)
ORDER BY SevenDayMovingAvg DESC
```

### Fleet Maintenance Priority Queue

```sql
-- Generate maintenance priority based on revenue impact
WITH VehicleUtilization AS (
    SELECT 
        VehicleID,
        COUNT(*) AS TripCount,
        SUM(DistanceMiles) AS TotalMiles,
        AVG(LoadWeightLbs) AS AvgLoadWeight,
        SUM(HarshBrakingEvents + HarshAccelerationEvents) AS StressEvents
    FROM FactFleetTrips
    WHERE TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY VehicleID
),
RevenueImpact AS (
    SELECT 
        fft.VehicleID,
        SUM(dpg.UnitValue * fwo.QuantityHandled) AS CargoValueHandled
    FROM FactFleetTrips fft
    INNER JOIN FactWarehouseOperations fwo 
        ON fwo.GeographyKey = fft.OriginGeographyKey
    INNER JOIN DimProductGravity dpg 
        ON fwo.ProductKey = dpg.ProductKey
    WHERE fft.TripStartTime >= DATEADD(DAY, -30, GETDATE())
    GROUP BY fft.VehicleID
)
SELECT 
    vu.VehicleID,
    vu.TripCount,
    vu.TotalMiles,
    vu.StressEvents,
    ri.CargoValueHandled,
    (vu.StressEvents * 0.3 + (vu.TotalMiles / 1000) * 0.5 + 
     (ri.CargoValueHandled / 100000) * 0.2) AS MaintenancePriorityScore
FROM VehicleUtilization vu
LEFT JOIN RevenueImpact ri ON vu.VehicleID = ri.VehicleID
ORDER BY MaintenancePriorityScore DESC
```

## Stored Procedures for Incremental Loading

### Incremental Warehouse Operations Load

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Incremental load from WMS staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationStartTime, OperationEndTime, QuantityHandled,
        StorageZone, WorkerID, EquipmentID, DwellTimeHours,
        CycleTimeSeconds, ErrorFlag, ErrorCode
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        st.OperationType,
        st.StartTime,
        st.EndTime,
        st.Quantity,
        st.Zone,
        st.WorkerID,
        st.EquipmentID,
        st.DwellHours,
        DATEDIFF(SECOND, st.StartTime, st.EndTime),
        CASE WHEN st.ErrorCode IS NOT NULL THEN 1 ELSE 0 END,
        st.ErrorCode
    FROM WMS_StagingOperations st
    LEFT JOIN DimTime dt 
        ON CAST(st.StartTime AS DATETIME2) = dt.DateTime
    LEFT JOIN DimGeography dg 
        ON st.LocationID = dg.LocationID
    LEFT JOIN DimProductGravity dp 
        ON st.SKU = dp.SKU
    WHERE st.LastModified > @LastLoadTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations fwo
            WHERE fwo.OperationStartTime = st.StartTime
                AND fwo.ProductKey = dp.ProductKey
        );
    
    -- Update last load timestamp
    UPDATE ETL_ControlTable 
    SET LastLoadTime = GETDATE() 
    WHERE TableName = 'FactWarehouseOperations';
END
GO
```

### Incremental Fleet Trips Load

```sql
CREATE PROCEDURE sp_LoadFleetTrips
    @LastLoadTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, VehicleID, DriverID, OriginGeographyKey, 
        DestinationGeographyKey, TripStartTime, TripEndTime,
        DistanceMiles, FuelConsumedGallons, IdleTimeMinutes,
        LoadingTimeMinutes, UnloadingTimeMinutes, LoadWeightLbs,
        AverageSpeedMPH, HarshBrakingEvents, HarshAccelerationEvents,
        WeatherDelayMinutes, TrafficDelayMinutes, OnTimeDelivery
    )
    SELECT 
        dt.TimeKey,
        st.VehicleID,
        st.DriverID,
        og.GeographyKey AS OriginKey,
        dg.GeographyKey AS DestKey,
        st.TripStart,
        st.TripEnd,
        st.Distance,
        st.FuelUsed,
        st.IdleMinutes,
        st.LoadTime,
        st.UnloadTime,
        st.LoadWeight,
        st.AvgSpeed,
        st.HarshBrakes,
        st.HarshAccel,
        st.WeatherDelay,
        st.TrafficDelay,
        CASE WHEN st.ActualArrival <= st.PromisedArrival THEN 1 ELSE 0 END
    FROM TelemetryStagingTrips st
    LEFT JOIN DimTime dt 
        ON CAST(st.TripStart AS DATETIME2) = dt.DateTime
    LEFT JOIN DimGeography og 
        ON st.OriginLocationID = og.LocationID
    LEFT JOIN DimGeography dg 
        ON st.DestLocationID = dg.LocationID
    WHERE st.LastModified > @LastLoadTime;
    
    UPDATE ETL_ControlTable 
    SET LastLoadTime = GETDATE() 
    WHERE TableName = 'FactFleetTrips';
END
GO
```

## Power BI DAX Measures

### Key Performance Indicators

```dax
// Total Warehouse Dwell Time (Hours)
TotalDwellTime = 
SUM(FactWarehouseOperations[DwellTimeHours])

// Average Cycle Time (Seconds)
AvgCycleTime = 
AVERAGE(FactWarehouseOperations[CycleTimeSeconds])

// Fleet Fuel Efficiency (MPG)
FleetFuelEfficiency = 
DIVIDE(
    SUM(FactFleetTrips[DistanceMiles]),
    SUM(FactFleetTrips[FuelConsumedGallons]),
    0
)

// On-Time Delivery Rate (%)
OnTimeDeliveryRate = 
DIVIDE(
    COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[OnTimeDelivery] = TRUE)),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Cross-Fact: Dwell Cost Impact
DwellCostImpact = 
VAR DwellHours = SUM(FactWarehouseOperations[DwellTimeHours])
VAR IdleMinutes = SUM(FactFleetTrips[IdleTimeMinutes])
VAR IdleCostPerMinute = 0.50
RETURN
    (DwellHours / 24) * (IdleMinutes * IdleCostPerMinute)

// Warehouse Gravity Score (Weighted)
WarehouseGravityScore = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[QuantityHandled] * 
    RELATED(DimProductGravity[GravityScore])
)

// Fleet Idle Time Percentage
FleetIdlePercentage = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[IdleTimeMinutes]) + 
    DATEDIFF(
        MIN(FactFleetTrips[TripStartTime]),
        MAX(FactFleetTrips[TripEndTime]),
        MINUTE
    ),
    0
) * 100

// Month-over-Month Growth (Generic Pattern)
MoMGrowth = 
VAR CurrentMonth = [TotalDwellTime]
VAR PreviousMonth = 
    CALCULATE(
        [TotalDwellTime],
        DATEADD(DimTime[DateTime], -1, MONTH)
    )
RETURN
    DIVIDE(CurrentMonth - PreviousMonth, PreviousMonth, 0) * 100
```

### Time Intelligence Measures

```dax
// Rolling 7-Day Average Dwell Time
DwellTime_7DayAvg = 
AVERAGEX(
    DATESINPERIOD(
        DimTime[DateTime],
        LASTDATE(DimTime[DateTime]),
        -7,
        DAY
    ),
    [TotalDwellTime]
)

// Year-to-Date Fleet Trips
YTD_FleetTrips = 
CALCULATE(
    COUNTROWS(FactFleetTrips),
    DATESYTD(DimTime[DateTime])
)

// Same Period Last Year
SPLY_DwellTime = 
CALCULATE(
    [TotalDwellTime],
    SAMEPERIODLASTYEAR(DimTime[DateTime])
)

// Year-over-Year Variance
YoY_Variance = 
[TotalDwellTime] - [SPLY_DwellTime]
```

## Alerting & Automation

### SQL Agent Job for Threshold Alerts

```sql
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    -- Check for excessive dwell time
    IF EXISTS (
        SELECT 1 
        FROM FactWarehouseOperations 
        WHERE DwellTimeHours > 72 
            AND OperationStartTime >= DATEADD(HOUR, -1, GETDATE())
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'ALERT: Excessive Warehouse Dwell Time Detected',
            @body = 'SKUs with dwell time exceeding 72 hours detected in the last hour.',
            @importance = 'High';
    END
    
    -- Check for high fleet idle time
    DECLARE @IdleThreshold DECIMAL(5,2) = 15.0;
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips 
        WHERE (IdleTimeMinutes * 100.0 / DATEDIFF(MINUTE, TripStartTime, TripEndTime)) > @IdleThreshold
            AND TripStartTime >= DATEADD(HOUR, -4, GETDATE())
    )
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'ALERT: High Fleet Idle Time Detected',
            @body = 'Vehicles with idle time exceeding 15% detected in the last 4 hours.',
            @importance = 'High';
    END
END
GO

-- Schedule the job to run every 15 minutes
EXEC msdb.dbo.sp_add_job
    @job_name = N'LogiFleet_KPI_Monitor',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep
    @job_name = N'LogiFleet_KPI_Monitor',
    @step_name = N'Check Thresholds',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_CheckKPIThresholds',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule
    @job_name = N'LogiFleet_KPI_Monitor',
    @schedule_name = N'Every15Minutes';

EXEC msdb.dbo.sp_add_jobserver
    @job_name = N'LogiFleet_KPI_Monitor';
```

## Power Automate Integration Example

### Trigger Flow from Power BI Alert

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "actions": {
      "Send_Teams_Message": {
        "inputs": {
          "body": {
            "messageBody": "<p>Warehouse dwell time exceeded threshold:<br>Zone: @{triggerBody()?['StorageZone']}<br>Dwell Hours: @{triggerBody()?['DwellHours']}</p>",
            "recipient": {
              "channelId": "${TEAMS_CHANNEL_ID}"
            }
          },
          "host": {
            "connection": {
              "name": "@parameters('$connections')['teams']['connectionId']"
            }
          },
          "method": "post",
          "path": "/v1.0/teams/@{encodeURIComponent('${TEAMS_TEAM_ID}')}/channels/@{encodeURIComponent('${TEAMS_CHANNEL_ID}')}/messages"
        },
        "type": "ApiConnection"
      }
    },
    "triggers": {
      "When_a_HTTP_request_is_received": {
        "inputs": {
          "schema": {
            "properties": {
              "DwellHours": { "type": "number" },
              "StorageZone": { "type": "string" }
            },
            "type": "object"
          }
        },
        "kind": "Http",
        "type": "Request"
      }
    }
  }
}
```

## Troubleshooting

### Issue: Slow Cross-Fact Queries

**Symptom:** Queries joining multiple fact tables take >10 seconds

**Solution:**
```sql
-- Create covering indexes on join keys
CREATE NONCLUSTERED INDEX IX_FactWarehouse_TimeProduct
ON FactWarehouseOperations (TimeKey, ProductKey, GeographyKey)
INCLUDE (DwellTimeHours, QuantityHandled);

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeOrigin
ON FactFleetTrips (TimeKey, OriginGeographyKey, VehicleID)
INCLUDE (IdleTimeMinutes, FuelConsumedGallons);

-- Consider columnstore index for analytical workloads
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, DwellTimeHours, QuantityHandled
);
```

### Issue: Power BI Refresh Failures

**Symptom:** Scheduled refresh fails with timeout errors

**Solution:**
```sql
-- Implement incremental refresh by creating a view with date filter
CREATE VIEW vw_FactWarehouseOperations_Incremental
AS
SELECT *
FROM FactWarehouseOperations
WHERE OperationStartTime >= DATEADD(MONTH, -3, GETDATE());

-- In Power BI, configure incremental refresh policy:
-- - Archive data older than 3 months
-- - Refresh last 7 days
```

### Issue: Missing Time Dimension Entries

**Symptom:** Facts not joining to DimTime

**Solution:**
```sql
-- Generate time dimension entries for extended range
DECLARE @StartDate DATETIME2 = '2025-01-01';
DECLARE @EndDate DATETIME2 = '2027-12-31';

;WITH TimeCTE AS (
    SELECT @StartDate AS DateTime
    UNION ALL
    SELECT DATEADD(MINUTE, 15, DateTime)
    FROM TimeCTE
    WHERE DateTime < @EndDate
)
INSERT INTO DimTime (TimeKey, DateTime, Hour, DayOfWeek, DayOfMonth, Month, Quarter, FiscalYear, IsWeekend, IsHoliday, ShiftNumber)
SELECT 
    CAST(FORMAT(DateTime, 'yyyyMMddHHmm') AS INT) AS TimeKey,
    DateTime,
    DATEPART(HOUR, DateTime) AS Hour,
    DATENAME(WEEKDAY, DateTime) AS DayOfWeek,
    DATEPART(DAY, DateTime) AS DayOfMonth,
    DATEPART(MONTH, DateTime) AS Month,
    DATEPART(QUARTER, DateTime) AS Quarter,
    YEAR(DateTime) AS FiscalYear,
    CASE WHEN DATEPART(WEEKDAY, DateTime) IN (1, 7) THEN 1 ELSE 0 END AS IsWeekend,
    0 AS IsHoliday, -- Populate separately
    
