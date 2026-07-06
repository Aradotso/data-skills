---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-modal logistics intelligence platform for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse analytics platform
  - configure supply chain data warehouse with logifleet
  - deploy logifleet sql schema and power bi dashboards
  - integrate warehouse and fleet data into logifleet
  - create logistics kpi dashboards with logifleet pulse
  - troubleshoot logifleet pulse data model
  - optimize logifleet warehouse gravity zones
  - implement cross-fact supply chain analytics
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill provides expertise in deploying and using LogiFleet Pulse, a multi-modal logistics intelligence engine that unifies warehouse operations, fleet telemetry, inventory management, and supply chain analytics using MS SQL Server and Power BI.

## What LogiFleet Pulse Does

LogiFleet Pulse is a data warehousing and analytics platform that:

- **Integrates multi-source logistics data** from WMS, fleet telematics, supplier portals, and external APIs
- **Implements multi-fact star schema** with time-phased dimensions for cross-modal KPI analysis
- **Provides Power BI dashboards** for real-time supply chain visibility
- **Enables predictive analytics** for bottleneck detection and fleet optimization
- **Supports role-based access** with row-level security for enterprise deployment

The platform connects warehouse velocity, fleet performance, and demand patterns into a unified semantic layer for decision intelligence.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to source systems (WMS, TMS, ERP) via SQL, REST API, or flat files

### Step 1: Deploy SQL Schema

Clone the repository and deploy the database schema:

```sql
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Deploy dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateValue DATE NOT NULL,
    HourOfDay TINYINT,
    MinuteOfHour TINYINT,
    DayOfWeek VARCHAR(20),
    WeekOfYear TINYINT,
    MonthName VARCHAR(20),
    Quarter TINYINT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT
)

CREATE CLUSTERED COLUMNSTORE INDEX IX_DimTime ON DimTime

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- Warehouse, DC, RouteNode
    Street VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    PostalCode VARCHAR(20),
    Region VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    EffectiveDate DATE,
    ExpirationDate DATE,
    IsCurrent BIT
)

CREATE INDEX IX_Geography_LocationID ON DimGeography(LocationID)
CREATE INDEX IX_Geography_Type ON DimGeography(LocationType)

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) UNIQUE NOT NULL,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    VolumeM3 DECIMAL(10,4),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    UnitValue DECIMAL(12,2),
    VelocityScore INT, -- 1-100
    FragilityScore INT, -- 1-100
    ValueScore INT, -- 1-100
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    RecommendedZone VARCHAR(20), -- HighGravity, MediumGravity, LowGravity
    EffectiveDate DATE,
    ExpirationDate DATE,
    IsCurrent BIT
)

CREATE INDEX IX_Product_SKU ON DimProductGravity(SKU)
CREATE INDEX IX_Product_Gravity ON DimProductGravity(GravityScore DESC)

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200),
    Country VARCHAR(100),
    AverageLeadTimeDays INT,
    LeadTimeVariance DECIMAL(8,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore INT, -- 1-100
    ReliabilityScore AS (OnTimeDeliveryPercent * 0.4 + (100 - DefectRatePercent) * 0.3 + ComplianceScore * 0.3) PERSISTED,
    EffectiveDate DATE,
    ExpirationDate DATE,
    IsCurrent BIT
)

-- Deploy fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50), -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100),
    OrderID VARCHAR(100),
    BatchID VARCHAR(100),
    QuantityUnits INT,
    DwellTimeMinutes INT,
    CycleTimeSeconds INT,
    ZoneAssigned VARCHAR(50),
    EmployeeID VARCHAR(50),
    EquipmentID VARCHAR(50),
    ErrorCount INT DEFAULT 0,
    ReworkRequired BIT DEFAULT 0,
    CONSTRAINT FK_Warehouse_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Warehouse_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Warehouse_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
)

CREATE COLUMNSTORE INDEX IX_FactWarehouse ON FactWarehouseOperations

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    TripID VARCHAR(100),
    RouteID VARCHAR(100),
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelLiters DECIMAL(10,2),
    LoadWeightKG DECIMAL(10,2),
    AverageSpeedKPH DECIMAL(6,2),
    DelayMinutes INT DEFAULT 0,
    DelayReason VARCHAR(200),
    WeatherCondition VARCHAR(100),
    MaintenanceAlertsCount INT DEFAULT 0,
    CONSTRAINT FK_Fleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_Fleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_Fleet_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)

CREATE COLUMNSTORE INDEX IX_FactFleet ON FactFleetTrips

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT NOT NULL,
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    QuantityUnits INT,
    DockDwellMinutes INT,
    TransferEfficiency AS (CASE WHEN DockDwellMinutes > 0 THEN (QuantityUnits * 60.0 / DockDwellMinutes) ELSE 0 END) PERSISTED,
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_CrossDock_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
)

CREATE COLUMNSTORE INDEX IX_FactCrossDock ON FactCrossDock
```

