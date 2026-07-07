---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - set up logistics analytics warehouse
  - configure logifleet pulse dashboard
  - implement supply chain data model
  - create fleet management analytics
  - deploy warehouse operations reporting
  - build multi-fact star schema for logistics
  - integrate power bi with fleet telemetry
  - setup supply chain intelligence platform
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection.

LogiFleet Pulse is an advanced data warehousing and analytics platform for logistics and supply chain management. It combines MS SQL Server's transactional integrity with Power BI visualization to create a unified semantic layer across warehouse operations, fleet telemetry, inventory management, and external data sources. The platform uses a custom multi-fact star schema with time-phased dimensions to enable cross-functional KPI analysis.

## What It Does

- **Multi-Modal Data Integration**: Combines warehouse management, fleet GPS/telematics, supplier data, weather/traffic APIs, and customer orders into a unified model
- **Cross-Fact KPI Analysis**: Links warehouse inventory turnover with fleet fuel consumption, dwell time with route efficiency
- **Predictive Analytics**: Identifies bottlenecks, generates proactive maintenance queues, and runs temporal simulation scenarios
- **Real-Time Dashboards**: Power BI reports refreshed every 15 minutes with role-based access control
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, item fragility, and replenishment lead times

## Installation

### Prerequisites

- MS SQL Server 2019+ (Enterprise, Standard, or Developer Edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, telematics APIs)

### Database Setup

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema**:
```sql
-- Connect to your SQL Server instance
-- Run the schema creation script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the main schema script
-- This creates fact tables, dimension tables, views, and stored procedures
:r .\sql\schema\01_create_dimensions.sql
:r .\sql\schema\02_create_facts.sql
:r .\sql\schema\03_create_bridges.sql
:r .\sql\schema\04_create_views.sql
:r .\sql\schema\05_create_procedures.sql
```

3. **Configure data sources**:
```json
// config.json (copy from config_sample.json)
{
  "database": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_USER}",
    "password": "${SQL_PASSWORD}"
  },
  "dataSources": {
    "wms": {
      "type": "sql",
      "connectionString": "${WMS_CONNECTION_STRING}"
    },
    "telemetry": {
      "type": "api",
      "endpoint": "${FLEET_API_ENDPOINT}",
      "apiKey": "${FLEET_API_KEY}"
    },
    "weather": {
      "type": "api",
      "endpoint": "https://api.weatherapi.com/v1",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refresh": {
    "intervalMinutes": 15,
    "incrementalLoad": true
  }
}
```

### Power BI Template Setup

1. **Open the template**:
```plaintext
File -> Open -> LogiFleet_Pulse_Master.pbit
```

2. **Configure data source connection**:
```plaintext
Server: your-sql-server.database.windows.net
Database: LogiFleetPulse
Authentication: SQL Server / Windows / Azure AD
```

3. **Refresh data model** to populate initial dataset

## Core Data Model

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking
```sql
CREATE TABLE FactWarehouseOperations (
    OperationID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20) NOT NULL, -- 'RECEIVE', 'PUTAWAY', 'PICK', 'PACK', 'SHIP'
    Quantity INT NOT NULL,
    DwellTimeMinutes INT,
    PickTimeSeconds INT,
    GravityZoneID INT,
    EmployeeKey INT,
    BatchID VARCHAR(50),
    CONSTRAINT FK_WO_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WO_Warehouse FOREIGN KEY (WarehouseKey) REFERENCES DimWarehouse(WarehouseKey),
    CONSTRAINT FK_WO_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);

-- Optimized indexes for common queries
CREATE NONCLUSTERED INDEX IX_WO_Time_Type 
    ON FactWarehouseOperations(TimeKey, OperationType) 
    INCLUDE (Quantity, DwellTimeMinutes);

CREATE NONCLUSTERED INDEX IX_WO_Product_Gravity 
    ON FactWarehouseOperations(ProductKey, GravityZoneID) 
    INCLUDE (DwellTimeMinutes);
```

