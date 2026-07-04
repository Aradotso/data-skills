---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing system for logistics, fleet management, and supply chain analytics with multi-fact star schema
triggers:
  - "set up LogiFleet Pulse analytics platform"
  - "create supply chain dashboard with Power BI"
  - "implement logistics data warehouse schema"
  - "build fleet management analytics"
  - "configure warehouse operations reporting"
  - "deploy multi-fact star schema for logistics"
  - "integrate fleet telemetry with warehouse data"
  - "analyze cross-dock operations and KPIs"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing solution for supply chain and logistics intelligence. It provides:

- **Multi-fact star schema** connecting warehouse operations, fleet telemetry, inventory, and external data
- **Cross-modal analytics** linking warehouse velocity with fleet performance
- **Real-time dashboarding** via Power BI with 15-minute refresh cycles
- **Predictive bottleneck detection** and fleet maintenance prioritization
- **Warehouse gravity zone optimization** based on pick frequency and item attributes
- **Temporal elasticity modeling** for scenario planning

The platform unifies fragmented logistics data into a semantic layer for operational intelligence.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- Access to data sources: WMS, TMS, telemetry APIs, ERP systems
- SQL Server Management Studio (SSMS) or Azure Data Studio

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables first
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeStamp DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthOfYear TINYINT NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear SMALLINT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
)

-- Create clustered columnstore index for optimal compression
CREATE CLUSTERED COLUMNSTORE INDEX CCI_DimTime ON DimTime
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL,
    LocationType VARCHAR(20) NOT NULL, -- 'Warehouse', 'DistributionCenter', 'RouteNode'
    LocationName VARCHAR(200) NOT NULL,
    Address VARCHAR(500),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    PostalCode VARCHAR(20),
    Region VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1
)
GO

CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    Brand VARCHAR(100),
    UnitOfMeasure VARCHAR(20),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    UnitCost MONEY,
    UnitPrice MONEY,
    -- Gravity zone attributes
    PickFrequencyScore DECIMAL(5,2), -- Calculated metric
    ValueDensity DECIMAL(10,2), -- Price per cubic meter
    GravityZone VARCHAR(20) -- 'High', 'Medium', 'Low'
)
GO

CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    ContactName VARCHAR(200),
    ContactEmail VARCHAR(200),
    ContactPhone VARCHAR(50),
    Country VARCHAR(100),
    -- Reliability metrics
    AverageLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,2),
    OnTimeDeliveryRate DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    IsActive BIT DEFAULT 1
)
GO

-- Create fact table for warehouse operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID VARCHAR(100) NOT NULL,
    OrderID VARCHAR(100),
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    EmployeeID VARCHAR(50),
    EquipmentID VARCHAR(50),
    StorageZone VARCHAR(50),
    FromLocation VARCHAR(100),
    ToLocation VARCHAR(100),
    BatchNumber VARCHAR(100),
    ExpirationDate DATE,
    QualityCheckPassed BIT,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO

-- Partition by month for better query performance
-- Note: Requires partitioned table setup with partition scheme
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Time 
    ON FactWarehouseOperations(TimeKey) 
    INCLUDE (OperationType, QuantityHandled, DwellTimeMinutes)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50) NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    RouteID VARCHAR(100),
    TripDistance DECIMAL(10,2), -- in kilometers
    TripDurationMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    AverageSpeed DECIMAL(5,2),
    MaxSpeed DECIMAL(5,2),
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    WeightCarriedKg DECIMAL(10,2),
    NumberOfStops INT,
    OnTimeDelivery BIT,
    DelayMinutes INT,
    DelayReason VARCHAR(200),
    FuelCost MONEY,
    TollCost MONEY,
    MaintenanceIncidents INT,
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle 
    ON FactFleetTrips(VehicleID, TimeKey) 
    INCLUDE (TripDistance, FuelConsumedLiters, IdleTimeMinutes)
GO

