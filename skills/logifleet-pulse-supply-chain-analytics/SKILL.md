---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehouse for logistics intelligence, fleet optimization, and warehouse operations analytics
triggers:
  - "set up logifleet pulse supply chain analytics"
  - "create logistics data warehouse with power bi"
  - "implement warehouse gravity zones and fleet tracking"
  - "build multi-fact star schema for logistics"
  - "configure logifleet pulse dashboards"
  - "deploy supply chain analytics warehouse"
  - "integrate fleet telemetry with warehouse operations"
  - "optimize logistics kpi dashboard"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI agents to help developers deploy and configure LogiFleet Pulse, a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. The system unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a multi-fact star schema data warehouse.

## What LogiFleet Pulse Does

LogiFleet Pulse is a data warehousing and analytics solution for:

- **Warehouse Operations**: Track putaway cycles, pick rates, packing times, dwell spans
- **Fleet Management**: Monitor route segments, fuel consumption, idle time, loading/unloading events
- **Cross-Docking**: Analyze transfers between inbound and outbound operations
- **Predictive Analytics**: Bottleneck detection, maintenance prioritization, capacity simulation
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency, value, and fragility

The platform uses a custom multi-fact star schema with time-phased dimensions and bridge tables for many-to-many relationships.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Access to data sources: WMS, fleet telemetry, ERP systems

