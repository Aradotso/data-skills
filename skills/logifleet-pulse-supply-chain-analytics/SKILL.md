---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for real-time logistics, fleet optimization, and cross-modal supply chain analytics
triggers:
  - "set up logistics analytics dashboard"
  - "implement supply chain data warehouse"
  - "create fleet tracking Power BI reports"
  - "build warehouse operations KPI dashboard"
  - "configure multi-fact star schema for logistics"
  - "integrate fleet telemetry with warehouse data"
  - "deploy LogiFleet Pulse analytics"
  - "analyze cross-dock operations with Power BI"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to deploy and configure LogiFleet Pulse, an advanced MS SQL Server & Power BI-based logistics intelligence platform. It provides multi-fact star schema data modeling, real-time fleet optimization, warehouse gravity zone analytics, and cross-modal supply chain KPI harmonization.

## What LogiFleet Pulse Does

LogiFleet Pulse is a **multi-modal logistics intelligence engine** that:
- Fuses warehouse operations, fleet telemetry, inventory aging, and external signals into a unified semantic layer
- Uses custom multi-fact star schema with time-phased dimensions for cross-fact KPI analysis
- Provides real-time Power BI dashboards with 15-minute refresh intervals
- Implements predictive bottleneck detection and adaptive fleet triage
- Features Warehouse Gravity Zones™ for spatial optimization based on pick frequency, item value, and fragility
- Enables temporal elasticity modeling for supply chain scenario simulation

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (recommended 2022)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS)
- Network access to WMS, TMS, or telemetry data sources

### Step 1: Deploy SQL Schema