### Step 2: Configure Data Integration

Create stored procedures for incremental data loading:

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Assumes external staging table from WMS
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationID, OrderID, QuantityUnits, DwellTimeMinutes,
        CycleTimeSeconds, ZoneAssigned, EmployeeID
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        stg.OperationType,
        stg.OperationID,
        stg.OrderID,
        stg.QuantityUnits,
        DATEDIFF(MINUTE, stg.StartTime, stg.EndTime) AS DwellTimeMinutes,
        DATEDIFF(SECOND, stg.StartTime, stg.EndTime) AS CycleTimeSeconds,
        stg.ZoneAssigned,
        stg.EmployeeID
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime dt ON CAST(stg.StartTime AS DATE) = dt.DateValue 
        AND DATEPART(HOUR, stg.StartTime) = dt.HourOfDay
        AND (DATEPART(MINUTE, stg.StartTime) / 15) * 15 = dt.MinuteOfHour
    INNER JOIN DimGeography dg ON stg.LocationID = dg.LocationID AND dg.IsCurrent = 1
    INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU AND dp.IsCurrent = 1
    WHERE stg.LoadedTimestamp > @LastLoadTime
    
    -- Update load watermark
    UPDATE LoadControl 
    SET LastLoadTime = GETDATE()
    WHERE TableName = 'FactWarehouseOperations'
END
GO

-- Stored procedure for alert generation
CREATE PROCEDURE usp_GenerateAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Alert: Fleet idling time exceeds 15% of trip duration
    INSERT INTO AlertLog (AlertType, Severity, EntityType, EntityID, Message, DetectedTime)
    SELECT 
        'FleetIdlingExcessive',
        'High',
        'Trip',
        TripID,
        CONCAT('Vehicle ', VehicleID, ' exceeded 15% idle time: ', 
               CAST((IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0)) AS DECIMAL(5,2)), '%'),
        GETDATE()
    FROM FactFleetTrips
    WHERE TimeKey >= (SELECT MAX(TimeKey) FROM DimTime WHERE DateValue = CAST(GETDATE() AS DATE))
        AND (IdleTimeMinutes * 100.0 / NULLIF(DurationMinutes, 0)) > 15
        AND TripID NOT IN (SELECT EntityID FROM AlertLog WHERE AlertType = 'FleetIdlingExcessive' AND DetectedTime > DATEADD(HOUR, -4, GETDATE()))
    
    -- Alert: Warehouse dwell time exceeds 72 hours
    INSERT INTO AlertLog (AlertType, Severity, EntityType, EntityID, Message, DetectedTime)
    SELECT 
        'WarehouseDwellExcessive',
        'Medium',
        'Operation',
        OperationID,
        CONCAT('SKU ', dp.SKU, ' dwell time: ', DwellTimeMinutes / 60, ' hours in zone ', ZoneAssigned),
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    WHERE DwellTimeMinutes > 4320 -- 72 hours
        AND OperationType = 'Putaway'
        AND TimeKey >= (SELECT MAX(TimeKey) FROM DimTime WHERE DateValue > DATEADD(DAY, -1, GETDATE()))
