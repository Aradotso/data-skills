---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact star schema for logistics intelligence, fleet optimization, and warehouse analytics
triggers:
  - set up supply chain analytics dashboard
  - implement logistics data warehouse
  - create fleet tracking power bi report
  - build warehouse operations star schema
  - deploy logifleet pulse analytics
  - configure cross-modal supply chain intelligence
  - design multi-fact logistics data model
  - integrate warehouse and fleet telemetry data
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

LogiFleet Pulse is an advanced logistics intelligence platform built on MS SQL Server and Power BI. It provides a multi-fact star schema data warehouse that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer for real-time supply chain analytics and predictive decision-making.

**Core capabilities:**
- Multi-fact star schema linking warehouse, fleet, and cross-dock operations
- Time-phased dimensions with 15-minute granularity
- Real-time Power BI dashboards with role-based access
- Predictive bottleneck detection and fleet triage
- Warehouse gravity zone optimization
- Cross-fact KPI harmonization

## Installation

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Access to data sources (WMS, TMS, telemetry APIs)

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

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeValue DATETIME NOT NULL,
    DateKey INT NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT NOT NULL,
    DayOfWeek INT NOT NULL,
    DayName NVARCHAR(20) NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    MonthName NVARCHAR(20) NOT NULL,
    Quarter INT NOT NULL,
    FiscalPeriod NVARCHAR(10) NOT NULL,
    Year INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, Route Node, Distribution Center
    AddressLine1 NVARCHAR(200),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    PostalCode NVARCHAR(20),
    Country NVARCHAR(100),
    Region NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1
);

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    Subcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,2),
    UnitVolume DECIMAL(10,2),
    IsFragile BIT NOT NULL DEFAULT 0,
    RequiresColdStorage BIT NOT NULL DEFAULT 0,
    ShelfLife INT, -- Days
    UnitValue DECIMAL(12,2),
    GravityScore DECIMAL(5,2), -- Calculated: velocity * value / fragility
    OptimalZone NVARCHAR(50),
    LastRecalculated DATETIME
);

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID NVARCHAR(50) NOT NULL UNIQUE,
    SupplierName NVARCHAR(200) NOT NULL,
    Country NVARCHAR(100),
    LeadTimeMean INT, -- Days
    LeadTimeStdDev DECIMAL(5,2),
    DefectRatePercent DECIMAL(5,2),
    OnTimeDeliveryPercent DECIMAL(5,2),
    ComplianceScore DECIMAL(5,2),
    ReliabilityTier NVARCHAR(20), -- Gold, Silver, Bronze
    LastAssessed DATETIME
);

CREATE TABLE DimFleet (
    FleetKey INT PRIMARY KEY IDENTITY(1,1),
    VehicleID NVARCHAR(50) NOT NULL UNIQUE,
    VehicleType NVARCHAR(50) NOT NULL, -- Truck, Van, Trailer
    Make NVARCHAR(50),
    Model NVARCHAR(50),
    Year INT,
    Capacity DECIMAL(10,2), -- Cubic meters or tons
    FuelType NVARCHAR(30),
    HomeWarehouse NVARCHAR(50),
    AcquisitionDate DATE,
    IsActive BIT NOT NULL DEFAULT 1
);

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    BatchID NVARCHAR(50),
    Quantity INT NOT NULL,
    DwellTimeMinutes INT, -- Time from arrival to next stage
    CycleTimeMinutes INT, -- Time to complete operation
    StorageZone NVARCHAR(50),
    HandlingCost DECIMAL(10,2),
    QualityIssues BIT NOT NULL DEFAULT 0,
    EventTimestamp DATETIME NOT NULL
);

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    FleetKey INT NOT NULL FOREIGN KEY REFERENCES DimFleet(FleetKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    RouteID NVARCHAR(50),
    TripStartTime DATETIME NOT NULL,
    TripEndTime DATETIME,
    PlannedDistanceKm DECIMAL(10,2),
    ActualDistanceKm DECIMAL(10,2),
    DrivingTimeMinutes INT,
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    AverageFuelEconomy DECIMAL(5,2),
    CargoWeightKg DECIMAL(10,2),
    DelayMinutes INT,
    DelayReason NVARCHAR(200),
    WeatherCondition NVARCHAR(50),
    TripCost DECIMAL(12,2)
);

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    Quantity INT NOT NULL,
    TransferTimeMinutes INT,
    DirectTransfer BIT NOT NULL DEFAULT 1,
    EventTimestamp DATETIME NOT NULL
);