-- Create bridge table for many-to-many relationships
CREATE TABLE BridgeRouteStorageZone (
    RouteID VARCHAR(100) NOT NULL,
    StorageZone VARCHAR(50) NOT NULL,
    AllocationPercentage DECIMAL(5,2),
    PRIMARY KEY (RouteID, StorageZone)
)
GO
```

### Step 2: Create ETL Stored Procedures

```sql
-- Incremental loading procedure for warehouse operations
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new records from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OperationID, OrderID, QuantityHandled, DwellTimeMinutes,
        ProcessingTimeMinutes, EmployeeID, EquipmentID,
        StorageZone, FromLocation, ToLocation, BatchNumber,
        ExpirationDate, QualityCheckPassed
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        stg.OperationType,
        stg.OperationID,
        stg.OrderID,
        stg.QuantityHandled,
        stg.DwellTimeMinutes,
        stg.ProcessingTimeMinutes,
        stg.EmployeeID,
        stg.EquipmentID,
        stg.StorageZone,
        stg.FromLocation,
        stg.ToLocation,
        stg.BatchNumber,
        stg.ExpirationDate,
        stg.QualityCheckPassed
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime dt 
        ON CAST(stg.OperationDateTime AS DATE) = CAST(dt.DateTimeStamp AS DATE)
        AND DATEPART(HOUR, stg.OperationDateTime) = dt.HourOfDay
        AND DATEPART(MINUTE, stg.OperationDateTime) / 15 = DATEPART(MINUTE, dt.DateTimeStamp) / 15
    INNER JOIN DimGeography dg ON stg.LocationID = dg.LocationID
    INNER JOIN DimProduct dp ON stg.SKU = dp.SKU
    WHERE stg.OperationDateTime > @LastLoadDateTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.OperationID = stg.OperationID
        );
    
    -- Log load completion
    INSERT INTO ETLLog (ProcedureName, RecordsProcessed, LoadDateTime)
    VALUES ('usp_LoadWarehouseOperations', @@ROWCOUNT, GETDATE());
END
GO

-- Calculate gravity zone scores
CREATE PROCEDURE usp_UpdateProductGravityZones
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate pick frequency score (last 90 days)
    WITH PickFrequency AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            SUM(QuantityHandled) AS TotalQuantity
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
            AND TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTimeStamp >= DATEADD(DAY, -90, GETDATE()))
        GROUP BY ProductKey
    ),
    Scores AS (
        SELECT 
            p.ProductKey,
            ISNULL(pf.PickCount, 0) AS PickCount,
            -- Normalize to 0-100 scale
            CAST(ISNULL(pf.PickCount, 0) * 100.0 / NULLIF((SELECT MAX(PickCount) FROM PickFrequency), 0) AS DECIMAL(5,2)) AS PickFrequencyScore,
            -- Value density = price per cubic meter
            CASE 
                WHEN p.UnitVolume > 0 THEN p.UnitPrice / p.UnitVolume
                ELSE 0
            END AS ValueDensity
        FROM DimProduct p
        LEFT JOIN PickFrequency pf ON p.ProductKey = pf.ProductKey
    )
    UPDATE p
    SET 
        p.PickFrequencyScore = s.PickFrequencyScore,
        p.ValueDensity = s.ValueDensity,
        p.GravityZone = CASE
            WHEN s.PickFrequencyScore >= 70 OR s.ValueDensity >= 1000 THEN 'High'
            WHEN s.PickFrequencyScore >= 30 OR s.ValueDensity >= 500 THEN 'Medium'
            ELSE 'Low'
        END
    FROM DimProduct p
    INNER JOIN Scores s ON p.ProductKey = s.ProductKey;
END
GO
```

### Step 3: Create Analytical Views

```sql
-- Cross-fact KPI view: Warehouse dwell time vs. fleet idle time
CREATE VIEW vw_DwellTimeFleetImpact
AS
SELECT 
    dt.DateKey,
    dt.DayName,
    dg.LocationName AS Warehouse,
    dp.Category AS ProductCategory,
    dp.GravityZone,
    AVG(fwo.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(fwo.QuantityHandled) AS TotalUnitsProcessed,
    -- Join to fleet data for correlated routes
    AVG(fft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(fft.DelayMinutes) AS AvgDeliveryDelayMinutes,
    SUM(fft.FuelCost) AS TotalFuelCost
FROM FactWarehouseOperations fwo
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON fwo.GeographyKey = dg.GeographyKey
INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
LEFT JOIN BridgeRouteStorageZone brsz ON fwo.StorageZone = brsz.StorageZone
LEFT JOIN FactFleetTrips fft 
    ON dt.TimeKey = fft.TimeKey
    AND dg.GeographyKey = fft.OriginGeographyKey
WHERE fwo.OperationType IN ('Picking', 'Packing')
GROUP BY 
    dt.DateKey, dt.DayName, dg.LocationName, 
    dp.Category, dp.GravityZone
GO

-- Predictive bottleneck index
CREATE VIEW vw_BottleneckIndex
AS
WITH CurrentMetrics AS (
    SELECT 
        GeographyKey,
        ProductKey,
        AVG(DwellTimeMinutes) AS AvgDwellTime,
        STDEV(DwellTimeMinutes) AS StdDevDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTimeStamp >= DATEADD(DAY, -7, GETDATE()))
    GROUP BY GeographyKey, ProductKey
),
HistoricalBaseline AS (
    SELECT 
        GeographyKey,
        ProductKey,
        AVG(DwellTimeMinutes) AS BaselineDwellTime
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTimeStamp >= DATEADD(DAY, -90, GETDATE()))
        AND TimeKey < (SELECT TimeKey FROM DimTime WHERE DateTimeStamp >= DATEADD(DAY, -7, GETDATE()))
    GROUP BY GeographyKey, ProductKey
)
SELECT 
    dg.LocationName,
    dp.SKU,
    dp.ProductName,
    dp.GravityZone,
    cm.AvgDwellTime,
    hb.BaselineDwellTime,
    -- Bottleneck index: (current - baseline) / baseline * 100
    CAST((cm.AvgDwellTime - hb.BaselineDwellTime) / NULLIF(hb.BaselineDwellTime, 0) * 100 AS DECIMAL(10,2)) AS BottleneckIndexPct,
    cm.StdDevDwellTime,
    cm.OperationCount