**FactFleetTrips** - Vehicle route and telemetry data
```sql
CREATE TABLE FactFleetTrips (
    TripID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    DriverKey INT NOT NULL,
    RouteKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    DistanceKM DECIMAL(10,2),
    FuelLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AverageSpeedKPH DECIMAL(5,2),
    HarshBrakingEvents INT,
    DelayMinutes INT,
    DelayReasonKey INT,
    CONSTRAINT FK_FT_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FT_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

CREATE NONCLUSTERED INDEX IX_FT_Vehicle_Time 
    ON FactFleetTrips(VehicleKey, TimeKey) 
    INCLUDE (FuelLiters, IdleTimeMinutes);
```

**FactCrossDock** - Cross-docking operations
```sql
CREATE TABLE FactCrossDock (
    CrossDockID BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    InboundTripID BIGINT,
    OutboundTripID BIGINT,
    ProductKey INT NOT NULL,
    Quantity INT,
    DockToLoadMinutes INT,
    CONSTRAINT FK_CD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CD_Inbound FOREIGN KEY (InboundTripID) REFERENCES FactFleetTrips(TripID),
    CONSTRAINT FK_CD_Outbound FOREIGN KEY (OutboundTripID) REFERENCES FactFleetTrips(TripID)
);
```

### Key Dimensions

**DimTime** - 15-minute granularity time dimension
```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    Date DATE NOT NULL,
    Year INT NOT NULL,
    Quarter INT NOT NULL,
    Month INT NOT NULL,
    Week INT NOT NULL,
    DayOfMonth INT NOT NULL,
    DayOfWeek INT NOT NULL,
    Hour INT NOT NULL,
    MinuteInterval INT NOT NULL, -- 0, 15, 30, 45
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL
);

-- Populate time dimension
EXEC sp_PopulateTimeDimension @StartDate = '2024-01-01', @EndDate = '2030-12-31';
```

**DimProductGravity** - Product classification with gravity scoring
```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY,
    SKU VARCHAR(50) NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    Weight DECIMAL(10,2),
    Volume DECIMAL(10,2),
    FragilityScore INT, -- 1-10
    VelocityScore INT, -- 1-10 based on pick frequency
    ValueScore INT, -- 1-10 based on unit price
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + (11 - FragilityScore) * 0.2) PERSISTED,
    RecommendedZone AS (
        CASE 
            WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + (11 - FragilityScore) * 0.2) >= 7 THEN 'HIGH_GRAVITY'
            WHEN (VelocityScore * 0.5 + ValueScore * 0.3 + (11 - FragilityScore) * 0.2) >= 4 THEN 'MEDIUM_GRAVITY'
            ELSE 'LOW_GRAVITY'
        END
    ) PERSISTED
);
```

## Key Stored Procedures

### Incremental Data Loading

**sp_LoadWarehouseOperations** - ETL for warehouse facts
```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Get last successful load time if not provided
    IF @LastLoadDateTime IS NULL
    BEGIN
        SELECT @LastLoadDateTime = ISNULL(MAX(LastLoadTime), '1900-01-01')
        FROM ETL_LoadLog
        WHERE TableName = 'FactWarehouseOperations' AND Status = 'SUCCESS';
    END
    
    -- Insert new records from source WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, WarehouseKey, ProductKey, OperationType, 
        Quantity, DwellTimeMinutes, PickTimeSeconds, GravityZoneID
    )
    SELECT 
        dt.TimeKey,
        dw.WarehouseKey,
        dp.ProductKey,
        src.OperationType,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, src.StartTime, src.EndTime) AS PickTimeSeconds,
        dp.RecommendedZone
    FROM WMS_LINKED_SERVER.dbo.Operations src
    INNER JOIN DimTime dt ON DATEADD(MINUTE, (DATEPART(MINUTE, src.OperationTime) / 15) * 15, 
                                     DATEADD(HOUR, DATEDIFF(HOUR, 0, src.OperationTime), 0)) = dt.FullDateTime
    INNER JOIN DimWarehouse dw ON src.WarehouseID = dw.SourceWarehouseID
    INNER JOIN DimProductGravity dp ON src.SKU = dp.SKU
    WHERE src.OperationTime > @LastLoadDateTime;
    
    -- Log successful load
    INSERT INTO ETL_LoadLog (TableName, LastLoadTime, RowsInserted, Status)
    VALUES ('FactWarehouseOperations', GETDATE(), @@ROWCOUNT, 'SUCCESS');
END;
```