-- Create indexes for performance
CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouseOperations_Geography ON FactWarehouseOperations(GeographyKey);
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Fleet ON FactFleetTrips(FleetKey);
CREATE NONCLUSTERED INDEX IX_FactFleetTrips_Route ON FactFleetTrips(OriginGeographyKey, DestinationGeographyKey);
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate time dimension
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATETIME,
    @EndDate DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (
            DateTimeValue, DateKey, TimeOfDay, HourOfDay, DayOfWeek, DayName,
            WeekOfYear, MonthNumber, MonthName, Quarter, FiscalPeriod, Year,
            IsWeekend, IsHoliday
        )
        VALUES (
            @CurrentDateTime,
            CAST(CONVERT(VARCHAR(8), @CurrentDateTime, 112) AS INT),
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            CONCAT('FY', YEAR(@CurrentDateTime), '-Q', DATEPART(QUARTER, @CurrentDateTime)),
            YEAR(@CurrentDateTime),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END,
            0 -- Set holidays separately
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute population for 2 years
EXEC PopulateTimeDimension '2025-01-01 00:00:00', '2026-12-31 23:45:00';
```

### Step 3: Create Data Loading Stored Procedures

```sql
-- Incremental load for warehouse operations
CREATE PROCEDURE LoadWarehouseOperations
    @LastLoadTime DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    -- If no last load time provided, get from control table
    IF @LastLoadTime IS NULL
    BEGIN
        SELECT @LastLoadTime = ISNULL(MAX(EventTimestamp), '1900-01-01')
        FROM FactWarehouseOperations;
    END
    
    -- Insert new records (assumes staging table exists)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, SupplierKey,
        OperationType, BatchID, Quantity, DwellTimeMinutes,
        CycleTimeMinutes, StorageZone, HandlingCost,
        QualityIssues, EventTimestamp
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        s.SupplierKey,
        stg.OperationType,
        stg.BatchID,
        stg.Quantity,
        DATEDIFF(MINUTE, stg.ArrivalTime, stg.CompletionTime) AS DwellTimeMinutes,
        stg.CycleTimeMinutes,
        stg.StorageZone,
        stg.HandlingCost,
        stg.QualityIssues,
        stg.EventTimestamp
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON stg.EventTimestamp >= t.DateTimeValue 
        AND stg.EventTimestamp < DATEADD(MINUTE, 15, t.DateTimeValue)
    INNER JOIN DimGeography g ON stg.WarehouseID = g.LocationID
    INNER JOIN DimProductGravity p ON stg.SKU = p.SKU
    LEFT JOIN DimSupplierReliability s ON stg.SupplierID = s.SupplierID
    WHERE stg.EventTimestamp > @LastLoadTime
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations f
            WHERE f.BatchID = stg.BatchID AND f.EventTimestamp = stg.EventTimestamp
        );
END;
GO

