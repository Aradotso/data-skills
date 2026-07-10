---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for multi-modal logistics intelligence with cross-fact KPI harmonization
triggers:
  - set up logifleet pulse supply chain analytics
  - configure logistics data warehouse with power bi
  - implement multi-fact star schema for fleet and warehouse data
  - build cross-modal supply chain dashboard
  - deploy logifleet warehouse gravity zones
  - create logistics intelligence platform with sql server
  - integrate fleet telemetry with warehouse operations analytics
  - setup adaptive supply chain pulse monitoring
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that combines:

- **Multi-fact star schema data warehouse** for warehouse operations, fleet trips, and cross-dock operations
- **Power BI dashboards** with real-time refresh capabilities
- **Cross-fact KPI harmonization** linking inventory turnover with fleet metrics
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and value
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Role-based access control** with row-level security

The platform integrates data from WMS, GPS/telematics, supplier portals, weather APIs, and customer order systems into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to warehouse management system and fleet telemetry data sources

### Step 1: Deploy SQL Schema

```sql
-- Create the main database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeStamp DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    Hour INT NOT NULL,
    QuarterHour INT NOT NULL, -- 15-minute buckets
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    MonthName VARCHAR(20) NOT NULL,
    Quarter INT NOT NULL,
    Year INT NOT NULL,
    FiscalPeriod VARCHAR(20) NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
)

CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- 'Warehouse', 'Route Node', 'Distribution Center'
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1
)

CREATE NONCLUSTERED INDEX IX_Geography_LocationID ON DimGeography(LocationID)
CREATE NONCLUSTERED INDEX IX_Geography_Type ON DimGeography(LocationType)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    GravityScore DECIMAL(5,2) NOT NULL, -- Calculated: velocity * value * fragility
    VelocityRank INT, -- Pick frequency ranking
    ValueRank INT, -- Unit value ranking
    FragilityScore DECIMAL(3,2), -- 0-1 scale
    StorageZoneRecommendation VARCHAR(50), -- 'High-Gravity', 'Medium-Gravity', 'Low-Gravity'
    CurrentStorageZone VARCHAR(50),
    LastGravityRecalculation DATETIME2,
    IsActive BIT DEFAULT 1
)

CREATE NONCLUSTERED INDEX IX_Product_SKU ON DimProductGravity(SKU)
CREATE NONCLUSTERED INDEX IX_Product_Gravity ON DimProductGravity(GravityScore DESC)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    Country VARCHAR(100),
    AverageLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2), -- Standard deviation
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryPercentage DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2), -- 0-100 scale
    ReliabilityGrade VARCHAR(2), -- A+, A, B, C, D
    LastEvaluationDate DATETIME2,
    IsActive BIT DEFAULT 1
)

CREATE NONCLUSTERED INDEX IX_Supplier_ID ON DimSupplierReliability(SupplierID)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime),
    DwellTimeHours DECIMAL(8,2), -- Time item spent in storage zone
    QuantityHandled INT NOT NULL,
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50), -- Forklift, conveyor, etc.
    FromZone VARCHAR(50),
    ToZone VARCHAR(50),
    IsDelayed BIT DEFAULT 0,
    DelayReasonCode VARCHAR(50),
    CostUSD DECIMAL(12,2)
)

CREATE CLUSTERED COLUMNSTORE INDEX CCI_WarehouseOps ON FactWarehouseOperations
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time ON FactWarehouseOperations(TimeKey, OperationType)
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Product ON FactWarehouseOperations(ProductKey)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    TripStartTime DATETIME2 NOT NULL,
    TripEndTime DATETIME2,
    PlannedDurationMinutes INT,
    ActualDurationMinutes AS DATEDIFF(MINUTE, TripStartTime, TripEndTime),
    DistanceMiles DECIMAL(8,2),
    FuelConsumedGallons DECIMAL(8,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    LoadWeightPounds INT,
    AverageSpeedMPH DECIMAL(5,2),
    WeatherCondition VARCHAR(50), -- From external API
    TrafficDelayMinutes INT, -- From external API
    MaintenanceAlertFlag BIT DEFAULT 0,
    MaintenanceAlertType VARCHAR(100),
    TripCostUSD DECIMAL(12,2),
    RevenueUSD DECIMAL(12,2)
)

CREATE CLUSTERED COLUMNSTORE INDEX CCI_FleetTrips ON FactFleetTrips
CREATE NONCLUSTERED INDEX IX_FleetTrips_Time ON FactFleetTrips(TimeKey)
CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle ON FactFleetTrips(VehicleID, TripStartTime)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ArrivalTime DATETIME2 NOT NULL,
    DepartureTime DATETIME2,
    DockDwellMinutes AS DATEDIFF(MINUTE, ArrivalTime, DepartureTime),
    QuantityTransferred INT NOT NULL,
    RequiresInspection BIT DEFAULT 0,
    InspectionTimeMinutes INT,
    IsExpedited BIT DEFAULT 0,
    CrossDockCostUSD DECIMAL(12,2)
)

CREATE CLUSTERED COLUMNSTORE INDEX CCI_CrossDock ON FactCrossDock
CREATE NONCLUSTERED INDEX IX_CrossDock_Time ON FactCrossDock(TimeKey)
GO
```