### Cross-Fact Analytics

**sp_GetFleetWarehouseDwellCorrelation** - Analyze relationship between warehouse dwell time and fleet delays
```sql
CREATE PROCEDURE sp_GetFleetWarehouseDwellCorrelation
    @StartDate DATE,
    @EndDate DATE,
    @WarehouseKey INT = NULL
AS
BEGIN
    SELECT 
        dw.WarehouseName,
        dt.Date,
        AVG(fwo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
        AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMin,
        AVG(fft.DelayMinutes) AS AvgDeliveryDelayMin,
        COUNT(DISTINCT fwo.ProductKey) AS UniqueProducts,
        SUM(fwo.Quantity) AS TotalQuantity,
        -- Correlation indicator (simplified)
        CASE 
            WHEN AVG(fwo.DwellTimeMinutes) > 120 AND AVG(fft.DelayMinutes) > 30 THEN 'HIGH_CORRELATION'
            WHEN AVG(fwo.DwellTimeMinutes) > 60 AND AVG(fft.DelayMinutes) > 15 THEN 'MODERATE_CORRELATION'
            ELSE 'LOW_CORRELATION'
        END AS CorrelationLevel
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
    LEFT JOIN FactFleetTrips fft ON dt.Date = (SELECT Date FROM DimTime WHERE TimeKey = fft.TimeKey)
        AND fft.OriginGeographyKey = dw.GeographyKey
    WHERE dt.Date BETWEEN @StartDate AND @EndDate
        AND (@WarehouseKey IS NULL OR fwo.WarehouseKey = @WarehouseKey)
    GROUP BY dw.WarehouseName, dt.Date
    ORDER BY dt.Date DESC, AvgWarehouseDwellMin DESC;
END;
```

### Predictive Bottleneck Detection

**sp_PredictBottlenecks** - Identify potential congestion points
```sql
CREATE PROCEDURE sp_PredictBottlenecks
    @LookaheadHours INT = 4
AS
BEGIN
    DECLARE @CurrentTime DATETIME = GETDATE();
    DECLARE @FutureTime DATETIME = DATEADD(HOUR, @LookaheadHours, @CurrentTime);
    
    -- Warehouse capacity bottlenecks
    SELECT 
        'WAREHOUSE' AS BottleneckType,
        dw.WarehouseName AS Location,
        CAST(SUM(fwo.Quantity) AS FLOAT) / dw.MaxCapacity AS CurrentUtilization,
        CASE 
            WHEN CAST(SUM(fwo.Quantity) AS FLOAT) / dw.MaxCapacity > 0.90 THEN 'CRITICAL'
            WHEN CAST(SUM(fwo.Quantity) AS FLOAT) / dw.MaxCapacity > 0.80 THEN 'WARNING'
            ELSE 'NORMAL'
        END AS AlertLevel,
        dw.MaxCapacity - SUM(fwo.Quantity) AS AvailableCapacity
    FROM FactWarehouseOperations fwo
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    INNER JOIN DimWarehouse dw ON fwo.WarehouseKey = dw.WarehouseKey
    WHERE dt.FullDateTime >= DATEADD(HOUR, -24, @CurrentTime)
        AND fwo.OperationType IN ('RECEIVE', 'PUTAWAY')
    GROUP BY dw.WarehouseName, dw.MaxCapacity
    HAVING CAST(SUM(fwo.Quantity) AS FLOAT) / dw.MaxCapacity > 0.70
    
    UNION ALL
    
    -- Fleet route congestion
    SELECT 
        'ROUTE' AS BottleneckType,
        dr.RouteName AS Location,
        COUNT(*) AS ActiveTrips,
        CASE 
            WHEN COUNT(*) > dr.OptimalTripCount * 1.5 THEN 'CRITICAL'
            WHEN COUNT(*) > dr.OptimalTripCount * 1.2 THEN 'WARNING'
            ELSE 'NORMAL'
        END AS AlertLevel,
        AVG(fft.DelayMinutes) AS AvgDelayMinutes
    FROM FactFleetTrips fft
    INNER JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    INNER JOIN DimRoute dr ON fft.RouteKey = dr.RouteKey
    WHERE dt.FullDateTime >= DATEADD(HOUR, -2, @CurrentTime)
    GROUP BY dr.RouteName, dr.OptimalTripCount
    HAVING COUNT(*) > dr.OptimalTripCount
    ORDER BY AlertLevel DESC, CurrentUtilization DESC;
END;
```