### Step 1: Deploy SQL Schema

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeSlot TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    MonthName VARCHAR(20) NOT NULL,
    Quarter INT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationCode VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, RoutNode, Port, etc.
    Continent VARCHAR(50),
    Country VARCHAR(100),
    Region VARCHAR(100),
    City VARCHAR(100),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE
);

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(300) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    ShelfLifeDays INT,
    StandardCost DECIMAL(18,2),
    RetailPrice DECIMAL(18,2),
    -- Gravity scoring components
    PickFrequencyScore INT, -- 1-100
    ValueScore INT, -- 1-100
    FragilityScore INT, -- 1-100
    GravityIndex AS (PickFrequencyScore * 0.5 + ValueScore * 0.3 + FragilityScore * 0.2) PERSISTED,
    GravityZone VARCHAR(20) -- High, Medium, Low
);

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode VARCHAR(50) UNIQUE NOT NULL,
    SupplierName VARCHAR(200) NOT NULL,
    Country VARCHAR(100),
    LeadTimeDaysAvg DECIMAL(10,2),
    LeadTimeDaysStdDev DECIMAL(10,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore INT, -- 1-100
    ReliabilityIndex AS (OnTimeDeliveryPercent * 0.4 + (100 - DefectRatePercent) * 0.3 + ComplianceScore * 0.3) PERSISTED,
    IsActive BIT DEFAULT 1
);

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    SupplierKey INT,
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    BatchNumber VARCHAR(100),
    OrderID VARCHAR(100),
    QuantityHandled INT NOT NULL,
    ProcessingTimeMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneAssigned VARCHAR(50),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    QualityCheckPassed BIT,
    DamageReported BIT DEFAULT 0,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey),
    CONSTRAINT FK_WarehouseOps_Supplier FOREIGN KEY (SupplierKey) REFERENCES DimSupplierReliability(SupplierKey)
);

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID VARCHAR(100) UNIQUE NOT NULL,
    StartTimeKey INT NOT NULL,
    EndTimeKey INT,
    OriginGeographyKey INT NOT NULL,
    DestinationGeographyKey INT NOT NULL,
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50) NOT NULL,
    RouteID VARCHAR(100),
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    PayloadWeightKg DECIMAL(10,2),
    PayloadVolumeM3 DECIMAL(10,2),
    OnTimeDelivery BIT,
    DelayMinutes INT DEFAULT 0,
    WeatherCondition VARCHAR(50),
    TrafficCondition VARCHAR(50),
    MaintenanceIssueReported BIT DEFAULT 0,
    CONSTRAINT FK_FleetTrips_StartTime FOREIGN KEY (StartTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_EndTime FOREIGN KEY (EndTimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Origin FOREIGN KEY (OriginGeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_FleetTrips_Destination FOREIGN KEY (DestinationGeographyKey) REFERENCES DimGeography(GeographyKey)
);

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    InboundTripKey BIGINT,
    OutboundTripKey BIGINT,
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    QuantityTransferred INT NOT NULL,
    TransferTimeMinutes DECIMAL(10,2),
    TemporaryStorageUsed BIT DEFAULT 0,
    CONSTRAINT FK_CrossDock_InboundTrip FOREIGN KEY (InboundTripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_CrossDock_OutboundTrip FOREIGN KEY (OutboundTripKey) REFERENCES FactFleetTrips(TripKey),
    CONSTRAINT FK_CrossDock_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_CrossDock_Geography FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    CONSTRAINT FK_CrossDock_Product FOREIGN KEY (ProductKey) REFERENCES DimProductGravity(ProductKey)
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_WarehouseOps_TimeGeo ON FactWarehouseOperations(TimeKey, GeographyKey);
CREATE NONCLUSTERED INDEX IX_WarehouseOps_Product ON FactWarehouseOperations(ProductKey) INCLUDE (QuantityHandled, DwellTimeHours);
CREATE NONCLUSTERED INDEX IX_FleetTrips_TimeRoute ON FactFleetTrips(StartTimeKey, RouteID);
CREATE NONCLUSTERED INDEX IX_FleetTrips_Vehicle ON FactFleetTrips(VehicleID, StartTimeKey);
```

### Step 2: Populate Dimension Tables

```sql
-- Populate time dimension with 15-minute granularity
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @TimeKey INT;

    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT);
        
        INSERT INTO DimTime (
            TimeKey, FullDateTime, DateKey, TimeSlot, 
            HourOfDay, DayOfWeek, DayName, WeekOfYear,
            MonthNumber, MonthName, Quarter, FiscalYear,
            IsWeekend, IsHoliday
        )
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMdd') AS INT),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            YEAR(@CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END,
            0 -- Holiday logic needs external calendar
        );

        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute for 2 years
EXEC PopulateTimeDimension '2025-01-01', '2026-12-31';
```

### Step 3: Create ETL Stored Procedures

```sql
-- ETL for warehouse operations
CREATE PROCEDURE LoadWarehouseOperations
    @SourceConnectionString NVARCHAR(500)
AS
BEGIN
    -- Example using OPENROWSET for external data
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, BatchNumber, OrderID, QuantityHandled,
        ProcessingTimeMinutes, DwellTimeHours, ZoneAssigned,
        OperatorID, EquipmentID, QualityCheckPassed, DamageReported
    )
    SELECT 
        CAST(FORMAT(src.OperationTimestamp, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        geo.GeographyKey,
        prod.ProductKey,
        supp.SupplierKey,
        src.OperationType,
        src.BatchNumber,
        src.OrderID,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS ProcessingTimeMinutes,
        DATEDIFF(HOUR, src.ArrivalTime, src.ProcessedTime) AS DwellTimeHours,
        src.StorageZone,
        src.OperatorID,
        src.EquipmentID,
        src.QCPassed,
        src.DamageFlag
    FROM 
        OPENROWSET('SQLNCLI', @SourceConnectionString,
            'SELECT * FROM WMS.dbo.Operations WHERE LoadedFlag = 0') AS src
    INNER JOIN DimGeography geo ON src.LocationCode = geo.LocationCode
    INNER JOIN DimProductGravity prod ON src.SKU = prod.SKU
    LEFT JOIN DimSupplierReliability supp ON src.SupplierCode = supp.SupplierCode;

    -- Mark source records as loaded
    -- (implementation depends on source system)
END;
GO

-- ETL for fleet trips
CREATE PROCEDURE LoadFleetTrips
AS
BEGIN
    INSERT INTO FactFleetTrips (
        TripID, StartTimeKey, EndTimeKey, OriginGeographyKey, DestinationGeographyKey,
        VehicleID, DriverID, RouteID, PlannedDistanceKm, ActualDistanceKm,
        FuelConsumedLiters, IdleTimeMinutes, LoadingTimeMinutes, UnloadingTimeMinutes,
        PayloadWeightKg, PayloadVolumeM3, OnTimeDelivery, DelayMinutes,
        WeatherCondition, TrafficCondition, MaintenanceIssueReported
    )
    SELECT 
        ext.TripID,
        CAST(FORMAT(ext.DepartureTime, 'yyyyMMddHHmm') AS INT),
        CAST(FORMAT(ext.ArrivalTime, 'yyyyMMddHHmm') AS INT),
        origin.GeographyKey,
        dest.GeographyKey,
        ext.VehicleID,
        ext.DriverID,
        ext.RouteID,
        ext.PlannedDistance,
        ext.ActualDistance,
        ext.FuelUsed,
        ext.IdleMinutes,
        ext.LoadingMinutes,
        ext.UnloadingMinutes,
        ext.PayloadWeight,
        ext.PayloadVolume,
        CASE WHEN DATEDIFF(MINUTE, ext.PlannedArrival, ext.ArrivalTime) <= 15 THEN 1 ELSE 0 END,
        CASE WHEN ext.ArrivalTime > ext.PlannedArrival 
             THEN DATEDIFF(MINUTE, ext.PlannedArrival, ext.ArrivalTime) 
             ELSE 0 END,
        ext.WeatherCondition,
        ext.TrafficCondition,
        ext.MaintenanceFlag
    FROM FleetTelemetry.dbo.CompletedTrips ext
    INNER JOIN DimGeography origin ON ext.OriginCode = origin.LocationCode
    INNER JOIN DimGeography dest ON ext.DestinationCode = dest.LocationCode
    WHERE ext.LoadedToWarehouse = 0;
END;
GO
```

### Step 4: Create Analytical Views

```sql
-- Cross-fact KPI view: Dwell time vs. fleet idling cost
CREATE VIEW vw_DwellTimeFleetImpact AS
SELECT 
    t.DateKey,
    t.MonthName,
    g.Region,
    p.GravityZone,
    COUNT(DISTINCT wo.OperationKey) AS WarehouseOps,
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    COUNT(DISTINCT ft.TripKey) AS FleetTrips,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMinutes,
    SUM(ft.FuelConsumedLiters * 1.50) AS TotalFuelCostUSD, -- Assumes $1.50/liter
    AVG(wo.DwellTimeHours) * AVG(ft.IdleTimeMinutes) AS DwellIdleIndex
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips ft ON t.DateKey = CAST(FORMAT(ft.StartTimeKey, 'yyyyMMdd') AS INT)
    AND g.GeographyKey = ft.OriginGeographyKey
GROUP BY t.DateKey, t.MonthName, g.Region, p.GravityZone;
GO

-- Predictive bottleneck detection view
CREATE VIEW vw_BottleneckPrediction AS
WITH CapacityMetrics AS (
    SELECT 
        GeographyKey,
        DATEPART(HOUR, FullDateTime) AS HourOfDay,
        COUNT(*) AS OperationCount,
        AVG(ProcessingTimeMinutes) AS AvgProcessingTime,
        STDEV(ProcessingTimeMinutes) AS ProcessingTimeStdDev
    FROM FactWarehouseOperations wo
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMdd') AS INT)
    GROUP BY GeographyKey, DATEPART(HOUR, FullDateTime)
)
SELECT 
    g.LocationName,
    cm.HourOfDay,
    cm.OperationCount,
    cm.AvgProcessingTime,
    cm.ProcessingTimeStdDev,
    CASE 
        WHEN cm.OperationCount > 100 AND cm.ProcessingTimeStdDev > 15 THEN 'High Risk'
        WHEN cm.OperationCount > 75 AND cm.ProcessingTimeStdDev > 10 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS BottleneckRisk
FROM CapacityMetrics cm
INNER JOIN DimGeography g ON cm.GeographyKey = g.GeographyKey
WHERE g.LocationType = 'Warehouse';
GO
```

### Step 5: Configure Automated Alerts

```sql
-- Alert stored procedure for KPI breaches
CREATE PROCEDURE AlertOnKPIBreach
    @FleetIdleThresholdPercent DECIMAL(5,2) = 15.0,
    @DwellTimeThresholdHours DECIMAL(10,2) = 72.0
AS
BEGIN
    -- Fleet idling alert
    DECLARE @IdleViolations TABLE (VehicleID VARCHAR(50), IdlePercent DECIMAL(5,2), TripID VARCHAR(100));
    
    INSERT INTO @IdleViolations
    SELECT 
        VehicleID,
        (IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, st.FullDateTime, et.FullDateTime), 0)) * 100 AS IdlePercent,
        TripID
    FROM FactFleetTrips ft
    INNER JOIN DimTime st ON ft.StartTimeKey = st.TimeKey
    INNER JOIN DimTime et ON ft.EndTimeKey = et.TimeKey
    WHERE st.DateKey = CAST(FORMAT(GETDATE(), 'yyyyMMdd') AS INT)
        AND (IdleTimeMinutes / NULLIF(DATEDIFF(MINUTE, st.FullDateTime, et.FullDateTime), 0)) * 100 > @FleetIdleThresholdPercent;

    IF EXISTS (SELECT 1 FROM @IdleViolations)
    BEGIN
        -- Send email or Teams notification
        -- Integration with sp_send_dbmail or external webhook
        SELECT 'ALERT: Fleet Idle Time Exceeded' AS AlertType,
               VehicleID, IdlePercent, TripID
        FROM @IdleViolations;
    END

    -- Dwell time alert
    DECLARE @DwellViolations TABLE (SKU VARCHAR(100), DwellHours DECIMAL(10,2), LocationName VARCHAR(200));
    
    INSERT INTO @DwellViolations
    SELECT 
        p.SKU,
        wo.DwellTimeHours,
        g.LocationName
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
    WHERE t.DateKey >= CAST(FORMAT(DATEADD(DAY, -1, GETDATE()), 'yyyyMMdd') AS INT)
        AND wo.DwellTimeHours > @DwellTimeThresholdHours;

    IF EXISTS (SELECT 1 FROM @DwellViolations)
    BEGIN
        SELECT 'ALERT: Dwell Time Exceeded' AS AlertType,
               SKU, DwellHours, LocationName
        FROM @DwellViolations;
    END