-- Calculate product gravity scores
CREATE PROCEDURE CalculateProductGravity
AS
BEGIN
    SET NOCOUNT ON;
    
    UPDATE DimProductGravity
    SET GravityScore = (
        -- Velocity component (higher is better)
        (SELECT COUNT(*) FROM FactWarehouseOperations f
         WHERE f.ProductKey = DimProductGravity.ProductKey
         AND f.EventTimestamp >= DATEADD(DAY, -30, GETDATE())) * 0.4
        +
        -- Value component (normalized)
        (UnitValue / NULLIF((SELECT MAX(UnitValue) FROM DimProductGravity), 0)) * 100 * 0.4
        -
        -- Fragility penalty
        (CASE WHEN IsFragile = 1 THEN 20 ELSE 0 END) * 0.2
    ),
    OptimalZone = CASE 
        WHEN GravityScore >= 80 THEN 'A-High-Velocity'
        WHEN GravityScore >= 60 THEN 'B-Medium-Velocity'
        WHEN GravityScore >= 40 THEN 'C-Slow-Velocity'
        ELSE 'D-Slow-Moving'
    END,
    LastRecalculated = GETDATE();
END;
GO
```

### Step 4: Configure Power BI Connection

Create a `config_powerbi.json` file:

```json
{
  "sql_server": {
    "server": "YOUR_SQL_SERVER_INSTANCE",
    "database": "LogiFleetPulse",
    "authentication": "windows"
  },
  "refresh_schedule": {
    "interval_minutes": 15,
    "enabled": true
  },
  "security": {
    "row_level_enabled": true,
    "role_table": "DimUserRoles"
  }
}
```

### Step 5: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter connection parameters:
   - Server: Your SQL Server instance
   - Database: LogiFleetPulse
5. Click Load to import the pre-configured data model

## Key Queries and Analytics Patterns

### Cross-Fact KPI: Fleet Efficiency vs. Warehouse Dwell Time

```sql
-- Analyze correlation between warehouse dwell time and fleet delays
SELECT 
    g.LocationName AS Warehouse,
    p.Category AS ProductCategory,
    AVG(wo.DwellTimeMinutes) AS AvgDwellTimeMin,
    AVG(ft.DelayMinutes) AS AvgDeliveryDelayMin,
    COUNT(DISTINCT ft.TripKey) AS TotalTrips,
    SUM(ft.TripCost) AS TotalTripCost,
    CASE 
        WHEN AVG(wo.DwellTimeMinutes) > 72 * 60 THEN 'High Risk'
        WHEN AVG(wo.DwellTimeMinutes) > 48 * 60 THEN 'Medium Risk'
        ELSE 'Normal'
    END AS DwellRiskLevel
FROM FactWarehouseOperations wo
INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
INNER JOIN FactFleetTrips ft ON g.GeographyKey = ft.OriginGeographyKey
WHERE wo.EventTimestamp >= DATEADD(MONTH, -3, GETDATE())
    AND wo.OperationType = 'Picking'
GROUP BY g.LocationName, p.Category
HAVING AVG(wo.DwellTimeMinutes) > 24 * 60 -- Focus on items dwelling > 24 hours
ORDER BY AvgDwellTimeMin DESC;
```

### Predictive Bottleneck Detection

```sql
-- Identify warehouses with increasing operational cycle times
WITH CycleTimeTrends AS (
    SELECT 
        g.LocationName,
        DATEPART(WEEK, wo.EventTimestamp) AS WeekNumber,
        AVG(wo.CycleTimeMinutes) AS AvgCycleTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations wo
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.EventTimestamp >= DATEADD(MONTH, -2, GETDATE())
    GROUP BY g.LocationName, DATEPART(WEEK, wo.EventTimestamp)
),
BottleneckScore AS (
    SELECT 
        LocationName,
        AvgCycleTime,
        LAG(AvgCycleTime) OVER (PARTITION BY LocationName ORDER BY WeekNumber) AS PrevWeekCycleTime,
        ((AvgCycleTime - LAG(AvgCycleTime) OVER (PARTITION BY LocationName ORDER BY WeekNumber)) 
         / NULLIF(LAG(AvgCycleTime) OVER (PARTITION BY LocationName ORDER BY WeekNumber), 0)) * 100 AS PercentIncrease
    FROM CycleTimeTrends
)
SELECT 
    LocationName,
    AvgCycleTime,
    PrevWeekCycleTime,
    PercentIncrease,
    CASE 
        WHEN PercentIncrease > 20 THEN 'Critical Alert'
        WHEN PercentIncrease > 10 THEN 'Warning'
        ELSE 'Normal'
    END AS BottleneckStatus