## Power BI DAX Measures

### Cross-Fact KPIs

**Total Logistics Cost Per Unit**
```dax
Total Logistics Cost Per Unit = 
VAR WarehouseCost = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[DwellTimeMinutes] * 0.15 // Cost per minute storage
    )
VAR FleetCost = 
    SUMX(
        FactFleetTrips,
        (FactFleetTrips[FuelLiters] * 1.50) + // Fuel cost
        (FactFleetTrips[IdleTimeMinutes] * 0.75) // Idle cost
    )
VAR TotalUnits = SUM(FactWarehouseOperations[Quantity])
RETURN
    DIVIDE(WarehouseCost + FleetCost, TotalUnits, 0)
```

**Warehouse-Fleet Efficiency Score**
```dax
Efficiency Score = 
VAR AvgDwellTime = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgPickTime = AVERAGE(FactWarehouseOperations[PickTimeSeconds])
VAR AvgFleetIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR AvgDelay = AVERAGE(FactFleetTrips[DelayMinutes])

// Normalized scores (0-100, higher is better)
VAR DwellScore = 100 - DIVIDE(AvgDwellTime, 240, 0) * 100 // 240 min = 4 hours baseline
VAR PickScore = 100 - DIVIDE(AvgPickTime, 300, 0) * 100 // 300 sec = 5 min baseline
VAR IdleScore = 100 - DIVIDE(AvgFleetIdle, 60, 0) * 100 // 60 min baseline
VAR DelayScore = 100 - DIVIDE(AvgDelay, 45, 0) * 100 // 45 min baseline

RETURN
    (DwellScore * 0.3) + (PickScore * 0.2) + (IdleScore * 0.25) + (DelayScore * 0.25)
```

**Gravity Zone Compliance**
```dax
Gravity Zone Compliance % = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR CompliantOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[GravityZoneID] = RELATED(DimProductGravity[RecommendedZone])
    )
RETURN
    DIVIDE(CompliantOps, TotalOps, 0) * 100
```

### Time Intelligence

**Rolling 7-Day Fleet Utilization**
```dax
Fleet Utilization 7D = 
VAR Last7Days = 
    CALCULATETABLE(
        VALUES(DimTime[Date]),
        DATESINPERIOD(DimTime[Date], LASTDATE(DimTime[Date]), -7, DAY)
    )
VAR TotalAvailableHours = COUNTROWS(Last7Days) * 24 * [Total Fleet Size]
VAR TotalActiveHours = 
    CALCULATE(
        SUMX(FactFleetTrips, FactFleetTrips[DistanceKM] / FactFleetTrips[AverageSpeedKPH]),
        Last7Days
    )
RETURN
    DIVIDE(TotalActiveHours, TotalAvailableHours, 0) * 100
```

## Common Workflows

### Daily Operations Dashboard Setup

1. **Create automated refresh schedule**:
```sql
-- SQL Server Agent Job for incremental loads
EXEC msdb.dbo.sp_add_job 
    @job_name = N'LogiFleet_IncrementalLoad',
    @enabled = 1;

EXEC msdb.dbo.sp_add_jobstep 
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load_Warehouse_Data',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_LoadWarehouseOperations;',
    @database_name = N'LogiFleetPulse';

EXEC msdb.dbo.sp_add_jobstep 
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load_Fleet_Data',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_LoadFleetTrips;',
    @database_name = N'LogiFleetPulse';

-- Schedule every 15 minutes
EXEC msdb.dbo.sp_add_schedule 
    @schedule_name = N'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC msdb.dbo.sp_attach_schedule 
    @job_name = N'LogiFleet_IncrementalLoad',
    @schedule_name = N'Every15Minutes';
```