```sql
-- Create the database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTime DATETIME2(0) NOT NULL,
    Date DATE NOT NULL,
    TimeOf15Min VARCHAR(5),
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    WeekOfYear TINYINT,
    MonthNumber TINYINT,
    MonthName VARCHAR(10),
    Quarter TINYINT,
    Year SMALLINT,
    FiscalPeriod VARCHAR(7),
    IsWeekend BIT,
    IsHoliday BIT
);

-- Create non-clustered index for performance
CREATE NONCLUSTERED INDEX IX_DimTime_DateTime 
ON DimTime(DateTime);

CREATE NONCLUSTERED INDEX IX_DimTime_Date 
ON DimTime(Date);

-- Geography dimension with hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(100) NOT NULL,
    LocationType VARCHAR(20), -- 'Warehouse', 'RouteNode', 'CrossDock'
    Address VARCHAR(200),
    City VARCHAR(50),
    Region VARCHAR(50),
    Country VARCHAR(50),
    Continent VARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    ValidFrom DATE,
    ValidTo DATE
);

-- Product hierarchy with gravity scoring
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(50) NOT NULL UNIQUE,
    ProductName VARCHAR(200),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    ShelfLifeDays INT,
    -- Gravity calculation inputs
    AvgDailyPickFrequency DECIMAL(10,2),
    UnitValue DECIMAL(10,2),
    FragilityScore TINYINT, -- 0-10
    GravityScore AS (
        (AvgDailyPickFrequency * 0.4) + 
        (UnitValue / 100 * 0.35) + 
        (FragilityScore * 0.25)
    ) PERSISTED,
    GravityZone AS (
        CASE 
            WHEN (AvgDailyPickFrequency * 0.4 + UnitValue / 100 * 0.35 + FragilityScore * 0.25) > 7 THEN 'High'
            WHEN (AvgDailyPickFrequency * 0.4 + UnitValue / 100 * 0.35 + FragilityScore * 0.25) > 4 THEN 'Medium'
            ELSE 'Low'
        END
    ) PERSISTED
);

-- Supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200),
    Country VARCHAR(50),
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVarianceDays DECIMAL(5,1),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore TINYINT, -- 0-100
    ReliabilityGrade AS (
        CASE 
            WHEN OnTimeDeliveryPercent >= 95 AND DefectRatePercent < 1 THEN 'A'
            WHEN OnTimeDeliveryPercent >= 90 AND DefectRatePercent < 2 THEN 'B'
            WHEN OnTimeDeliveryPercent >= 80 AND DefectRatePercent < 5 THEN 'C'
            ELSE 'D'
        END
    ) PERSISTED
);

-- Fleet dimension
CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID VARCHAR(50) NOT NULL UNIQUE,
    VehicleType VARCHAR(50),
    Make VARCHAR(50),
    Model VARCHAR(50),
    Year SMALLINT,
    LicensePlate VARCHAR(20),
    Capacity DECIMAL(10,2),
    FuelType VARCHAR(20),
    HasRefrigeration BIT,
    HasLiftGate BIT,
    IsActive BIT DEFAULT 1,
    LastMaintenanceDate DATE,
    NextMaintenanceDate DATE
);

-- Fact table: Warehouse Operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType VARCHAR(20), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID VARCHAR(50),
    Quantity INT,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    StorageZone VARCHAR(20),
    PickPathLength DECIMAL(10,2), -- meters
    ErrorsCount INT DEFAULT 0,
    ReworkRequired BIT DEFAULT 0,
    CONSTRAINT FK_FactWH_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactWH_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactWH_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Create columnstore index for analytics performance
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOperations
ON FactWarehouseOperations (TimeKey, GeographyKey, ProductKey, Quantity, DurationMinutes, DwellTimeHours);

-- Fact table: Fleet Trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    FleetKey INT NOT NULL,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    TripID VARCHAR(50),
    DistanceKm DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeight DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    AvgSpeedKmH DECIMAL(5,2),
    HarshBrakingEvents INT DEFAULT 0,
    WeatherCondition VARCHAR(20),
    TrafficDelayMinutes DECIMAL(10,2) DEFAULT 0,
    OnTimeDelivery BIT,
    CONSTRAINT FK_FactFleet_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactFleet_Fleet FOREIGN KEY (FleetKey) REFERENCES DimFleet(FleetKey),
    CONSTRAINT FK_FactFleet_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactFleet_Dest FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactFleetTrips
ON FactFleetTrips (TimeKey, FleetKey, OriginGeographyKey, DestinationGeographyKey, DistanceKm, FuelConsumedLiters);

-- Fact table: Cross Dock Operations
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    InboundTripID VARCHAR(50),
    OutboundTripID VARCHAR(50),
    Quantity INT,
    DockDwellMinutes DECIMAL(10,2),
    TransferTimeMinutes DECIMAL(10,2),
    CONSTRAINT FK_FactCD_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FactCD_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FactCD_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate DimTime with 15-minute granularity
CREATE PROCEDURE PopulateDimTime
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2(0) = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @TimeKey = CONVERT(INT, FORMAT(@CurrentDateTime, 'yyyyMMddHHmm'));
        
        INSERT INTO DimTime (
            TimeKey, DateTime, Date, TimeOf15Min, HourOfDay, DayOfWeek, 
            DayName, WeekOfYear, MonthNumber, MonthName, Quarter, Year, 
            FiscalPeriod, IsWeekend, IsHoliday
        )
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            FORMAT(@CurrentDateTime, 'HH:mm'),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            DATEPART(YEAR, @CurrentDateTime),
            CONCAT('FY', DATEPART(YEAR, @CurrentDateTime), '-', 
                   RIGHT('0' + CAST(DATEPART(MONTH, @CurrentDateTime) AS VARCHAR), 2)),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1,7) THEN 1 ELSE 0 END,
            0 -- Update holiday logic as needed
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate 2 years of data
EXEC PopulateDimTime '2025-01-01', '2026-12-31';
```

### Step 3: Create ETL Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE LoadWarehouseOperations
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Example ETL from staging table (replace with your WMS source)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        OperationID, Quantity, DurationMinutes, DwellTimeHours, 
        StorageZone, PickPathLength
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.OperationType,
        s.OperationID,
        s.Quantity,
        DATEDIFF(MINUTE, s.StartTime, s.EndTime) AS DurationMinutes,
        CASE WHEN s.OperationType = 'Putaway' 
             THEN DATEDIFF(HOUR, s.StartTime, GETDATE()) 
             ELSE 0 END AS DwellTimeHours,
        s.StorageZone,
        s.PickPathLength
    FROM StagingWarehouseOps s
    INNER JOIN DimTime t ON CAST(s.StartTime AS DATETIME2(0)) = t.DateTime
    INNER JOIN DimGeography g ON s.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON s.SKU = p.SKU
    WHERE NOT EXISTS (
        SELECT 1 FROM FactWarehouseOperations f 
        WHERE f.OperationID = s.OperationID
    );
END;
GO