### Step 2: Create Stored Procedures for Data Loading

```sql
-- Incremental loading procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example: Load from WMS staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, OperationStartTime, OperationEndTime,
        QuantityHandled, OperatorID, EquipmentID,
        FromZone, ToZone, IsDelayed, DelayReasonCode, CostUSD
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        ds.SupplierKey,
        stg.OperationType,
        stg.OperationStartTime,
        stg.OperationEndTime,
        stg.QuantityHandled,
        stg.OperatorID,
        stg.EquipmentID,
        stg.FromZone,
        stg.ToZone,
        stg.IsDelayed,
        stg.DelayReasonCode,
        stg.CostUSD
    FROM WMS_Staging.dbo.Operations stg
    INNER JOIN DimTime dt ON dt.DateTimeStamp = DATEADD(MINUTE, (DATEPART(MINUTE, stg.OperationStartTime) / 15) * 15, CAST(CAST(stg.OperationStartTime AS DATE) AS DATETIME2) + CAST(CAST(DATEPART(HOUR, stg.OperationStartTime) AS VARCHAR) + ':00' AS TIME))
    INNER JOIN DimGeography dg ON dg.LocationID = stg.WarehouseID
    INNER JOIN DimProductGravity dp ON dp.SKU = stg.SKU
    LEFT JOIN DimSupplierReliability ds ON ds.SupplierID = stg.SupplierID
    WHERE stg.OperationStartTime > @LastLoadDateTime
    
    RETURN @@ROWCOUNT
END
GO

-- Calculate Gravity Score for products
CREATE PROCEDURE sp_RecalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH VelocityCalc AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) AS VelocityRank
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    ),
    ValueCalc AS (
        SELECT 
            ProductKey,
            AVG(CostUSD / NULLIF(QuantityHandled, 0)) AS AvgUnitValue,
            ROW_NUMBER() OVER (ORDER BY AVG(CostUSD / NULLIF(QuantityHandled, 0)) DESC) AS ValueRank
        FROM FactWarehouseOperations
        WHERE OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    )
    UPDATE dp
    SET 
        dp.GravityScore = (
            (1.0 - (vc.VelocityRank * 1.0 / (SELECT MAX(VelocityRank) FROM VelocityCalc))) * 40 +
            (1.0 - (val.ValueRank * 1.0 / (SELECT MAX(ValueRank) FROM ValueCalc))) * 35 +
            dp.FragilityScore * 25
        ),
        dp.VelocityRank = vc.VelocityRank,
        dp.ValueRank = val.ValueRank,
        dp.StorageZoneRecommendation = CASE 
            WHEN (1.0 - (vc.VelocityRank * 1.0 / (SELECT MAX(VelocityRank) FROM VelocityCalc))) * 40 + (1.0 - (val.ValueRank * 1.0 / (SELECT MAX(ValueRank) FROM ValueCalc))) * 35 + dp.FragilityScore * 25 >= 70 THEN 'High-Gravity'
            WHEN (1.0 - (vc.VelocityRank * 1.0 / (SELECT MAX(VelocityRank) FROM VelocityCalc))) * 40 + (1.0 - (val.ValueRank * 1.0 / (SELECT MAX(ValueRank) FROM ValueCalc))) * 35 + dp.FragilityScore * 25 >= 40 THEN 'Medium-Gravity'
            ELSE 'Low-Gravity'
        END,
        dp.LastGravityRecalculation = GETDATE()
    FROM DimProductGravity dp
    LEFT JOIN VelocityCalc vc ON vc.ProductKey = dp.ProductKey
    LEFT JOIN ValueCalc val ON val.ProductKey = dp.ProductKey
    WHERE dp.IsActive = 1
END
GO

-- Alert procedure for threshold breaches
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Fleet idling time alert
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips
        WHERE TripStartTime >= DATEADD(HOUR, -4, GETDATE())
            AND (IdleTimeMinutes * 100.0 / NULLIF(ActualDurationMinutes, 0)) > 15
    )
    BEGIN
        -- Insert into alert table or send notification
        INSERT INTO AlertLog (AlertType, AlertMessage, AlertTime, Severity)
        SELECT 
            'FleetIdling',
            'Vehicle ' + VehicleID + ' exceeded 15% idle time threshold',
            GETDATE(),
            'Warning'
        FROM FactFleetTrips
        WHERE TripStartTime >= DATEADD(HOUR, -4, GETDATE())
            AND (IdleTimeMinutes * 100.0 / NULLIF(ActualDurationMinutes, 0)) > 15
    END
    
    -- Warehouse dwell time alert
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
        WHERE wo.OperationStartTime >= DATEADD(HOUR, -24, GETDATE())
            AND wo.DwellTimeHours > 72
            AND dp.StorageZoneRecommendation = 'High-Gravity'
    )
    BEGIN
        INSERT INTO AlertLog (AlertType, AlertMessage, AlertTime, Severity)
        SELECT 
            'HighGravityDwell',
            'High-gravity SKU ' + dp.SKU + ' has dwell time > 72 hours',
            GETDATE(),
            'Critical'
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
        WHERE wo.OperationStartTime >= DATEADD(HOUR, -24, GETDATE())
            AND wo.DwellTimeHours > 72
            AND dp.StorageZoneRecommendation = 'High-Gravity'
    END
END
GO

-- Create alert log table
CREATE TABLE AlertLog (
    AlertID BIGINT PRIMARY KEY IDENTITY(1,1),
    AlertType VARCHAR(50) NOT NULL,
    AlertMessage VARCHAR(1000) NOT NULL,
    AlertTime DATETIME2 NOT NULL,
    Severity VARCHAR(20) NOT NULL, -- 'Info', 'Warning', 'Critical'
    IsAcknowledged BIT DEFAULT 0,
    AcknowledgedBy VARCHAR(100),
    AcknowledgedTime DATETIME2
)
GO
```