END
GO
```

### Step 3: Load Power BI Template

The repository includes `LogiFleet_Pulse_Master.pbit`. To use it:

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Enter connection parameters:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server (use `${SQL_USERNAME}` and `${SQL_PASSWORD}` env vars)
4. The template auto-detects relationships and builds visuals

### Step 4: Configure Refresh Schedule

For automated dashboard updates, publish to Power BI Service and configure refresh:

```powershell
# PowerShell script to publish and configure refresh
Install-Module -Name MicrosoftPowerBIMgmt

Connect-PowerBIServiceAccount

# Publish report
New-PowerBIReport -Path "LogiFleet_Pulse_Master.pbix" -WorkspaceId $env:POWERBI_WORKSPACE_ID

# Configure gateway and scheduled refresh
$datasetId = (Get-PowerBIDataset -WorkspaceId $env:POWERBI_WORKSPACE_ID | Where-Object {$_.Name -eq "LogiFleet_Pulse_Master"}).Id

$scheduleJson = @{
    value = @{
        days = @("Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday")
        times = @("06:00", "12:00", "18:00")
        enabled = $true
        localTimeZoneId = "UTC"
    }
} | ConvertTo-Json

Invoke-PowerBIRestMethod -Url "datasets/$datasetId/refreshSchedule" -Method Patch -Body $scheduleJson
```

## Key Commands & API

### SQL Views for Common Queries

```sql
-- View: Unified KPI across warehouse and fleet
CREATE VIEW vw_UnifiedLogisticsKPI AS
SELECT 
    dt.DateValue,
    dt.MonthName,
    dg.LocationName,
    dg.LocationType,
    -- Warehouse KPIs
    SUM(CASE WHEN wo.OperationType = 'Picking' THEN wo.QuantityUnits ELSE 0 END) AS TotalPickedUnits,
    AVG(CASE WHEN wo.OperationType = 'Picking' THEN wo.CycleTimeSeconds ELSE NULL END) AS AvgPickTimeSeconds,
    SUM(CASE WHEN wo.DwellTimeMinutes > 4320 THEN 1 ELSE 0 END) AS ExcessiveDwellCount,
    -- Fleet KPIs
    COUNT(DISTINCT ft.TripID) AS TotalTrips,
    AVG(ft.FuelLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelEfficiency,
    SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
    AVG(ft.DelayMinutes) AS AvgDelayMinutes
FROM DimTime dt
LEFT JOIN FactWarehouseOperations wo ON dt.TimeKey = wo.TimeKey
LEFT JOIN FactFleetTrips ft ON dt.TimeKey = ft.TimeKey
LEFT JOIN DimGeography dg ON wo.GeographyKey = dg.GeographyKey OR ft.OriginGeographyKey = dg.GeographyKey
GROUP BY dt.DateValue, dt.MonthName, dg.LocationName, dg.LocationType
GO

-- View: Product gravity zone recommendations
CREATE VIEW vw_ProductGravityRecommendations AS
SELECT 
    SKU,
    ProductName,
    GravityScore,
    RecommendedZone,
    CurrentZone = (
        SELECT TOP 1 ZoneAssigned 
        FROM FactWarehouseOperations wo
        WHERE wo.ProductKey = dp.ProductKey
        ORDER BY TimeKey DESC
    ),
    CASE 
        WHEN RecommendedZone != (SELECT TOP 1 ZoneAssigned FROM FactWarehouseOperations wo WHERE wo.ProductKey = dp.ProductKey ORDER BY TimeKey DESC)
        THEN 'Rearrangement Suggested'
        ELSE 'Optimal'
    END AS ZoneStatus
FROM DimProductGravity dp
WHERE IsCurrent = 1
GO
```

### DAX Measures for Power BI

Create these measures in Power BI:

```dax
// Total Warehouse Throughput
TotalThroughput = 
CALCULATE(
    SUM(FactWarehouseOperations[QuantityUnits]),
    FactWarehouseOperations[OperationType] IN {"Picking", "Shipping"}
)

// Fleet Utilization Percentage
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Cross-Fact KPI: Dwell Time vs Fleet Delay Correlation
DwellDelayCorrelation = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgDelay = AVERAGE(FactFleetTrips[DelayMinutes])
RETURN
    IF(AvgDwell > 4320 && AvgDelay > 30, "High Risk", "Normal")

// Warehouse Gravity Zone Efficiency
GravityZoneEfficiency = 
CALCULATE(
    AVERAGEX(
        FactWarehouseOperations,
        FactWarehouseOperations[QuantityUnits] / NULLIF(FactWarehouseOperations[CycleTimeSeconds], 0) * 3600
    ),
    DimProductGravity[RecommendedZone] = DimProductGravity[ZoneAssigned]
)

// Predictive Bottleneck Index
BottleneckIndex = 
VAR ExcessiveDwell = COUNTROWS(FILTER(FactWarehouseOperations, FactWarehouseOperations[DwellTimeMinutes] > 4320))
VAR HighIdle = COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[IdleTimeMinutes] > FactFleetTrips[DurationMinutes] * 0.15))
VAR MaintenanceAlerts = SUM(FactFleetTrips[MaintenanceAlertsCount])
RETURN
    (ExcessiveDwell * 0.4 + HighIdle * 0.3 + MaintenanceAlerts * 0.3) / COUNTROWS(FactWarehouseOperations)