FROM CurrentMetrics cm
INNER JOIN HistoricalBaseline hb 
    ON cm.GeographyKey = hb.GeographyKey 
    AND cm.ProductKey = hb.ProductKey
INNER JOIN DimGeography dg ON cm.GeographyKey = dg.GeographyKey
INNER JOIN DimProduct dp ON cm.ProductKey = dp.ProductKey
WHERE cm.AvgDwellTime > hb.BaselineDwellTime * 1.2 -- 20% threshold
GO
```

### Step 4: Configure Power BI Connection

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `YOUR_SQL_SERVER`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** for real-time, or **Import** for scheduled refresh
6. Load the following tables/views:
   - All Dim* tables
   - All Fact* tables
   - vw_DwellTimeFleetImpact
   - vw_BottleneckIndex

```dax
// Create calculated measures in Power BI

// Fleet efficiency ratio
Fleet Efficiency % = 
DIVIDE(
    SUM(FactFleetTrips[TripDistance]),
    SUM(FactFleetTrips[TripDurationMinutes]) * 0.016667, // Convert to hours
    0
)

// Warehouse throughput per hour
Warehouse Throughput = 
DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    DISTINCTCOUNT(FactWarehouseOperations[TimeKey]) * 0.25, // 15-min buckets = 0.25 hrs
    0
)

// Cross-fact KPI: Cost per unit shipped
Cost Per Unit Shipped = 
VAR TotalShipped = 
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        FactWarehouseOperations[OperationType] = "Shipping"
    )
VAR TotalFleetCost = SUM(FactFleetTrips[FuelCost]) + SUM(FactFleetTrips[TollCost])
RETURN DIVIDE(TotalFleetCost, TotalShipped, 0)

// Gravity zone distribution
High Gravity Zone % = 
DIVIDE(
    CALCULATE(
        SUM(FactWarehouseOperations[QuantityHandled]),
        DimProduct[GravityZone] = "High"
    ),
    SUM(FactWarehouseOperations[QuantityHandled]),
    0
)
```

## Key Configuration

### Data Refresh Schedule

Configure SQL Server Agent jobs for incremental ETL:

```sql
-- Create SQL Agent job for hourly refresh
USE msdb
GO

EXEC dbo.sp_add_job
    @job_name = N'LogiFleet_Hourly_ETL',
    @enabled = 1,
    @description = N'Incremental load of warehouse and fleet data'
GO

EXEC dbo.sp_add_jobstep
    @job_name = N'LogiFleet_Hourly_ETL',
    @step_name = N'Load Warehouse Operations',
    @subsystem = N'TSQL',
    @database_name = N'LogiFleetPulse',
    @command = N'
        DECLARE @LastLoad DATETIME2
        SELECT @LastLoad = MAX(LoadDateTime) FROM ETLLog WHERE ProcedureName = ''usp_LoadWarehouseOperations''
        EXEC usp_LoadWarehouseOperations @LastLoad
    '
GO

EXEC dbo.sp_add_schedule
    @schedule_name = N'Every Hour',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 8, -- Hour
    @freq_subday_interval = 1
GO

EXEC dbo.sp_attach_schedule
    @job_name = N'LogiFleet_Hourly_ETL',
    @schedule_name = N'Every Hour'
GO
```

### Row-Level Security

```sql
-- Create security roles for different user types
CREATE ROLE WarehouseManager
GO

CREATE ROLE FleetCoordinator
GO

CREATE ROLE ExecutiveViewer
GO