FROM BottleneckScore
WHERE PercentIncrease IS NOT NULL
ORDER BY PercentIncrease DESC;
```

### Fleet Maintenance Triage

```sql
-- Prioritize fleet maintenance based on vehicle usage and cargo value
SELECT 
    f.VehicleID,
    f.VehicleType,
    COUNT(ft.TripKey) AS TotalTrips,
    SUM(ft.ActualDistanceKm) AS TotalDistanceKm,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTimeMin,
    AVG(ft.FuelConsumedLiters / NULLIF(ft.ActualDistanceKm, 0)) AS AvgFuelPerKm,
    SUM(ft.CargoWeightKg) AS TotalCargoKg,
    SUM(ft.TripCost) AS TotalRevenue,
    CASE 
        WHEN AVG(ft.IdleTimeMinutes) > 45 THEN 'Engine Diagnostics Needed'
        WHEN AVG(ft.FuelConsumedLiters / NULLIF(ft.ActualDistanceKm, 0)) > 0.35 THEN 'Fuel System Check'
        WHEN SUM(ft.ActualDistanceKm) > 50000 THEN 'Scheduled Maintenance Due'
        ELSE 'Normal Operation'
    END AS MaintenanceRecommendation,
    ROW_NUMBER() OVER (ORDER BY SUM(ft.TripCost) DESC) AS RevenuePriority
FROM FactFleetTrips ft
INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
WHERE ft.TripStartTime >= DATEADD(MONTH, -6, GETDATE())
GROUP BY f.VehicleID, f.VehicleType
HAVING COUNT(ft.TripKey) > 10
ORDER BY RevenuePriority, TotalDistanceKm DESC;
```

### Warehouse Gravity Zone Optimization

```sql
-- Recommend storage zone reassignments based on updated gravity scores
SELECT 
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone AS RecommendedZone,
    wo.StorageZone AS CurrentZone,
    COUNT(*) AS RecentPicks,
    AVG(wo.CycleTimeMinutes) AS AvgPickTimeMin,
    CASE 
        WHEN p.OptimalZone != wo.StorageZone THEN 'Reassignment Recommended'
        ELSE 'Current Assignment Optimal'
    END AS Action
FROM DimProductGravity p
INNER JOIN FactWarehouseOperations wo ON p.ProductKey = wo.ProductKey
WHERE wo.EventTimestamp >= DATEADD(DAY, -30, GETDATE())
    AND wo.OperationType = 'Picking'
    AND p.OptimalZone != wo.StorageZone
GROUP BY p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, wo.StorageZone
HAVING COUNT(*) > 5
ORDER BY RecentPicks DESC;
```

## Power BI DAX Measures

### Key Performance Indicators

```dax
// Total Fleet Utilization %
Fleet Utilization % = 
VAR TotalAvailableTime = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[DrivingTimeMinutes] + FactFleetTrips[IdleTimeMinutes]
    )
VAR ActiveDrivingTime = SUM(FactFleetTrips[DrivingTimeMinutes])
RETURN
    DIVIDE(ActiveDrivingTime, TotalAvailableTime, 0) * 100

// Average Dwell Time by Product Gravity
Avg Dwell Time by Gravity = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    ALLEXCEPT(DimProductGravity, DimProductGravity[OptimalZone])
)

// Cross-Dock Efficiency (Direct Transfer %)
Cross-Dock Efficiency % = 
VAR DirectTransfers = CALCULATE(COUNTROWS(FactCrossDock), FactCrossDock[DirectTransfer] = TRUE())
VAR TotalTransfers = COUNTROWS(FactCrossDock)
RETURN
    DIVIDE(DirectTransfers, TotalTransfers, 0) * 100