2. **Configure Power BI Service refresh** (via Power BI REST API):
```powershell
# PowerShell script for automated Power BI refresh
$workspaceId = $env:POWERBI_WORKSPACE_ID
$datasetId = $env:POWERBI_DATASET_ID
$apiUrl = "https://api.powerbi.com/v1.0/myorg/groups/$workspaceId/datasets/$datasetId/refreshes"

$headers = @{
    "Authorization" = "Bearer $env:POWERBI_ACCESS_TOKEN"
    "Content-Type" = "application/json"
}

$body = @{
    "notifyOption" = "MailOnFailure"
} | ConvertTo-Json

Invoke-RestMethod -Uri $apiUrl -Method Post -Headers $headers -Body $body
```

### Gravity Zone Optimization

**Identify misplaced products**:
```sql
-- Find high-gravity products in low-gravity zones
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.RecommendedZone,
    fwo.GravityZoneID AS CurrentZone,
    COUNT(*) AS PickFrequency,
    AVG(fwo.PickTimeSeconds) AS AvgPickTimeSeconds
FROM FactWarehouseOperations fwo
INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
WHERE fwo.OperationType = 'PICK'
    AND dp.RecommendedZone != fwo.GravityZoneID
    AND dp.GravityScore >= 7 -- High gravity
    AND fwo.GravityZoneID = 'LOW_GRAVITY'
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, dp.RecommendedZone, fwo.GravityZoneID
HAVING COUNT(*) > 50 -- Frequent picks
ORDER BY PickFrequency DESC;
```

**Generate relocation recommendations**:
```sql
CREATE PROCEDURE sp_GenerateRelocationPlan
AS
BEGIN
    -- Create temp table for recommendations
    SELECT 
        dp.ProductKey,
        dp.SKU,
        dp.GravityScore,
        dp.RecommendedZone,
        fwo.GravityZoneID AS CurrentZone,
        COUNT(*) AS DailyPickFrequency,
        AVG(fwo.PickTimeSeconds) AS AvgPickTime,
        -- Estimated time savings
        CASE 
            WHEN dp.RecommendedZone = 'HIGH_GRAVITY' AND fwo.GravityZoneID = 'LOW_GRAVITY' 
                THEN AVG(fwo.PickTimeSeconds) * 0.40 -- 40% time reduction
            WHEN dp.RecommendedZone = 'HIGH_GRAVITY' AND fwo.GravityZoneID = 'MEDIUM_GRAVITY' 
                THEN AVG(fwo.PickTimeSeconds) * 0.20 -- 20% time reduction
            WHEN dp.RecommendedZone = 'MEDIUM_GRAVITY' AND fwo.GravityZoneID = 'LOW_GRAVITY' 
                THEN AVG(fwo.PickTimeSeconds) * 0.25 -- 25% time reduction
            ELSE 0
        END AS EstimatedTimeSavingsSeconds,
        -- Priority score
        COUNT(*) * CASE 
            WHEN dp.RecommendedZone = 'HIGH_GRAVITY' AND fwo.GravityZoneID = 'LOW_GRAVITY' THEN 3
            WHEN dp.RecommendedZone = 'HIGH_GRAVITY' AND fwo.GravityZoneID = 'MEDIUM_GRAVITY' THEN 2
            ELSE 1
        END AS PriorityScore
    INTO #RelocationPlan
    FROM FactWarehouseOperations fwo
    INNER JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
    WHERE fwo.OperationType = 'PICK'
        AND dp.RecommendedZone != fwo.GravityZoneID
        AND dt.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY dp.ProductKey, dp.SKU, dp.GravityScore, dp.RecommendedZone, fwo.GravityZoneID;
    
    -- Return prioritized recommendations
    SELECT TOP 100
        SKU,
        CurrentZone,
        RecommendedZone,
        DailyPickFrequency,
        AvgPickTime,
        EstimatedTimeSavingsSeconds,
        EstimatedTimeSavingsSeconds * DailyPickFrequency AS DailyTimeSavingsSeconds,
        PriorityScore
    FROM #RelocationPlan
    WHERE EstimatedTimeSavingsSeconds > 0
    ORDER BY PriorityScore DESC, DailyPickFrequency DESC;
    
    DROP TABLE #RelocationPlan;
END;
```