-- Incremental load for fleet trips
CREATE PROCEDURE LoadFleetTrips
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TimeKey, FleetKey, OriginGeographyKey, DestinationGeographyKey,
        TripID, DistanceKm, DurationMinutes, IdleTimeMinutes, 
        FuelConsumedLiters, LoadWeight, OnTimeDelivery
    )
    SELECT 
        t.TimeKey,
        f.FleetKey,
        g_origin.GeographyKey,
        g_dest.GeographyKey,
        s.TripID,
        s.DistanceKm,
        DATEDIFF(MINUTE, s.DepartureTime, s.ArrivalTime),
        s.IdleTimeMinutes,
        s.FuelConsumedLiters,
        s.LoadWeight,
        CASE WHEN s.ArrivalTime <= s.ScheduledArrivalTime THEN 1 ELSE 0 END
    FROM StagingFleetTrips s
    INNER JOIN DimTime t ON CAST(s.DepartureTime AS DATETIME2(0)) = t.DateTime
    INNER JOIN DimFleet f ON s.VehicleID = f.VehicleID
    INNER JOIN DimGeography g_origin ON s.OriginLocationID = g_origin.LocationID
    INNER JOIN DimGeography g_dest ON s.DestinationLocationID = g_dest.LocationID
    WHERE NOT EXISTS (
        SELECT 1 FROM FactFleetTrips ft 
        WHERE ft.TripID = s.TripID
    );
END;
GO
```

### Step 4: Create Analytical Views

```sql
-- Cross-fact KPI view: Fleet efficiency vs warehouse dwell time
CREATE VIEW vw_FleetWarehouseEfficiency AS
SELECT 
    t.Date,
    t.MonthName,
    g.Country,
    g.LocationName,
    -- Warehouse metrics
    SUM(CASE WHEN wh.OperationType = 'Picking' THEN wh.DurationMinutes ELSE 0 END) AS TotalPickingMinutes,
    AVG(wh.DwellTimeHours) AS AvgDwellTimeHours,
    -- Fleet metrics
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelEfficiency,
    SUM(CASE WHEN ft.OnTimeDelivery = 1 THEN 1 ELSE 0 END) * 100.0 / COUNT(ft.TripKey) AS OnTimeDeliveryPercent
FROM DimTime t
LEFT JOIN FactWarehouseOperations wh ON t.TimeKey = wh.TimeKey
LEFT JOIN FactFleetTrips ft ON t.TimeKey = ft.TimeKey
INNER JOIN DimGeography g ON COALESCE(wh.GeographyKey, ft.OriginGeographyKey) = g.GeographyKey
GROUP BY t.Date, t.MonthName, g.Country, g.LocationName;
GO