### Step 3: Configure Power BI Connection

Create a `config.json` file (not committed to source control):

```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER.database.windows.net",
    "database": "LogiFleetPulse",
    "authentication": "sql_login",
    "username_env": "LOGIFLEET_DB_USER",
    "password_env": "LOGIFLEET_DB_PASSWORD"
  },
  "refresh_schedule": {
    "enabled": true,
    "interval_minutes": 15
  },
  "external_apis": {
    "weather_api_key_env": "WEATHER_API_KEY",
    "traffic_api_key_env": "TRAFFIC_API_KEY"
  }
}
```

Environment variables should be set:
```bash
export LOGIFLEET_DB_USER="your_username"
export LOGIFLEET_DB_PASSWORD="your_secure_password"
export WEATHER_API_KEY="your_weather_api_key"
export TRAFFIC_API_KEY="your_traffic_api_key"
```

## Power BI Setup

### Import Template

1. Open Power BI Desktop
2. File → Import → Power BI Template (`.pbit` file from repository)
3. Enter connection parameters when prompted
4. The model will automatically detect relationships based on foreign keys

### Key DAX Measures

```dax
// Cross-Fact KPI: Cost per Mile with Dwell Penalty
CostPerMile_DwellAdjusted = 
VAR FleetCost = SUM(FactFleetTrips[TripCostUSD])
VAR TotalMiles = SUM(FactFleetTrips[DistanceMiles])
VAR WarehouseCost = SUM(FactWarehouseOperations[CostUSD])
VAR HighDwellPenalty = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[DwellTimeHours] > 72
    ) * 50 // $50 penalty per high-dwell occurrence
RETURN
DIVIDE(FleetCost + WarehouseCost + HighDwellPenalty, TotalMiles, 0)

// Warehouse Gravity Utilization Score
GravityUtilizationScore = 
VAR HighGravityMisplaced = 
    CALCULATE(
        COUNTROWS(DimProductGravity),
        DimProductGravity[StorageZoneRecommendation] = "High-Gravity",
        DimProductGravity[CurrentStorageZone] <> "High-Gravity"
    )
VAR TotalHighGravity = 
    CALCULATE(
        COUNTROWS(DimProductGravity),
        DimProductGravity[StorageZoneRecommendation] = "High-Gravity"
    )
RETURN
1 - DIVIDE(HighGravityMisplaced, TotalHighGravity, 0)

// Fleet Idling Efficiency
FleetIdlingPercentage = 
DIVIDE(
    SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[ActualDurationMinutes]),
    0
) * 100

// Predictive Bottleneck Index (time-based)
BottleneckIndex = 
VAR CurrentHour = HOUR(NOW())
VAR HistoricalDelays = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        FactWarehouseOperations[IsDelayed] = TRUE,
        HOUR(FactWarehouseOperations[OperationStartTime]) = CurrentHour
    )
VAR TotalOps = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        HOUR(FactWarehouseOperations[OperationStartTime]) = CurrentHour
    )
RETURN
DIVIDE(HistoricalDelays, TotalOps, 0) * 100

// Supplier Reliability Impact
SupplierImpactScore = 
AVERAGE(DimSupplierReliability[ComplianceScore]) * 
(1 - AVERAGE(DimSupplierReliability[DefectPercentage]) / 100)
```