END;
GO

-- Schedule with SQL Server Agent (example T-SQL)
USE msdb;
GO

EXEC sp_add_job @job_name = 'LogiFleet_KPI_Alerts', @enabled = 1;

EXEC sp_add_jobstep 
    @job_name = 'LogiFleet_KPI_Alerts',
    @step_name = 'Check KPI Breaches',
    @subsystem = 'TSQL',
    @command = 'EXEC LogiFleetPulse.dbo.AlertOnKPIBreach;',
    @database_name = 'LogiFleetPulse';

EXEC sp_add_schedule 
    @schedule_name = 'Every_15_Minutes',
    @freq_type = 4, -- Daily
    @freq_interval = 1,
    @freq_subday_type = 4, -- Minutes
    @freq_subday_interval = 15;

EXEC sp_attach_schedule 
    @job_name = 'LogiFleet_KPI_Alerts',
    @schedule_name = 'Every_15_Minutes';

EXEC sp_add_jobserver @job_name = 'LogiFleet_KPI_Alerts';
GO
```

## Power BI Configuration

### Step 1: Connect to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `your-sql-server.database.windows.net` (or local instance)
4. Database: `LogiFleetPulse`
5. Data Connectivity Mode: **DirectQuery** (for real-time) or **Import** (for performance)

### Step 2: Load Fact and Dimension Tables

```powerquery
// Power Query M code for loading tables
let
    Source = Sql.Database("YourServer", "LogiFleetPulse"),
    FactWarehouseOps = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FactFleetTrips = Source{[Schema="dbo",Item="FactFleetTrips"]}[Data],
    FactCrossDock = Source{[Schema="dbo",Item="FactCrossDock"]}[Data],
    DimTime = Source{[Schema="dbo",Item="DimTime"]}[Data],
    DimGeography = Source{[Schema="dbo",Item="DimGeography"]}[Data],
    DimProductGravity = Source{[Schema="dbo",Item="DimProductGravity"]}[Data],
    DimSupplierReliability = Source{[Schema="dbo",Item="DimSupplierReliability"]}[Data]