-- Create security function
CREATE FUNCTION fn_SecurityPredicate(@LocationID VARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessGranted
WHERE 
    @LocationID IN (
        SELECT LocationID 
        FROM UserLocationAccess 
        WHERE UserName = USER_NAME()
    )
    OR IS_MEMBER('ExecutiveViewer') = 1
GO

-- Apply security policy to geography dimension
CREATE SECURITY POLICY GeographySecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(LocationID) ON dbo.DimGeography
WITH (STATE = ON)
GO
```

### Environment Variables for External APIs

Store connection strings securely:

```sql
-- Reference environment variables in connection strings
-- In your ETL process or external table setup:

-- Example: Weather API integration
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = 'https://api.weather.example.com',
    CREDENTIAL = WeatherAPICredential -- Created separately with API key from env
)
GO

-- Store credentials using SQL Server Credential Manager
-- API keys should be stored in environment variables and retrieved via:
-- $env:WEATHER_API_KEY (PowerShell)
-- or managed through Azure Key Vault for production
```

## Common Patterns

### Pattern 1: Cross-Fact Analysis Query

```sql
-- Find products with high warehouse dwell time AND high fleet delay
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityZone,
    AVG(fwo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    AVG(fft.DelayMinutes) AS AvgFleetDelayMin,
    COUNT(DISTINCT fwo.OperationKey) AS WarehouseOps,
    COUNT(DISTINCT fft.TripKey) AS FleetTrips,
    -- Composite friction score
    (AVG(fwo.DwellTimeMinutes) * 0.5 + AVG(fft.DelayMinutes) * 0.5) AS FrictionScore
FROM FactWarehouseOperations fwo
INNER JOIN DimProduct dp ON fwo.ProductKey = dp.ProductKey
INNER JOIN DimTime dt ON fwo.TimeKey = dt.TimeKey
LEFT JOIN FactFleetTrips fft 
    ON dt.DateKey = (SELECT DateKey FROM DimTime WHERE TimeKey = fft.TimeKey)
WHERE fwo.OperationType = 'Shipping'
    AND dt.DateTimeStamp >= DATEADD(DAY, -30, GETDATE())
GROUP BY dp.SKU, dp.ProductName, dp.GravityZone
HAVING AVG(fwo.DwellTimeMinutes) > 240 -- More than 4 hours
    AND AVG(fft.DelayMinutes) > 60 -- More than 1 hour delay
ORDER BY FrictionScore DESC
```

### Pattern 2: Temporal Elasticity Simulation

```sql
-- Simulate impact of 95% warehouse capacity vs. current
WITH CurrentCapacity AS (
    SELECT 
        GeographyKey,
        COUNT(*) AS CurrentOpsPerDay,
        AVG(ProcessingTimeMinutes) AS CurrentAvgProcessingTime
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTimeStamp >= DATEADD(DAY, -7, GETDATE()))
    GROUP BY GeographyKey
),
SimulatedCapacity AS (
    SELECT 
        GeographyKey,
        CurrentOpsPerDay * 1.19 AS ProjectedOpsPerDay, -- 95/80 = 1.1875
        -- Assume 15% processing time increase due to congestion
        CurrentAvgProcessingTime * 1.15 AS ProjectedAvgProcessingTime
    FROM CurrentCapacity
)
SELECT 
    dg.LocationName,
    cc.CurrentOpsPerDay,
    sc.ProjectedOpsPerDay,
    cc.CurrentAvgProcessingTime,
    sc.ProjectedAvgProcessingTime,
    -- Estimated daily hours needed
    (sc.ProjectedOpsPerDay * sc.ProjectedAvgProcessingTime) / 60.0 AS ProjectedDailyHours,
    -- Compare to current
    ((sc.ProjectedOpsPerDay * sc.ProjectedAvgProcessingTime) - 
     (cc.CurrentOpsPerDay * cc.CurrentAvgProcessingTime)) / 60.0 AS AdditionalHoursNeeded