### Row-Level Security

```dax
// Create roles in Power BI Desktop
// Role: WarehouseManager
[DimGeography[LocationType]] = "Warehouse" && [DimGeography[Region]] = USERPRINCIPALNAME()

// Role: FleetSupervisor  
[FactFleetTrips[DriverID]] IN LOOKUPVALUE(
    UserAccessTable[EmployeeID],
    UserAccessTable[Email], USERPRINCIPALNAME()
)

// Role: Executive (see all data)
1=1
```

## Common Usage Patterns

### Query: Cross-Fact Analysis

```sql
-- Find high-gravity SKUs with excessive dwell time that departed on delayed trips
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    AVG(wo.DwellTimeHours) AS AvgDwellHours,
    COUNT(DISTINCT ft.TripKey) AS AffectedTrips,
    AVG(ft.ActualDurationMinutes - ft.PlannedDurationMinutes) AS AvgDelayMinutes,
    SUM(ft.TripCostUSD + wo.CostUSD) AS TotalCostImpact
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
INNER JOIN FactCrossDock cd ON wo.GeographyKey = cd.GeographyKey 
    AND wo.ProductKey = cd.ProductKey
INNER JOIN FactFleetTrips ft ON cd.OutboundTripKey = ft.TripKey
WHERE dp.StorageZoneRecommendation = 'High-Gravity'
    AND wo.DwellTimeHours > 48
    AND ft.ActualDurationMinutes > ft.PlannedDurationMinutes
    AND wo.OperationStartTime >= DATEADD(MONTH, -1, GETDATE())
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore
HAVING AVG(wo.DwellTimeHours) > 60
ORDER BY TotalCostImpact DESC
```

### Query: Fleet Maintenance Priority Queue

```sql
-- Generate proactive maintenance queue weighted by revenue impact
WITH VehicleRisk AS (
    SELECT 
        VehicleID,
        COUNT(CASE WHEN MaintenanceAlertFlag = 1 THEN 1 END) AS AlertCount,
        AVG(LoadWeightPounds) AS AvgLoad,
        SUM(RevenueUSD) AS TotalRevenue,
        AVG(IdleTimeMinutes * 1.0 / NULLIF(ActualDurationMinutes, 0)) AS AvgIdleRatio
    FROM FactFleetTrips
    WHERE TripStartTime >= DATEADD(DAY, -7, GETDATE())
    GROUP BY VehicleID
)
SELECT 
    vr.VehicleID,
    vr.AlertCount,
    vr.TotalRevenue,
    vr.AvgIdleRatio,
    -- Weighted priority score
    (vr.AlertCount * 40 + 
     (vr.TotalRevenue / 1000) * 35 + 
     vr.AvgIdleRatio * 100 * 25) AS PriorityScore
FROM VehicleRisk vr
WHERE vr.AlertCount > 0
ORDER BY PriorityScore DESC
```

### Power BI: Custom Visual with Python