in
    #"Your table name here"
```

### Step 3: Define Relationships in Model View

- **FactWarehouseOperations[TimeKey]** → **DimTime[TimeKey]** (Many-to-One)
- **FactWarehouseOperations[GeographyKey]** → **DimGeography[GeographyKey]** (Many-to-One)
- **FactWarehouseOperations[ProductKey]** → **DimProductGravity[ProductKey]** (Many-to-One)
- **FactFleetTrips[StartTimeKey]** → **DimTime[TimeKey]** (Many-to-One, role-playing dimension)
- **FactFleetTrips[OriginGeographyKey]** → **DimGeography[GeographyKey]** (Many-to-One)

### Step 4: Create DAX Measures

```dax
// Total warehouse operations
TotalOperations = COUNTROWS(FactWarehouseOperations)

// Average dwell time
AvgDwellTime = 
AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Fleet utilization percentage
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[ActualDistanceKm]),
    SUM(FactFleetTrips[PlannedDistanceKm]),
    0
) * 100

// On-time delivery rate
OnTimeDeliveryRate = 
DIVIDE(
    CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeDelivery] = TRUE()),
    COUNTROWS(FactFleetTrips),
    0
) * 100

// Warehouse gravity zone efficiency
GravityZoneEfficiency = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]),
    DimProductGravity[GravityZone] = "High"
) / 
CALCULATE(
    AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes]),
    DimProductGravity[GravityZone] = "Low"
)