```

## Configuration

### Row-Level Security (RLS)

Implement role-based access in Power BI:

```dax
// RLS Role: Warehouse Manager
[DimGeography].[LocationType] = "Warehouse" && 
[DimGeography].[Region] = USERPRINCIPALNAME()

// RLS Role: Fleet Supervisor
[FactFleetTrips].[VehicleID] IN 
    (SELECT VehicleID FROM UserVehicleAssignments WHERE UserEmail = USERPRINCIPALNAME())

// RLS Role: Executive (all access)
1=1
```

Apply roles in Power BI Desktop:
1. Modeling tab → Manage Roles
2. Create roles with DAX filters above
3. Test with "View as Role"

### External Data Source Integration

Configure polybase for external weather API:

```sql
-- Create external data source for weather API
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = N'https://api.weather.example.com/v1/',
    CREDENTIAL = WeatherAPICredential
)

CREATE EXTERNAL TABLE ExternalWeather (
    LocationID VARCHAR(50),
    Timestamp DATETIME,
    Condition VARCHAR(100),
    Temperature DECIMAL(5,2),
    Precipitation DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = WeatherAPI,
    LOCATION = N'current',
    FORMAT = 'JSON'
)
```

### Alert Notification Setup

Configure SQL Server Database Mail for automated alerts:

```sql
-- Configure Database Mail profile
EXEC msdb.dbo.sysmail_add_profile_sp
    @profile_name = 'LogiFleetAlerts',
    @description = 'Profile for LogiFleet Pulse alerts'

EXEC msdb.dbo.sysmail_add_account_sp
    @account_name = 'LogiFleetSMTP',
    @email_address = 'alerts@company.com',
    @mailserver_name = 'smtp.company.com',
    @username = '${SMTP_USERNAME}',
    @password = '${SMTP_PASSWORD}'

-- Create alert job
EXEC msdb.dbo.sp_add_job
    @job_name = 'LogiFleet_HourlyAlertCheck'

EXEC msdb.dbo.sp_add_jobstep
    @job_name = 'LogiFleet_HourlyAlertCheck',
    @step_name = 'Generate and Send Alerts',
    @subsystem = 'TSQL',
    @command = N'
        EXEC usp_GenerateAlerts;
        
        DECLARE @AlertMessage NVARCHAR(MAX)
        SELECT @AlertMessage = STRING_AGG(Message, CHAR(13) + CHAR(10))
        FROM AlertLog
        WHERE DetectedTime > DATEADD(HOUR, -1, GETDATE())
        
        IF @AlertMessage IS NOT NULL
        BEGIN
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = ''LogiFleetAlerts'',
                @recipients = ''operations@company.com'',
                @subject = ''LogiFleet Pulse Hourly Alerts'',
                @body = @AlertMessage
        END
    ',
    @database_name = 'LogiFleetPulse'