// Fuel Cost per Kilometer
Fuel Cost per KM = 
VAR TotalFuelCost = SUMX(FactFleetTrips, FactFleetTrips[FuelConsumedLiters] * 1.45) // Assuming $1.45/liter
VAR TotalDistance = SUM(FactFleetTrips[ActualDistanceKm])
RETURN
    DIVIDE(TotalFuelCost, TotalDistance, 0)

// Supplier Reliability Index
Supplier Reliability Index = 
CALCULATE(
    AVERAGE(DimSupplierReliability[OnTimeDeliveryPercent]) * 0.4 +
    (100 - AVERAGE(DimSupplierReliability[DefectRatePercent])) * 0.3 +
    AVERAGE(DimSupplierReliability[ComplianceScore]) * 0.3
)

// Time-Based Bottleneck Score
Bottleneck Score = 
VAR CurrentAvgCycleTime = CALCULATE(AVERAGE(FactWarehouseOperations[CycleTimeMinutes]))
VAR PreviousAvgCycleTime = 
    CALCULATE(
        AVERAGE(FactWarehouseOperations[CycleTimeMinutes]),
        DATEADD(DimTime[DateTimeValue], -7, DAY)
    )
RETURN
    IF(
        ISBLANK(PreviousAvgCycleTime),
        0,
        DIVIDE(CurrentAvgCycleTime - PreviousAvgCycleTime, PreviousAvgCycleTime, 0) * 100
    )
```

## Configuration Best Practices

### Row-Level Security Setup

```sql
-- Create user roles table
CREATE TABLE DimUserRoles (
    UserID NVARCHAR(100) PRIMARY KEY,
    UserName NVARCHAR(200),
    Role NVARCHAR(50), -- Executive, Supervisor, Operator
    RegionAccess NVARCHAR(MAX), -- Comma-separated list
    WarehouseAccess NVARCHAR(MAX) -- Comma-separated list
);

-- Insert sample users
INSERT INTO DimUserRoles VALUES 
    ('user@domain.com', 'John Smith', 'Executive', 'North America,Europe', NULL),
    ('supervisor@domain.com', 'Jane Doe', 'Supervisor', 'North America', 'WH-001,WH-002'),
    ('operator@domain.com', 'Bob Johnson', 'Operator', 'North America', 'WH-001');
```

In Power BI, create RLS roles:

```dax
// Role: Regional Access
[Region] IN VALUES(DimUserRoles[RegionAccess])

// Role: Warehouse Access
[LocationID] IN VALUES(DimUserRoles[WarehouseAccess])
```

### Automated Alerting

```sql
-- Create alerting stored procedure
CREATE PROCEDURE CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Check for excessive fleet idle time
    INSERT INTO AlertLog (AlertType, AlertMessage, Severity, CreatedAt)
    SELECT 
        'Fleet Idle Time Exceeded',
        CONCAT('Vehicle ', f.VehicleID, ' idle time: ', AVG(ft.IdleTimeMinutes), ' minutes'),
        'High',
        GETDATE()
    FROM FactFleetTrips ft
    INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
    WHERE ft.TripStartTime >= DATEADD(HOUR, -24, GETDATE())
    GROUP BY f.VehicleID
    HAVING AVG(ft.IdleTimeMinutes) > 60;
    
    -- Check for warehouse dwell time breaches
    INSERT INTO AlertLog (AlertType, AlertMessage, Severity, CreatedAt)
    SELECT 
        'Warehouse Dwell Time Critical',
        CONCAT('SKU ', p.SKU, ' at ', g.LocationName, ' dwell time: ', 
               AVG(wo.DwellTimeMinutes) / 60, ' hours'),
        'Critical',
        GETDATE()
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity p ON wo.ProductKey = p.ProductKey
    INNER JOIN DimGeography g ON wo.GeographyKey = g.GeographyKey
    WHERE wo.EventTimestamp >= DATEADD(HOUR, -12, GETDATE())
        AND p.RequiresColdStorage = 1
    GROUP BY p.SKU, g.LocationName
    HAVING AVG(wo.DwellTimeMinutes) > 240; -- 4 hours
