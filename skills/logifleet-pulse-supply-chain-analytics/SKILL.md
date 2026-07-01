---
name: logifleet-pulse-supply-chain-analytics
description: Advanced MS SQL Server & Power BI data warehousing and fleet logistics analytics engine with multi-fact star schema for supply chain intelligence
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard with SQL Server"
  - "implement multi-fact star schema for warehouse and fleet data"
  - "create supply chain KPI dashboards in Power BI"
  - "build logistics data warehouse with temporal dimensions"
  - "integrate warehouse management system with fleet telemetry"
  - "deploy cross-modal supply chain analytics platform"
  - "configure real-time logistics intelligence engine"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

This skill enables AI coding agents to help developers implement LogiFleet Pulse, a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, and supply chain data into a unified Power BI analytics platform backed by MS SQL Server.

## What LogiFleet Pulse Does

LogiFleet Pulse is a data warehousing and analytics solution that:

- **Unifies multi-source logistics data** (warehouse operations, fleet GPS/telemetry, supplier data, external APIs)
- **Implements multi-fact star schema** with time-phased dimensions for cross-fact KPI queries
- **Provides real-time Power BI dashboards** with 15-minute refresh intervals
- **Enables predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Supports role-based access control** with row-level security for compliance

Core architectural components:
- MS SQL Server 2019+ data warehouse with fact and dimension tables
- Power BI template (`.pbit`) with pre-built semantic layer
- Stored procedures for incremental data loading
- Automated alerting system for KPI threshold breaches

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Database access with CREATE permissions
- Data sources: WMS, TMS, or telemetry APIs

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Run the schema deployment script

USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension with 15-minute granularity
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay TINYINT NOT NULL,
    QuarterHourBucket TINYINT NOT NULL, -- 0-3 for each quarter hour
    DayOfWeek TINYINT NOT NULL,
    DayName VARCHAR(10) NOT NULL,
    WeekOfYear TINYINT NOT NULL,
    MonthNumber TINYINT NOT NULL,
    MonthName VARCHAR(10) NOT NULL,
    Quarter TINYINT NOT NULL,
    FiscalYear INT NOT NULL,
    IsWeekday BIT NOT NULL,
    IsBusinessHour BIT NOT NULL
);

CREATE CLUSTERED INDEX IX_DimTime_FullDateTime ON DimTime(FullDateTime);
CREATE NONCLUSTERED INDEX IX_DimTime_DateKey ON DimTime(DateKey);
GO

-- Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT IDENTITY(1,1) PRIMARY KEY,
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, Distribution Center, Route Node
    AddressLine1 VARCHAR(200),
    City VARCHAR(100),
    StateProvince VARCHAR(100),
    Country VARCHAR(100),
    Region VARCHAR(100),
    Continent VARCHAR(50),
    PostalCode VARCHAR(20),
    Latitude DECIMAL(10, 8),
    Longitude DECIMAL(11, 8),
    TimeZone VARCHAR(50),
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE
);

CREATE NONCLUSTERED INDEX IX_DimGeography_LocationType ON DimGeography(LocationType);
CREATE NONCLUSTERED INDEX IX_DimGeography_Country ON DimGeography(Country);
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT IDENTITY(1,1) PRIMARY KEY,
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(300) NOT NULL,
    Category VARCHAR(100),
    Subcategory VARCHAR(100),
    VelocityScore DECIMAL(5, 2), -- Calculated from pick frequency
    ValueScore DECIMAL(5, 2), -- Based on unit price
    FragilityScore DECIMAL(5, 2), -- 0-100 scale
    GravityScore AS (VelocityScore * 0.5 + ValueScore * 0.3 + (100 - FragilityScore) * 0.2) PERSISTED,
    OptimalZoneType VARCHAR(50), -- Fast-Mover, Slow-Mover, Bulk, Cold-Storage
    Weight DECIMAL(10, 3),
    Volume DECIMAL(10, 3),
    UnitOfMeasure VARCHAR(20),
    RequiresTemperatureControl BIT,
    OptimalTemperatureMin DECIMAL(5, 2),
    OptimalTemperatureMax DECIMAL(5, 2)
);