// Cross-fact composite: Dwell-Idle correlation
DwellIdleCorrelation = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
RETURN AvgDwell * AvgIdle

// Predictive bottleneck score
BottleneckScore = 
VAR OpCount = COUNTROWS(FactWarehouseOperations)
VAR AvgProcessTime = AVERAGE(FactWarehouseOperations[ProcessingTimeMinutes])
VAR StdDev = STDEV.P(FactWarehouseOperations[ProcessingTimeMinutes])
RETURN 
    IF(
        OpCount > 100 && StdDev > 15,
        100,
        IF(OpCount > 75 && StdDev > 10, 60, 20)
    )

// Fuel cost per kilometer
FuelCostPerKm = 
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedLiters]) * 1.50, // $1.50/liter from env
    SUM(FactFleetTrips[ActualDistanceKm]),
    0
)

// Year-over-year growth
YoY_Operations = 
VAR CurrentYearOps = 
    CALCULATE(
        [TotalOperations],
        FILTER(ALL(DimTime), DimTime[FiscalYear] = MAX(DimTime[FiscalYear]))
    )
VAR PriorYearOps = 
    CALCULATE(
        [TotalOperations],
        FILTER(ALL(DimTime), DimTime[FiscalYear] = MAX(DimTime[FiscalYear]) - 1)
    )
RETURN 
    DIVIDE(CurrentYearOps - PriorYearOps, PriorYearOps, 0) * 100
```

### Step 5: Build Dashboard Visuals

**Executive Overview Page:**
- KPI cards: Total Operations, Avg Dwell Time, Fleet Utilization, On-Time Delivery Rate
- Line chart: Operations by Month (DimTime[MonthName], TotalOperations)
- Map: Warehouse locations colored by BottleneckScore
- Clustered column chart: Operations by Gravity Zone

**Fleet Optimization Page:**
- Table: VehicleID, TripID, IdleTimeMinutes, FuelCostPerKm, OnTimeDelivery
- Scatter plot: ActualDistanceKm (X) vs. FuelConsumedLiters (Y), sized by PayloadWeightKg
- Gauge: Fleet Utilization vs. 85% target
- Matrix: OriginGeographyKey, DestinationGeographyKey, Avg DelayMinutes

**Warehouse Efficiency Page:**
- Heatmap: DwellTimeHours by LocationName and ProductCategory
- Donut chart: Operations by OperationType
- Waterfall chart: Avg ProcessingTimeMinutes by HourOfDay
- Slicer: GravityZone (High, Medium, Low)

### Step 6: Row-Level Security

```dax
// Create role: Regional Managers
// Filter expression on DimGeography table:
DimGeography[Region] = USERPRINCIPALNAME()

// Or using lookup table:
VAR UserRegion = 
    LOOKUPVALUE(
        UserRegionMapping[Region],
        UserRegionMapping[Email],
        USERPRINCIPALNAME()
    )
RETURN DimGeography[Region] = UserRegion
```

## Configuration Files

### config.json (Connection Strings)

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectoryIntegrated"
  },
  "data_sources": {
    "wms_connection": "${WMS_CONNECTION_STRING}",
    "fleet_telemetry_api": "${FLEET_API_ENDPOINT}",
    "weather_api_key": "${WEATHER_API_KEY}"
  },
  "etl_schedule": {
    "warehouse_ops_interval_minutes": 15,
    "fleet_trips_interval_minutes": 30,
    "dimension_refresh_interval_hours": 24
  },
  "alert_thresholds": {
    "fleet_idle_percent": 15.0,
    "dwell_time_hours": 72.0,
    "on_time_delivery_target_percent": 95.0
  },
  "powerbi": {
    "workspace_id": "${POWERBI_WORKSPACE_ID}",
    "dataset_id": "${POWERBI_DATASET_ID}",
    "refresh_schedule_cron": "0 */15 * * *"
  }
}
```

## Common Patterns

### Pattern 1: Adding New Data Source

```sql