```python
# In Power BI, create a Python visual to show gravity zone optimization
import matplotlib.pyplot as plt
import numpy as np

# dataset comes from Power BI context
gravity_scores = dataset['GravityScore'].values
current_zones = dataset['CurrentStorageZone'].values
recommended_zones = dataset['StorageZoneRecommendation'].values

misalignment = np.where(current_zones != recommended_zones, 1, 0)

fig, ax = plt.subplots(figsize=(10, 6))
scatter = ax.scatter(
    range(len(gravity_scores)), 
    gravity_scores,
    c=misalignment,
    cmap='RdYlGn_r',
    s=100,
    alpha=0.6
)
ax.set_xlabel('Product Index')
ax.set_ylabel('Gravity Score')
ax.set_title('Warehouse Gravity Zone Alignment')
ax.axhline(y=70, color='r', linestyle='--', label='High-Gravity Threshold')
ax.axhline(y=40, color='orange', linestyle='--', label='Medium-Gravity Threshold')
ax.legend()
plt.colorbar(scatter, label='Misaligned (Red=Yes)')
plt.tight_layout()
plt.show()
```

## Configuration

### External API Integration

```sql
-- Create external table for weather data (requires PolyBase or external data source)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = 'https://api.weatherprovider.com/v1/',
    CREDENTIAL = WeatherAPICredential
)

CREATE EXTERNAL TABLE ExternalWeather (
    LocationID VARCHAR(50),
    Timestamp DATETIME2,
    Condition VARCHAR(100),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = WeatherAPI,
    LOCATION = 'weather/historical',
    FILE_FORMAT = JSON_FORMAT
)
```

### Automated Refresh with SQL Agent

```sql
-- Create SQL Agent job for hourly refresh
USE msdb
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_IncrementalLoad',
    @enabled = 1,
    @description = N'Incremental load of warehouse and fleet data'
GO

EXEC dbo.sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @command = N'
        DECLARE @LastLoad DATETIME2
        SELECT @LastLoad = MAX(OperationStartTime) FROM FactWarehouseOperations
        EXEC sp_LoadWarehouseOperations @LastLoad
    ',
    @database_name = N'LogiFleetPulse'
GO

EXEC dbo.sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Recalculate Gravity Scores',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_RecalculateProductGravity',
    @database_name = N'LogiFleetPulse'
GO

EXEC dbo.sp_add_jobstep
    @job_name = N'LogiFleet_IncrementalLoad',
    @step_name = N'Check Threshold Alerts',
    @subsystem = N'TSQL',
    @command = N'EXEC sp_CheckKPIThresholds',
    @database_name = N'LogiFleetPulse'
GO

EXEC dbo.sp_add_schedule
    @schedule_name = N'Hourly',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hours
    @freq_subday_interval = 1,
    @active_start_time = 000000
GO

EXEC dbo.sp_attach_schedule
    @job_name = N'LogiFleet_IncrementalLoad',
    @schedule_name = N'Hourly'
GO

EXEC dbo.sp_add_jobserver
    @job_name = N'LogiFleet_IncrementalLoad'
GO
```

## Troubleshooting

### Issue: Power BI Refresh Fails

**Symptoms**: "Unable to connect to data source" error

**Solution**:
```sql
-- Check SQL Server authentication mode
SELECT SERVERPROPERTY('IsIntegratedSecurityOnly') AS [Authentication Mode]
-- Should return 0 for mixed mode

-- Verify login exists and has proper permissions
USE LogiFleetPulse
GO
CREATE USER [PowerBIServiceAccount] FOR LOGIN [PowerBIServiceAccount]
ALTER ROLE db_datareader ADD MEMBER [PowerBIServiceAccount]
GRANT EXECUTE ON SCHEMA::dbo TO [PowerBIServiceAccount]
GO
```

### Issue: Slow Cross-Fact Queries

**Symptoms**: Queries joining FactWarehouseOperations and FactFleetTrips timeout

**Solution**:
```sql
-- Add composite indexes on frequently joined columns
CREATE NONCLUSTERED INDEX IX_Warehouse_TimeGeo 
ON FactWarehouseOperations(TimeKey, GeographyKey, ProductKey)
INCLUDE (DwellTimeHours, CostUSD)

CREATE NONCLUSTERED INDEX IX_Fleet_TimeGeo 
ON FactFleetTrips(TimeKey, OriginGeographyKey, DestinationGeographyKey)
INCLUDE (IdleTimeMinutes, TripCostUSD)

-- Update statistics
UPDATE STATISTICS FactWarehouseOperations WITH FULLSCAN
UPDATE STATISTICS FactFleetTrips WITH FULLSCAN
```

### Issue: Gravity Scores Not Updating

**Symptoms**: `sp_RecalculateProductGravity` runs but scores remain static

**Solution**:
```sql
-- Check for missing data in base calculations
SELECT 
    COUNT(*) AS TotalProducts,
    COUNT(CASE WHEN VelocityRank IS NOT NULL THEN 1 