CREATE NONCLUSTERED INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);
CREATE NONCLUSTERED INDEX IX_DimProductGravity_Category ON DimProductGravity(Category);
GO

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT IDENTITY(1,1) PRIMARY KEY,
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(300) NOT NULL,
    Country VARCHAR(100),
    AverageLeadTimeDays DECIMAL(6, 2),
    LeadTimeVarianceDays DECIMAL(6, 2),
    DefectRatePercentage DECIMAL(5, 2),
    OnTimeDeliveryRate DECIMAL(5, 2),
    ComplianceScore DECIMAL(5, 2), -- 0-100
    ReliabilityGrade AS (
        CASE 
            WHEN OnTimeDeliveryRate >= 95 AND DefectRatePercentage <= 2 THEN 'A'
            WHEN OnTimeDeliveryRate >= 85 AND DefectRatePercentage <= 5 THEN 'B'
            WHEN OnTimeDeliveryRate >= 70 AND DefectRatePercentage <= 10 THEN 'C'
            ELSE 'D'
        END
    ) PERSISTED,
    LastReviewDate DATE,
    IsActive BIT DEFAULT 1
);
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    BatchNumber VARCHAR(100),
    OrderID VARCHAR(100),
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT,
    CycleTimeMinutes DECIMAL(8, 2),
    ZoneAssignment VARCHAR(50),
    EmployeeID VARCHAR(50),
    EquipmentID VARCHAR(50),
    ErrorOccurred BIT DEFAULT 0,
    ErrorType VARCHAR(100),
    TemperatureAtOperation DECIMAL(5, 2),
    CostAmount DECIMAL(12, 2)
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_OpType ON FactWarehouseOperations(OperationType);
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouse_Columnstore ON FactWarehouseOperations;
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TripID VARCHAR(100) NOT NULL,
    StartTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    EndTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50),
    DistanceKM DECIMAL(10, 2),
    FuelConsumedLiters DECIMAL(10, 3),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DrivingTimeMinutes INT,
    AverageSpeedKPH DECIMAL(6, 2),
    MaxSpeedKPH DECIMAL(6, 2),
    HarshBrakingEvents INT DEFAULT 0,
    RapidAccelerationEvents INT DEFAULT 0,
    LoadWeightKG DECIMAL(10, 2),
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT DEFAULT 0,
    MaintenanceAlertTriggered BIT DEFAULT 0,
    FuelCostAmount DECIMAL(10, 2),
    TotalTripCost DECIMAL(12, 2)
);