FROM CurrentCapacity cc
INNER JOIN SimulatedCapacity sc ON cc.GeographyKey = sc.GeographyKey
INNER JOIN DimGeography dg ON cc.GeographyKey = dg.GeographyKey
ORDER BY AdditionalHoursNeeded DESC
```

### Pattern 3: Automated Alert Configuration

```sql
-- Create alert stored procedure
CREATE PROCEDURE usp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    DECLARE @AlertSubject NVARCHAR(200);
    
    -- Check 1: Fleet idle time exceeds 15% of trip duration
    IF EXISTS (
        SELECT 1 FROM FactFleetTrips
        WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE DateTimeStamp >= DATEADD(HOUR, -1, GETDATE()))
            AND CAST(IdleTimeMinutes AS FLOAT) / NULLIF(TripDurationMinutes, 0) > 0.15
    )
    BEGIN
        SET @AlertSubject = 'LogiFleet Alert: Excessive Fleet Idle Time';
        SET @AlertMessage = 'Fleet vehicles have exceeded 15% idle time threshold in the last hour.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'fleet.manager@company.com', -- Use @recipients from env
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END
    
    -- Check 2: Bottleneck index exceeds 50% for high-gravity products
    IF EXISTS (
        SELECT 1 FROM vw_BottleneckIndex
        WHERE GravityZone = 'High'
            AND BottleneckIndexPct > 50
    )
    BEGIN
        SET @AlertSubject = 'LogiFleet Alert: High-Gravity Product Bottleneck';
        SET @AlertMessage = 'High-value products are experiencing significant dwell time increases.';
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = 'warehouse.manager@company.com',
            @subject = @AlertSubject,
            @body = @AlertMessage;
    END
END
GO

-- Schedule the alert check every 15 minutes via SQL Agent
```

## Real Code Examples

### Example 1: Data Loading from External API

```sql
-- Python script for loading fleet telemetry from API
-- Save as load_fleet_telemetry.py

import pyodbc
import requests
import os
from datetime import datetime, timedelta

# Connection string from environment
conn_str = os.getenv('LOGIFLEET_SQL_CONNECTION')
api_key = os.getenv('FLEET_TELEMETRY_API_KEY')
api_url = os.getenv('FLEET_TELEMETRY_API_URL')

conn = pyodbc.connect(conn_str)
cursor = conn.cursor()

# Fetch last load timestamp
cursor.execute("SELECT MAX(LoadDateTime) FROM ETLLog WHERE ProcedureName = 'LoadFleetTelemetry'")
last_load = cursor.fetchone()[0] or datetime.now() - timedelta(hours=24)

# Call external API
headers = {'Authorization': f'Bearer {api_key}'}
params = {'since': last_load.isoformat()}
response = requests.get(f'{api_url}/trips', headers=headers, params=params)
trips = response.json()

# Insert into staging table
for trip in trips:
    cursor.execute("""
        INSERT INTO StagingFleetTrips (
            VehicleID, DriverID, TripStartDateTime, TripEndDateTime,
            OriginLocationID, DestinationLocationID, RouteID,
            TripDistance, FuelConsumed, IdleTimeMinutes, AverageSpeed,
            MaxSpeed, LoadingTimeMinutes, UnloadingTimeMinutes,
            WeightCarried, NumberOfStops, DelayMinutes, DelayReason
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        trip['vehicle_id'], trip['driver_id'], 
        trip['start_time'], trip['end_time'],
        trip['origin'], trip['destination'], trip['route_id'],
        trip['distance_km'], trip['fuel_liters'], trip['idle_minutes'],
        trip['avg_speed'], trip['max_speed'],
        trip['loading_minutes'], trip['unloading_minutes'],
        trip['weight_kg'], trip['stops'], trip['delay_minutes'],
        trip.get('delay_reason')
    ))

conn.commit()

# Call SQL procedure to process staging data
cursor.execute("EXEC usp_LoadFleetTrips ?", last_load)
conn.commit()

cursor.close()
conn.close()
```

### Example 2: Power BI Custom Visual Configuration

```javascript
// Custom JavaScript for embedded Power BI dashboard
// Include in HTML report page

const config = {
    type: 'report',
    tokenType: models.TokenType.Embed,
    accessToken: window.POWERBI_EMBED_TOKEN, // From server-side auth
    embedUrl: window.POWERBI_EMBED_URL,
    id: window.POWERBI_REPORT_ID,
    permissions: models.Permissions.Read,
    settings: {
        filterPaneEnabled: false,
        navContentPaneEnabled: true,
        background: models.BackgroundType.Transparent
    }
};

// Embed report
const reportContainer = document.getElementById('reportContainer');
const report = powerbi.embed(reportContainer, config);

// Event handling for cross-filtering
report.on('dataSelected', function(event) {
    const selectedData = event.detail;
    console.log('User selected:', selectedData);
    
    // Trigger custom action, e.g., load drill-down data
    if (selectedData.dataPoints.length > 0) {
        const locationName = selectedData.dataPoints[0].identity[0].equals;
        loadDrillDownDetails(locationName);
    }
});

function loadDrillDownDetails(location) {
    // Fetch additional data from SQL via API
    fetch(`/api/warehouse-details?location=${encodeURIComponent(location)}`)
        .then(response => response.json())
        .then(data =>