### Fleet Maintenance Triage

**Proactive maintenance queue**:
```sql
-- Rank vehicles by maintenance urgency + revenue impact
WITH VehicleMetrics AS (
    SELECT 
        dv.VehicleID,
        dv.VehicleType,
        dv.LastMaintenanceDate,
        COUNT(fft.TripID) AS RecentTrips,
        AVG(fft.FuelLiters / NULLIF(fft.DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(fft.HarshBrakingEvents) AS TotalHarshBraking,
        AVG(fft.IdleTimeMinutes) AS AvgIdleTime,
        SUM(CASE WHEN fft.DelayMinutes > 30 THEN 1 ELSE 0 END) AS DelayIncidents,
        -- Revenue impact (simplified)
        SUM(fwo.Quantity) * AVG(dp.ValueScore) AS EstimatedRevenueImpact
    FROM DimVehicle dv
    LEFT JOIN FactFleetTrips fft ON dv.VehicleKey = fft.VehicleKey
    LEFT JOIN DimTime dt ON fft.TimeKey = dt.TimeKey
    LEFT JOIN FactCrossDock fcd ON fft.TripID IN (fcd.InboundTripID, fcd.OutboundTripID)
    LEFT JOIN FactWarehouseOperations fwo ON fcd.ProductKey = fwo.ProductKey
    LEFT JOIN DimProductGravity dp ON fwo.ProductKey = dp.ProductKey
    WHERE dt.Date >= DATEADD(DAY, -30, GETDATE())
    GROUP BY dv.VehicleID, dv.VehicleType, dv.LastMaintenanceDate
),
MaintenanceScore AS (
    SELECT 
        *,
        -- Composite urgency score (0-100)
        (
            (DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) / 365.0 * 30) + -- Age factor
            (AvgIdleTime / 60.0 * 15) + -- Idle time factor
            (TotalHarshBraking / 100.0 * 20) + -- Driving behavior factor
            (DelayIncidents * 5) + -- Reliability factor
            (CASE WHEN AvgFuelEfficiency > 0.12 THEN 30 ELSE 0 END) -- Fuel efficiency threshold
        ) AS UrgencyScore,
        -- Revenue impact tier
        NTILE(5) OVER (ORDER BY EstimatedRevenueImpact DESC) AS RevenueTier
    FROM VehicleMetrics
)
SELECT 
    VehicleID,
    VehicleType,
    DATEDIFF(DAY, LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
    RecentTrips,
    AvgFuelEfficiency,
    TotalHarshBraking,
    DelayIncidents,
    CAST(UrgencyScore AS DECIMAL(5,1)) AS MaintenanceUrgency,
    RevenueTier,
    -- Final priority combines urgency and revenue impact
    CAST(UrgencyScore * (1 + RevenueTier * 0.15) AS DECIMAL(5,1)) AS FinalPriority,
    CASE 
        WHEN UrgencyScore > 70 AND RevenueTier >= 4 THEN 'CRITICAL_HIGH_VALUE'
        WHEN UrgencyScore > 70 THEN 'CRITICAL'
        WHEN UrgencyScore > 50 THEN 'HIGH'
        WHEN UrgencyScore > 30 THEN 'MEDIUM'
        ELSE 'LOW'
    END AS PriorityLevel
FROM MaintenanceScore
WHERE UrgencyScore > 30
ORDER BY FinalPriority DESC;
```

## Alerting System

### Configure threshold-based alerts

**sp_CheckAndAlert** - Monitor KPI breaches
```sql
CREATE PROCEDURE sp_CheckAndAlert
AS
BEGIN
    