EXEC msdb.dbo.sp_add_schedule
    @schedule_name = 'Hourly',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hours
    @freq_subday_interval = 1

EXEC msdb.dbo.sp_attach_schedule
    @job_name = 'LogiFleet_HourlyAlertCheck',
    @schedule_name = 'Hourly'

EXEC msdb.dbo.sp_add_jobserver
    @job_name = 'LogiFleet_HourlyAlertCheck'
```

## Common Patterns

### Pattern 1: Cross-Modal KPI Dashboard

Build a Power BI page that correlates warehouse and fleet metrics:

```dax
// Measure: Combined efficiency score
CombinedEfficiencyScore = 
VAR WarehouseEff = [GravityZoneEfficiency]
VAR FleetEff = [FleetUtilization]
VAR NormWarehouse = DIVIDE(WarehouseEff, MAXX(ALL(FactWarehouseOperations), [GravityZoneEfficiency]), 0)
VAR NormFleet = DIVIDE(FleetEff, 100, 0)
RETURN
    (NormWarehouse * 0.6 + NormFleet * 0.4) * 100

// Measure: Correlation visual tooltip
CorrelationTooltip = 
"Warehouse gravity efficiency: " & FORMAT([GravityZoneEfficiency], "#,##0.00") & " units/hour" & UNICHAR(10) &
"Fleet utilization: " & FORMAT([FleetUtilization], "#0.0") & "%" & UNICHAR(10) &
"Combined score: " & FORMAT([CombinedEfficiencyScore], "#0.0")
```

Visual setup:
- Matrix visual with DimTime[MonthName] on rows
- [CombinedEfficiencyScore] as values
- Conditional formatting: Green (>80), Yellow (60-80), Red (<60)
- Tooltip with [CorrelationTooltip]

### Pattern 2: Warehouse Gravity Zone Optimization

Query to identify rearrangement opportunities:

```sql
-- Products in wrong gravity zone
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    dp.RecommendedZone,
    CurrentZone = (SELECT TOP 1 wo.ZoneAssigned FROM FactWarehouseOperations wo WHERE wo.ProductKey = dp.ProductKey ORDER BY TimeKey DESC),
    AvgPickTime = (SELECT AVG(wo.CycleTimeSeconds) FROM FactWarehouseOperations wo WHERE wo.ProductKey = dp.ProductKey AND wo.OperationType = 'Picking'),
    ProjectedTimeSavings = (
        (SELECT AVG(wo.CycleTimeSeconds) FROM FactWarehouseOperations wo WHERE wo.ProductKey = dp.ProductKey AND wo.OperationType = 'Picking') -
        (SELECT AVG(wo.CycleTimeSeconds) FROM FactWarehouseOperations wo 
         INNER JOIN DimProductGravity dp2 ON wo.ProductKey = dp2.ProductKey
         WHERE wo.ZoneAssigned = dp.RecommendedZone AND wo.OperationType = 'Picking')
    ) / 60.0 AS MinutesSavedPerPick
FROM DimProductGravity dp
WHERE dp.IsCurrent = 1
    AND dp.RecommendedZone != (SELECT TOP 1 wo.ZoneAssigned FROM FactWarehouseOperations wo WHERE wo.ProductKey = dp.ProductKey ORDER BY TimeKey DESC)
ORDER BY GravityScore DESC
```

Implement rearrangement workflow:

```sql
-- Create work order for zone rearrangement
CREATE TABLE ZoneRearrangementWorkOrders (
    WorkOrderID INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50),
    CurrentZone VARCHAR(50),
    TargetZone VARCHAR(50),
    Priority INT,
    Status VARCHAR(50) DEFAULT 'Pending',
    CreatedDate DATETIME DEFAULT GETDATE(),
    CompletedDate DATETIME NULL
)