-- Predictive bottleneck index view
CREATE VIEW vw_PredictiveBottleneckIndex AS
SELECT 
    t.Date,
    g.LocationName,
    p.GravityZone,
    -- Bottleneck scoring factors
    AVG(wh.DwellTimeHours) AS AvgDwellTime,
    STDEV(wh.DwellTimeHours) AS DwellTimeVariance,
    COUNT(*) AS OperationCount,
    SUM(CASE WHEN wh.ErrorsCount > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS ErrorRatePercent,
    -- Composite bottleneck score (higher = more congestion risk)
    (AVG(wh.DwellTimeHours) * 0.4) + 
    (STDEV(wh.DwellTimeHours) * 0.3) + 
    ((SUM(CASE WHEN wh.ErrorsCount > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*)) * 0.3) AS BottleneckScore
FROM FactWarehouseOperations wh
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
GROUP BY t.Date, g.LocationName, p.GravityZone;
GO
```

### Step 5: Configure Automated Alerts

```sql
-- Alert stored procedure for high fleet idling
CREATE PROCEDURE AlertHighFleetIdling
    @ThresholdPercent DECIMAL(5,2) = 15.0
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        f.VehicleID,
        f.VehicleType,
        AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) AS IdlePercent,
        COUNT(*) AS TripCount
    INTO #IdlingViolations
    FROM FactFleetTrips ft
    INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
    INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
    WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
    GROUP BY f.VehicleID, f.VehicleType
    HAVING AVG(ft.IdleTimeMinutes * 100.0 / NULLIF(ft.DurationMinutes, 0)) > @ThresholdPercent;
    
    -- Send email/insert into alert table
    IF EXISTS (SELECT 1 FROM #IdlingViolations)
    BEGIN
        -- Implementation: integrate with SQL Server Database Mail or external API
        PRINT 'Alert: High idling detected for ' + CAST((SELECT COUNT(*) FROM #IdlingViolations) AS VARCHAR) + ' vehicles';
    END;
END;
GO
```

## Power BI Configuration

### Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-sql-server.database.windows.net` (use env var: `${SQL_SERVER_HOST}`)
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: DirectQuery (for real-time) or Import (for performance)

### Import Core Tables

```dax
// DAX measure: Fleet Efficiency Score
Fleet Efficiency Score = 
VAR AvgIdlePercent = AVERAGE(FactFleetTrips[IdleTimeMinutes]) / AVERAGE(FactFleetTrips[DurationMinutes])
VAR FuelEfficiency = AVERAGE(DIVIDE(FactFleetTrips[FuelConsumedLiters], FactFleetTrips[DistanceKm]))
VAR OnTimeRate = DIVIDE(COUNTROWS(FILTER(FactFleetTrips, FactFleetTrips[OnTimeDelivery] = TRUE)), COUNTROWS(FactFleetTrips))
RETURN
(1 - AvgIdlePercent) * 0.4 + (1 - FuelEfficiency / 0.1) * 0.3 + OnTimeRate * 0.3

// DAX measure: Warehouse Gravity Utilization
Warehouse Gravity Utilization = 
VAR HighGravityOps = CALCULATE(COUNTROWS(FactWarehouseOperations), DimProductGravity[GravityZone] = "High")
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
RETURN
DIVIDE(HighGravityOps, TotalOps, 0)

// DAX measure: Cross-Fact Dwell-to-Fuel Ratio
Dwell to Fuel Ratio = 
VAR AvgDwellHours = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgFuelPerTrip = AVERAGE(FactFleetTrips[FuelConsumedLiters])
RETURN
DIVIDE(AvgDwellHours, AvgFuelPerTrip, 0)

// DAX calculated column: Time Period Classification
Time Period = 
SWITCH(
    TRUE(),
    DimTime[HourOfDay] >= 6 && DimTime[HourOfDay] < 12, "Morning",
    DimTime[HourOfDay] >= 12 && DimTime[HourOfDay] < 18, "Afternoon",
    DimTime[HourOfDay] >= 18 && DimTime[HourOfDay] < 22, "Evening",
    "Night"
)
```

### Create Relationships in Power BI

```dax
// Relationship configuration (done via Model view, not code):
// FactWarehouseOperations[TimeKey] → DimTime[TimeKey] (Many-to-One)
// FactWarehouseOperations[GeographyKey] → DimGeography[GeographyKey] (Many-to-One)
// FactWarehouseOperations[ProductKey] → DimProductGravity[ProductKey] (Many-to-One)
// FactFleetTrips[TimeKey] → DimTime[TimeKey] (Many-to-One)
// FactFleetTrips[FleetKey] → DimFleet[FleetKey] (Many-to-One)
// FactFleetTrips[OriginGeographyKey] → DimGeography[GeographyKey] (Many-to-One, inactive)
// FactFleetTrips[DestinationGeographyKey] → DimGeography[GeographyKey] (Many-to-One, active)
```

### Configure Row-Level Security

```dax
// Create role: "Warehouse Supervisors"
// Filter on DimGeography table:
[Country] = USERPRINCIPALNAME() // Replace with actual user mapping logic

// Apply in Power BI Desktop → Modeling → Manage Roles → New Role
```

## Common Usage Patterns

### Pattern 1: Analyze Dwell Time by Gravity Zone

```sql
-- SQL query
SELECT 
    p.GravityZone,
    g.LocationName,
    AVG(wh.DwellTimeHours) AS AvgDwellTime,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wh
INNER JOIN DimProductGravity p ON wh.ProductKey = p.ProductKey
INNER JOIN DimGeography g ON wh.GeographyKey = g.GeographyKey
INNER JOIN DimTime t ON wh.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
GROUP BY p.GravityZone, g.LocationName
ORDER BY AvgDwellTime DESC;
```

### Pattern 2: Fleet Fuel Efficiency vs Load Weight

```sql
SELECT 
    f.VehicleType,
    CASE 
        WHEN ft.LoadWeight < 5000 THEN 'Light'
        WHEN ft.LoadWeight < 10000 THEN 'Medium'
        ELSE 'Heavy'
    END AS LoadCategory,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKm, 0)) AS AvgFuelPerKm,
    COUNT(*) AS TripCount
FROM FactFleetTrips ft
INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(MONTH, -3, GETDATE())
GROUP BY f.VehicleType, 
    CASE 
        WHEN ft.LoadWeight < 5000 THEN 'Light'
        WHEN ft.LoadWeight < 10000 THEN 'Medium'
        ELSE 'Heavy'
    END
ORDER BY AvgFuelPerKm DESC;
```

### Pattern 3: Cross-Dock Efficiency Analysis

```sql
SELECT 
    t.Date,
    g.LocationName,
    AVG(cd.DockDwellMinutes) AS AvgDockDwellMinutes,
    AVG(cd.TransferTimeMinutes) AS AvgTransferMinutes,
    COUNT(*) AS CrossDockCount,
    -- Efficiency: lower is better
    AVG(cd.DockDwellMinutes + cd.TransferTimeMinutes) AS TotalCrossDockTime
FROM FactCrossDock cd
INNER JOIN DimTime t ON cd.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON cd.GeographyKey = g.GeographyKey
WHERE t.Date >= DATEADD(DAY, -7, GETDATE())
GROUP BY t.Date, g.LocationName
ORDER BY TotalCrossDockTime DESC;
```

### Pattern 4: Predictive Maintenance Queue (Fleet Triage)

```sql
-- Weighted scoring for proactive maintenance
SELECT 
    f.VehicleID,
    f.VehicleType,
    DATEDIFF(DAY, f.LastMaintenanceDate, GETDATE()) AS DaysSinceLastMaintenance,
    AVG(ft.HarshBrakingEvents) AS AvgHarshBraking,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime,
    -- Composite maintenance priority score
    (DATEDIFF(DAY, f.LastMaintenanceDate, GETDATE()) * 0.4) +
    (AVG(ft.HarshBrakingEvents) * 0.35) +
    (AVG(ft.IdleTimeMinutes) * 0.25) AS MaintenancePriorityScore
FROM DimFleet f
INNER JOIN FactFleetTrips ft ON f.FleetKey = ft.FleetKey
INNER JOIN DimTime t ON ft.TimeKey = t.TimeKey
WHERE t.Date >= DATEADD(DAY, -30, GETDATE())
    AND f.IsActive = 1
GROUP BY f.VehicleID, f.VehicleType, f.LastMaintenanceDate
ORDER BY MaintenancePriorityScore DESC;
```

## Configuration Reference

### Connection String Template

```plaintext
Server=${SQL_SERVER_HOST};Database=LogiFleetPulse;User Id=${SQL_USER};Password=${SQL_PASSWORD};Encrypt=True;TrustServerCertificate=False;
```

### External Data Source Configuration (Polybase)

```sql
-- Enable Polybase (if using external data sources)
sp_configure 'polybase enabled', 1;
RECONFIGURE;
GO

-- Create external data source for telemetry API
CREATE EXTERNAL DATA SOURCE TelemetryAPI
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://yourstorage.blob.core.windows.net/telemetry',
    CREDENTIAL = AzureStorageCredential
);

-- Create external file format
CREATE EXTERNAL FILE FORMAT JSONFormat
WITH (FORMAT_TYPE = JSON);
```

### Scheduling Refresh (SQL Server Agent Jobs)

```sql
-- Create SQL Server Agent job for ETL
USE msdb;
GO

EXEC sp_add_job @job_name = 'LogiFleet_ETL_Job';

EXEC sp_add_jobstep 
    @job_name = 'LogiFleet_ETL_Job',
    @step_name = 'Load Warehouse Operations',
    @subsystem = 'TSQL',
    @command = 'EXEC LoadWarehouseOperations',
    @database_name = 'LogiFleetPulse';

EXEC sp_add_jobstep 
    @job_name = 'LogiFleet_ETL_Job',
    @step_name = 'Load Fleet Trips',
    @subsystem = 'TSQL',
    @command = 'EXEC LoadFleetTrips',
    @database_name = 'LogiFleetPulse';

-- Schedule every 15 minutes
EXEC sp_add_schedule 
    @schedule_name = 'Every15Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule 
    @job_name = 'LogiFleet_ETL_Job',
    @schedule_name = 'Every15Minutes';

EXEC sp_add_jobserver @job_name = 'LogiFleet_ETL_Job';
```

## Troubleshooting

### Issue: Power BI refresh taking too long

**Solution**: Switch from Import to DirectQuery mode, or implement partitioning:

```sql
-- Create partitioned fact table (SQL Server 2016+)
CREATE PARTITION FUNCTION PF_DateRange (DATE)
AS RANGE RIGHT FOR VALUES 
    ('2025-01-01', '2025-04-