END;
GO

-- Schedule this procedure via SQL Server Agent (runs every 15 minutes)
```

## Integration Patterns

### External API Data Ingestion (Weather)

```sql
-- Create staging table for weather data
CREATE TABLE StagingWeatherData (
    LocationID NVARCHAR(50),
    Timestamp DATETIME,
    Temperature DECIMAL(5,2),
    Precipitation NVARCHAR(50),
    WindSpeed DECIMAL(5,2),
    Visibility NVARCHAR(50),
    LoadedAt DATETIME DEFAULT GETDATE()
);

-- Python script to load weather data (scheduled task)
```

```python
import pyodbc
import requests
import os
from datetime import datetime

# Connection setup
conn = pyodbc.connect(
    f"DRIVER={{ODBC Driver 17 for SQL Server}};"
    f"SERVER={os.getenv('SQL_SERVER')};"
    f"DATABASE=LogiFleetPulse;"
    f"Trusted_Connection=yes;"
)
cursor = conn.cursor()

# Fetch weather from external API
api_key = os.getenv('WEATHER_API_KEY')
locations = ['WH-001', 'WH-002', 'WH-003']

for location_id in locations:
    # Get coordinates from database
    cursor.execute(f"SELECT Latitude, Longitude FROM DimGeography WHERE LocationID = '{location_id}'")
    lat, lon = cursor.fetchone()
    
    # Call weather API
    url = f"https://api.weatherapi.com/v1/current.json?key={api_key}&q={lat},{lon}"
    response = requests.get(url)
    data = response.json()
    
    # Insert into staging
    cursor.execute("""
        INSERT INTO StagingWeatherData (LocationID, Timestamp, Temperature, Precipitation, WindSpeed, Visibility)
        VALUES (?, ?, ?, ?, ?, ?)
    """, location_id, datetime.now(), data['current']['temp_c'], 
        data['current']['condition']['text'], data['current']['wind_kph'], data['current']['vis_km'])

conn.commit()
cursor.close()
conn.close()
```

### REST API Exposure (Optional)

```csharp
// ASP.NET Core API endpoint to expose real-time KPIs
using Microsoft.AspNetCore.Mvc;
using System.Data.SqlClient;

[ApiController]
[Route("api/[controller]")]
public class LogisticsKPIController : ControllerBase
{
    private readonly string _connectionString = Environment.GetEnvironmentVariable("SQL_CONNECTION_STRING");

    [HttpGet("fleet-utilization")]
    public async Task<IActionResult> GetFleetUtilization([FromQuery] DateTime? startDate = null)
    {
        startDate ??= DateTime.Now.AddDays(-7);
        
        using var conn = new SqlConnection(_connectionString);
        await conn.OpenAsync();
        
        var cmd = new SqlCommand(@"
            SELECT 
                f.VehicleID,
                SUM(ft.DrivingTimeMinutes) AS TotalDrivingMin,
                SUM(ft.IdleTimeMinutes) AS TotalIdleMin,
                (SUM(ft.DrivingTimeMinutes) * 1.0 / 
                 NULLIF(SUM(ft.DrivingTimeMinutes + ft.IdleTimeMinutes), 0)) * 100 AS UtilizationPercent
            FROM FactFleetTrips ft
            INNER JOIN DimFleet f ON ft.FleetKey = f.FleetKey
            WHERE ft.TripStartTime >= @StartDate
            GROUP BY f.VehicleID
        ", conn);
        
        cmd.Parameters.AddWithValue("@StartDate", startDate);
        
        var results