INSERT INTO ZoneRearrangementWorkOrders (SKU, CurrentZone, TargetZone, Priority)
SELECT TOP 20
    dp.SKU,
    (SELECT TOP 1 wo.ZoneAssigned FROM FactWarehouseOperations wo WHERE wo.ProductKey = dp.ProductKey ORDER BY TimeKey DESC),
    dp.RecommendedZone,
    CAST(dp.GravityScore AS INT) -- Higher gravity = higher priority
FROM DimProductGravity dp
WHERE dp.IsCurrent = 1
    AND dp.RecommendedZone != (SELECT TOP 1 wo.ZoneAssigned FROM FactWarehouseOperations wo WHERE wo.ProductKey = dp.ProductKey ORDER BY TimeKey DESC)
ORDER BY dp.GravityScore DESC
```

### Pattern 3: Fleet Predictive Maintenance Triage

Rank vehicles by maintenance urgency weighted by cargo value:

```sql
-- Maintenance triage with revenue impact
WITH VehicleMaintenanceStatus AS (
    SELECT 
        ft.VehicleID,
        SUM(ft.MaintenanceAlertsCount) AS TotalAlerts,
        AVG(ft.FuelLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelConsumption,
        AVG(CAST(ft.IdleTimeMinutes AS FLOAT) / NULLIF(ft.DurationMinutes, 0)) AS AvgIdleRatio,
        MAX(ft.TimeKey) AS LastTripTimeKey
    FROM FactFleetTrips ft
    WHERE ft.TimeKey >= (SELECT MIN(TimeKey) FROM DimTime WHERE DateValue > DATEADD(DAY, -30, GETDATE()))
    GROUP BY ft.VehicleID
),
VehicleCargoValue AS (
    SELECT 
        ft.VehicleID,
        AVG(dp.UnitValue * wo.QuantityUnits) AS AvgCargoValue
    FROM FactFleetTrips ft
    INNER JOIN FactWarehouseOperations wo ON wo.OrderID = ft.TripID
    INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    GROUP BY ft.VehicleID
)
SELECT 
    vms.VehicleID,
    vms.TotalAlerts,
    vms.AvgFuelConsumption,
    vms.AvgIdleRatio,
    vcv.AvgCargoValue,
    TriageScore = (
        vms.TotalAlerts * 0.3 +
        (vms.AvgFuelConsumption / 10.0) * 0.2 +
        (vms.AvgIdleRatio * 100) * 0.2 +
        (vcv.AvgCargoValue / 10000.0) * 0.3
    ),
    Priority = CASE 
        WHEN (vms.TotalAlerts * 0.3 + (vms.AvgFuelConsumption / 10.0) * 0.2 + (vms.AvgIdleRatio * 100) * 0.2 + (vcv.AvgCargoValue / 10000.0) * 0.3) > 10 THEN 'Critical'
        WHEN (vms.TotalAlerts * 0.3 + (vms.AvgFuelConsumption / 10.0) * 0.2 + (vms.AvgIdleRatio * 100) * 0.2 + (vcv.AvgCargoValue / 10000.0) * 0.3) > 5 THEN 'High'
        ELSE 'Normal'
    END
FROM VehicleMaintenanceStatus vms
INNER JOIN VehicleCargoValue vcv ON vms.VehicleID = vcv.VehicleID
ORDER BY TriageScore DESC
```

### Pattern 4: Temporal Elasticity Simulation

Simulate impact of capacity changes:

```sql
-- Scenario: Increase warehouse capacity from 80% to 95%
WITH CurrentUtilization AS (
    SELECT 
        GeographyKey,
        SUM(QuantityUnits) AS CurrentInventory,
        4000 AS MaxCapacity, -- Example capacity
        CAST(SUM(QuantityUnits) AS FLOAT) / 4000 AS UtilizationPercent
    FROM FactWarehouseOperations
    WHERE OperationType = 'Putaway'