CREATE NONCLUSTERED INDEX IX_FactFleet_StartTime ON FactFleetTrips(StartTimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
CREATE NONCLUSTERED INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleet_Columnstore ON FactFleetTrips;
GO

-- Create cross-dock operations fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ArrivalTimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    DepartureTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    QuantityTransferred INT NOT NULL,
    DwellTimeMinutes INT,
    TemperatureMaintained BIT,
    QualityCheckPassed BIT DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_FactCrossDock_Arrival ON FactCrossDock(ArrivalTimeKey);
CREATE NONCLUSTERED INDEX IX_FactCrossDock_Product ON FactCrossDock(ProductKey);
GO
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate time dimension
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = CAST(@StartDate AS DATETIME2);
    DECLARE @EndDateTime DATETIME2 = CAST(DATEADD(DAY, 1, @EndDate) AS DATETIME2);
    
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        INSERT INTO DimTime (
            TimeKey,
            FullDateTime,
            DateKey,
            TimeOfDay,
            HourOfDay,
            QuarterHourBucket,
            DayOfWeek,
            DayName,
            WeekOfYear,
            MonthNumber,
            MonthName,
            Quarter,
            FiscalYear,
            IsWeekday,
            IsBusinessHour
        )
        VALUES (
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMdd') AS INT),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime) / 15,
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            YEAR(@CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) BETWEEN 2 AND 6 THEN 1 ELSE 0 END,
            CASE WHEN DATEPART(HOUR, @CurrentDateTime) BETWEEN 8 AND 17 
                 AND DATEPART(WEEKDAY, @CurrentDateTime) BETWEEN 2 AND 6 THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate for 2 years
EXEC PopulateTimeDimension '2025-01-01', '2026-12-31';
GO
```

### Step 3: Create Data Loading Procedures

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- This is a template - adapt to your WMS data source
    INSERT INTO FactWarehouseOperations (
        TimeKey,
        GeographyKey,
        ProductKey,
        SupplierKey,
        OperationType,
        BatchNumber,
        OrderID,
        QuantityHandled,
        DwellTimeMinutes,
        CycleTimeMinutes,
        ZoneAssignment,
        EmployeeID,
        EquipmentID,
        ErrorOccurred,
        CostAmount
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        src.OperationType,
        src.BatchNumber,
        src.OrderID,
        src.Quantity,
        DATEDIFF(MINUTE, src.StartTime, src.EndTime) AS DwellTimeMinutes,
        src.CycleTimeMinutes,
        src.ZoneAssignment,
        src.EmployeeID,
        src.EquipmentID,
        src.ErrorOccurred,
        src.CostAmount
    FROM YourWarehouseSource src
    INNER JOIN DimTime t ON FORMAT(src.OperationDateTime, 'yyyyMMddHHmm') = t.TimeKey
    INNER JOIN DimGeography g ON src.LocationID = g.LocationID
    INNER JOIN DimProductGravity p ON src.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON src.SupplierID = s.SupplierID
    WHERE src.OperationDateTime > @LastLoadDateTime
        AND src.OperationDateTime <= GETDATE();
END;
GO

-- Incremental load procedure for fleet trips
CREATE PROCEDURE LoadFleetTrips
    @LastLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    INSERT INTO FactFleetTrips (
        TripID,
        StartTimeKey,
        EndTimeKey,
        OriginGeographyKey,
        DestinationGeographyKey,
        VehicleID,
        DriverID,
        DistanceKM,
        FuelConsumedLiters,
        IdleTimeMinutes,
        DrivingTimeMinutes,
        LoadWeightKG,
        FuelCostAmount,
        TotalTripCost
    )
    SELECT 
        src.TripID,
        t1.TimeKey AS StartTimeKey,
        t2.TimeKey AS EndTimeKey,
        g1.GeographyKey AS OriginGeographyKey,
        g2.GeographyKey AS DestinationGeographyKey,
        src.VehicleID,
        src.DriverID,
        src.DistanceKM,
        src.FuelConsumedLiters,
        src.IdleTimeMinutes,
        src.DrivingTimeMinutes,
        src.LoadWeightKG,
        src.FuelConsumedLiters * @FuelPricePerLiter AS FuelCostAmount,
        src.TotalTripCost
    FROM YourFleetSource src
    INNER JOIN DimTime t1 ON FORMAT(src.StartDateTime, 'yyyyMMddHHmm') = t1.TimeKey
    LEFT JOIN DimTime t2 ON FORMAT(src.EndDateTime, 'yyyyMMddHHmm') = t2.TimeKey
    INNER JOIN DimGeography g1 ON src.OriginLocationID = g1.LocationID
    INNER JOIN DimGeography g2 ON src.DestinationLocationID = g2.LocationID
    WHERE src.StartDateTime > @LastLoadDateTime
        AND src.StartDateTime <= GETDATE();
END;
GO
```

### Step 4: Configure Automated Alerts

```sql
-- Alert configuration table
CREATE TABLE AlertConfiguration (
    AlertID INT IDENTITY(1,1) PRIMARY KEY,
    AlertName VARCHAR(200) NOT NULL,
    MetricType VARCHAR(100) NOT NULL,
    ThresholdValue DECIMAL(12, 2) NOT NULL,
    ComparisonOperator VARCHAR(10) NOT NULL, -- >, <, >=, <=, =
    IsActive BIT DEFAULT 1,
    NotificationEmail VARCHAR(500),
    NotificationSMS VARCHAR(500)
);

-- Insert sample alerts
INSERT INTO AlertConfiguration (AlertName, MetricType, ThresholdValue, ComparisonOperator, NotificationEmail)
VALUES 
    ('High Fleet Idle Time', 'IdleTimePercentage', 15.0, '>', '${LOGISTICS_ALERT_EMAIL}'),
    ('Warehouse Dwell Time Exceeded', 'DwellTimeMinutes', 180, '>', '${WAREHOUSE_ALERT_EMAIL}'),
    ('Temperature Out of Range', 'TemperatureDeviation', 2.0, '>', '${COLD_CHAIN_ALERT_EMAIL}'),
    ('Fleet Maintenance Alert', 'MaintenanceScore', 70, '>', '${FLEET_MANAGER_EMAIL}');

-- Alert monitoring stored procedure
CREATE PROCEDURE CheckAndSendAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check fleet idle time
    IF EXISTS (
        SELECT 1
        FROM FactFleetTrips
        WHERE StartTimeKey >= CAST(FORMAT(DATEADD(HOUR, -24, GETDATE()), 'yyyyMMddHHmm') AS INT)
        GROUP BY VehicleID
        HAVING (SUM(IdleTimeMinutes) * 100.0 / SUM(DrivingTimeMinutes + IdleTimeMinutes)) > 
               (SELECT ThresholdValue FROM AlertConfiguration WHERE MetricType = 'IdleTimePercentage')
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded threshold in last 24 hours';
        -- Integration point: Send to email service using sp_send_dbmail or external API
        -- EXEC msdb.dbo.sp_send_dbmail ...
    END;
    
    -- Check warehouse dwell time
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations
        WHERE TimeKey >= CAST(FORMAT(DATEADD(HOUR, -24, GETDATE()), 'yyyyMMddHHmm') AS INT)
          AND DwellTimeMinutes > (SELECT ThresholdValue FROM AlertConfiguration WHERE MetricType = 'DwellTimeMinutes')
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Warehouse dwell time exceeded threshold';
        -- Send alert
    END;
END;
GO

-- Schedule via SQL Server Agent (run every 15 minutes)
-- Alternatively, use Azure Functions or external scheduler
```

## Power BI Configuration

### Step 1: Open Power BI Template

1. Download the repository and locate `LogiFleet_Pulse_Master.pbit`
2. Open in Power BI Desktop
3. When prompted, enter your SQL Server connection details:
   - Server: `${SQL_SERVER_HOST}`
   - Database: `LogiFleetPulse`
   - Authentication: Windows or SQL Server based on your setup

### Step 2: Configure Data Refresh

```powerquery
// Power Query M code for incremental refresh
// Configure in Power BI Desktop > Transform Data

let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FactWarehouseOperations = Source{[Schema="dbo",Item="FactWarehouseOperations"]}[Data],
    FilteredRows = Table.SelectRows(
        FactWarehouseOperations, 
        each [TimeKey] >= Number.From(Text.From(DateTime.ToText(RangeStart, "yyyyMMddHHmm")))
         and [TimeKey] < Number.From(Text.From(DateTime.ToText(RangeEnd, "yyyyMMddHHmm")))
    )
in
    FilteredRows
```

### Step 3: Create Composite KPI Measures

```dax
// DAX measures for cross-fact KPIs

// Total dwell time across warehouse
TotalDwellTimeHours = 
    SUM(FactWarehouseOperations[DwellTimeMinutes]) / 60

// Fleet utilization percentage
FleetUtilizationPercent = 
    DIVIDE(
        SUM(FactFleetTrips[DrivingTimeMinutes]),
        SUM(FactFleetTrips[DrivingTimeMinutes]) + SUM(FactFleetTrips[IdleTimeMinutes]),
        0
    ) * 100

// Cross-fact KPI: Dwell time cost impact
DwellTimeCostImpact = 
    VAR DwellCostPerHour = 25 // Configurable parameter
    RETURN
        [TotalDwellTimeHours] * DwellCostPerHour

// Warehouse-to-Fleet efficiency ratio
WarehouseFleetEfficiency = 
    DIVIDE(
        CALCULATE(
            AVERAGE(FactWarehouseOperations[CycleTimeMinutes]),
            FactWarehouseOperations[OperationType] = "Shipping"
        ),
        CALCULATE(
            AVERAGE(FactFleetTrips[LoadingTimeMinutes]),
            FactFleetTrips[LoadingTimeMinutes] > 0
        ),
        1
    )

// Product gravity-weighted throughput
GravityWeightedThroughput = 
    SUMX(
        FactWarehouseOperations,
        FactWarehouseOperations[QuantityHandled] * 
        RELATED(DimProductGravity[GravityScore])
    )

// Time-phased demand elasticity
DemandElasticity = 
    VAR CurrentPeriodDemand = 
        CALCULATE(
            SUM(FactWarehouseOperations[QuantityHandled]),
            FactWarehouseOperations[OperationType] = "Shipping"
        )
    VAR PreviousPeriodDemand = 
        CALCULATE(
            SUM(FactWarehouseOperations[QuantityHandled]),
            FactWarehouseOperations[OperationType] = "Shipping",
            DATEADD(DimTime[FullDateTime], -7, DAY)
        )
    RETURN
        DIVIDE(
            (CurrentPeriodDemand - PreviousPeriodDemand),
            PreviousPeriodDemand,
            0
        ) * 100

// Fleet maintenance priority score
FleetMaintenancePriority = 
    SUMX(
        FactFleetTrips,
        (FactFleetTrips[HarshBrakingEvents] * 10) +
        (FactFleetTrips[RapidAccelerationEvents] * 8) +
        IF(FactFleetTrips[MaintenanceAlertTriggered] = TRUE(), 50, 0)
    )

// Supplier reliability impact on warehouse
SupplierImpactScore = 
    CALCULATE(
        AVERAGE(DimSupplierReliability[LeadTimeVarianceDays]) * 
        AVERAGE(DimSupplierReliability[DefectRatePercentage]),
        USERELATIONSHIP(FactWarehouseOperations[SupplierKey], DimSupplierReliability[SupplierKey])
    )
```

### Step 4: Configure Row-Level Security

```dax
// In Power BI Desktop > Modeling > Manage Roles

// Role: Regional Manager
[DimGeography[Region]] = USERNAME()

// Role: Warehouse Supervisor
[DimGeography[LocationID]] IN {
    LOOKUPVALUE(
        UserLocationMapping[LocationID],
        UserLocationMapping[UserEmail],
        USERNAME()
    )
}

// Role: Fleet Manager
[FactFleetTrips[VehicleID]] IN {
    LOOKUPVALUE(
        UserFleetAssignment[VehicleID],
        UserFleetAssignment[UserEmail],
        USERNAME()
    )
}
```

## Key API Integration Patterns

### External Data Source Integration

```sql
-- Create external data source for weather API (using PolyBase)
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST_API,
    LOCATION = '${WEATHER_API_ENDPOINT}',
    CREDENTIAL = WeatherAPICredential
);

-- Create external table for weather data
CREATE EXTERNAL TABLE ExternalWeather (
    Timestamp DATETIME2,
    LocationID VARCHAR(50),
    Temperature DECIMAL(5, 2),
    Condition VARCHAR(50),
    WindSpeed DECIMAL(5, 2)
)
WITH (
    DATA_SOURCE = WeatherAPI,
    FILE_FORMAT = JSONFormat
);

-- Join with fleet data for correlation analysis
SELECT 
    f.TripID,
    f.VehicleID,
    f.AverageSpeedKPH,
    w.Condition AS WeatherCondition,
    w.WindSpeed,
    CASE 
        WHEN w.Condition IN ('Rain', 'Snow', 'Fog') THEN 'Adverse'
        ELSE 'Normal'
    END AS WeatherImpact
FROM FactFleetTrips f
LEFT JOIN ExternalWeather w 
    ON f.OriginGeographyKey = (SELECT GeographyKey FROM DimGeography WHERE LocationID = w.LocationID)
    AND f.StartTimeKey = CAST(FORMAT(w.Timestamp, 'yyyyMMddHHmm') AS INT);
```

### REST API Integration for Telematics

```csharp
// C# example for pulling fleet telematics data
using System;
using System.Net.Http;
using System.Data.SqlClient;
using Newtonsoft.Json;

public class TelematicsIngestion
{
    private readonly string _apiEndpoint = Environment.GetEnvironmentVariable("TELEMATICS_API_ENDPOINT");
    private readonly string _apiKey = Environment.GetEnvironmentVariable("TELEMATICS_API_KEY");
    private readonly string _connectionString = Environment.GetEnvironmentVariable("SQL_CONNECTION_STRING");

    public async Task IngestFleetData()
    {
        using (var client = new HttpClient())
        {
            client.DefaultRequestHeaders.Add("Authorization", $"Bearer {_apiKey}");
            
            var response = await client.GetAsync($"{_apiEndpoint}/fleet/trips?since=last15min");
            var jsonData = await response.Content.ReadAsStringAsync();
            
            var trips = JsonConvert.DeserializeObject<List<FleetTrip>>(jsonData);
            
            using (var connection = new SqlConnection(_connectionString))
            {
                await connection.OpenAsync();
                
                foreach (var trip in trips)
                {
                    using (var command = new SqlCommand("LoadFleetTrips", connection))
                    {
                        command.CommandType = System.Data.CommandType.StoredProcedure;
                        command.Parameters.AddWithValue("@TripID", trip.TripId);
                        command.Parameters.AddWithValue("@VehicleID", trip.VehicleId);
                        command.Parameters.AddWithValue("@StartDateTime", trip.StartTime);
                        command.Parameters.AddWithValue("@FuelConsumed", trip.FuelLiters);
                        // ... additional parameters
                        
                        await command.ExecuteNonQueryAsync();
                    }
                }
            }
        }
    }
}

public class FleetTrip
{
    public string TripId { get; set; }
    public string VehicleId { get; set; }
    public DateTime StartTime { get; set; }
    public decimal FuelLiters { get; set; }
    // ... additional properties
}
```

## Common Analysis Patterns

### Pattern 1: Identify High-Gravity SKUs with Poor Placement

```sql
-- Find products with high gravity scores in wrong zones
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZoneType AS RecommendedZone,
    wo.ZoneAssignment AS CurrentZone,
    AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
    COUNT(*) AS OperationCount
FROM FactWarehouseOperations wo
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
WHERE wo.OperationType = 'Picking'
  AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZoneType, wo.ZoneAssignment
HAVING p.OptimalZoneType <> wo.ZoneAssignment
ORDER BY p.GravityScore DESC, AVG(wo.CycleTimeMinutes) DESC;
```

### Pattern 2: Cross-Fact Route Optimization Analysis

```sql
-- Analyze fleet routes with correlated warehouse dwell time
WITH WarehouseDwell AS (
    SELECT 
        g.LocationID,
        AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS ShipmentCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.OperationType = 'Shipping'
      AND wo.TimeKey >= CAST(FORMAT(DATEADD(DAY, -7, GETDATE()), 'yyyyMMddHHmm') AS INT)
    
