---
name: logifleet-pulse-supply-chain-analytics
description: Multi-modal logistics intelligence engine using MS SQL Server star schema and Power BI for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure multi-fact star schema for fleet and warehouse
  - implement cross-modal supply chain analytics
  - create logistics kpi dashboard with warehouse gravity zones
  - build real-time fleet optimization analytics
  - integrate warehouse and fleet telemetry data model
  - design supply chain data warehouse architecture
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and supply chain data into a multi-fact star schema data warehouse. Built on MS SQL Server with Power BI visualization, it enables:

- **Cross-fact KPI harmonization** across warehouse, fleet, and inventory domains
- **Real-time operational dashboards** with 15-minute refresh cycles
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Warehouse Gravity Zones™** for optimal storage layout based on velocity, value, and fragility
- **Adaptive fleet triage** with maintenance prioritization by revenue impact
- **Multi-dimensional analysis** linking dwell time, route efficiency, and demand patterns

The platform uses a custom star schema with time-phased dimensions, bridge tables for many-to-many relationships, and role-playing dimensions for comprehensive supply chain visibility.

## Installation & Deployment

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Appropriate permissions: `db_owner` or `db_ddladmin` + `db_datawriter`

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Create dimension tables
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    DateTimeStamp DATETIME2 NOT NULL,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayOfMonth TINYINT,
    WeekOfYear TINYINT,
    MonthOfYear TINYINT,
    QuarterOfYear TINYINT,
    FiscalYear INT,
    FiscalPeriod INT,
    IsWeekend BIT,
    IsHoliday BIT
)
GO

CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName NVARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'RouteNode', 'CrossDock'
    City NVARCHAR(100),
    Region NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    IsActive BIT DEFAULT 1
)
GO

CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName NVARCHAR(300),
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    RequiresRefrigeration BIT,
    GravityScore DECIMAL(5,2), -- Computed: velocity × value weight × (1/fragility factor)
    VelocityTier VARCHAR(20), -- 'Fast', 'Medium', 'Slow'
    ValueTier VARCHAR(20) -- 'High', 'Medium', 'Low'
)
GO

CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) UNIQUE NOT NULL,
    SupplierName NVARCHAR(200),
    LeadTimeMean DECIMAL(8,2),
    LeadTimeStdDev DECIMAL(8,2),
    DefectRate DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2),
    ReliabilityGrade VARCHAR(10)
)
GO

-- Create fact tables
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OperationID VARCHAR(100),
    DwellTimeMinutes INT,
    PickRateUnitsPerHour DECIMAL(10,2),
    PackingTimeMinutes INT,
    StorageZone VARCHAR(50),
    GravityZoneAssigned VARCHAR(50),
    EmployeeID VARCHAR(50),
    EquipmentID VARCHAR(50)
)
GO

CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID VARCHAR(100) UNIQUE NOT NULL,
    VehicleID VARCHAR(50),
    DriverID VARCHAR(50),
    DepartureTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    ArrivalTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    AverageSpeedKPH DECIMAL(6,2),
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT
)
GO

CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    ArrivalTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    DepartureTimeKey INT FOREIGN KEY REFERENCES DimTime(TimeKey),
    CrossDockGeographyKey INT FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripID VARCHAR(100),
    OutboundTripID VARCHAR(100),
    DwellTimeMinutes INT,
    UnitsTransferred INT
)
GO

-- Create indexes for query performance
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Departure ON FactFleetTrips(DepartureTimeKey)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
GO

-- Create view for cross-fact analysis
CREATE VIEW vw_CrossFactLogistics AS
SELECT 
    dt.DateTimeStamp,
    dt.HourOfDay,
    dt.DayOfWeek,
    dg.LocationName AS WarehouseLocation,
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    wo.DwellTimeMinutes AS WarehouseDwellMin,
    wo.OperationType,
    ft.TripID,
    ft.DistanceKM,
    ft.FuelConsumedLiters,
    ft.IdleTimeMinutes AS FleetIdleMin,
    CASE 
        WHEN wo.DwellTimeMinutes > 72*60 THEN 'High Risk'
        WHEN wo.DwellTimeMinutes > 48*60 THEN 'Medium Risk'
        ELSE 'Normal'
    END AS DwellRiskCategory
FROM FactWarehouseOperations wo
INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON wo.GeographyKey = dg.GeographyKey
INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
LEFT JOIN FactFleetTrips ft ON dg.GeographyKey = ft.OriginGeographyKey 
    AND DATEDIFF(hour, dt.DateTimeStamp, (SELECT DateTimeStamp FROM DimTime WHERE TimeKey = ft.DepartureTimeKey)) <= 24
GO
```

### Step 2: Configure Data Sources

Create a configuration file `config.json` for ETL connections:

```json
{
  "sql_server": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "username": "${SQL_SERVER_USER}",
    "password": "${SQL_SERVER_PASSWORD}",
    "port": 1433
  },
  "data_sources": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "api_key": "${WMS_API_KEY}"
    },
    "telematics_api": {
      "endpoint": "${TELEMATICS_API_ENDPOINT}",
      "api_key": "${TELEMATICS_API_KEY}"
    },
    "weather_api": {
      "endpoint": "https://api.openweathermap.org/data/2.5",
      "api_key": "${WEATHER_API_KEY}"
    }
  },
  "refresh_intervals": {
    "warehouse_operations": 15,
    "fleet_trips": 15,
    "dimensions": 1440
  }
}
```

### Step 3: Create ETL Stored Procedures

```sql
-- Stored procedure for incremental warehouse operations load
CREATE PROCEDURE usp_LoadWarehouseOperations
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType, 
        OperationID, DwellTimeMinutes, PickRateUnitsPerHour,
        PackingTimeMinutes, StorageZone, GravityZoneAssigned
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        stg.OperationType,
        stg.OperationID,
        stg.DwellTimeMinutes,
        stg.PickRateUnitsPerHour,
        stg.PackingTimeMinutes,
        stg.StorageZone,
        CASE 
            WHEN dp.GravityScore > 8.0 THEN 'GravityZone_A'
            WHEN dp.GravityScore > 5.0 THEN 'GravityZone_B'
            ELSE 'GravityZone_C'
        END AS GravityZoneAssigned
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime dt ON CAST(stg.OperationTimestamp AS DATE) = CAST(dt.DateTimeStamp AS DATE)
        AND DATEPART(HOUR, stg.OperationTimestamp) = dt.HourOfDay
    INNER JOIN DimGeography dg ON stg.LocationID = dg.LocationID
    INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU
    WHERE stg.OperationTimestamp > @LastLoadTimestamp
        AND NOT EXISTS (
            SELECT 1 FROM FactWarehouseOperations 
            WHERE OperationID = stg.OperationID
        );
        
    RETURN @@ROWCOUNT;
END
GO

-- Stored procedure for fleet trip analysis with KPI calculation
CREATE PROCEDURE usp_CalculateFleetKPIs
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        ft.VehicleID,
        COUNT(DISTINCT ft.TripID) AS TotalTrips,
        SUM(ft.DistanceKM) AS TotalDistanceKM,
        SUM(ft.FuelConsumedLiters) AS TotalFuelLiters,
        AVG(ft.FuelConsumedLiters / NULLIF(ft.DistanceKM, 0)) AS AvgFuelEfficiency,
        SUM(ft.IdleTimeMinutes) AS TotalIdleMinutes,
        SUM(ft.IdleTimeMinutes) * 1.0 / NULLIF(SUM(DATEDIFF(MINUTE, dt_dep.DateTimeStamp, dt_arr.DateTimeStamp)), 0) AS IdleTimePercentage,
        AVG(ft.TrafficDelayMinutes) AS AvgTrafficDelayMinutes,
        SUM(CASE WHEN ft.TrafficDelayMinutes > 30 THEN 1 ELSE 0 END) AS HighDelayTrips
    FROM FactFleetTrips ft
    INNER JOIN DimTime dt_dep ON ft.DepartureTimeKey = dt_dep.TimeKey
    INNER JOIN DimTime dt_arr ON ft.ArrivalTimeKey = dt_arr.TimeKey
    WHERE CAST(dt_dep.DateTimeStamp AS DATE) BETWEEN @StartDate AND @EndDate
    GROUP BY ft.VehicleID
    ORDER BY TotalDistanceKM DESC;
END
GO

-- Stored procedure for gravity zone optimization recommendations
CREATE PROCEDURE usp_RecommendGravityZoneChanges
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH CurrentAssignments AS (
        SELECT 
            dp.SKU,
            dp.ProductName,
            dp.GravityScore,
            wo.GravityZoneAssigned AS CurrentZone,
            COUNT(*) AS OperationCount,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime,
            CASE 
                WHEN dp.GravityScore > 8.0 THEN 'GravityZone_A'
                WHEN dp.GravityScore > 5.0 THEN 'GravityZone_B'
                ELSE 'GravityZone_C'
            END AS RecommendedZone
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
        WHERE wo.TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE DateTimeStamp >= DATEADD(DAY, -30, GETDATE())
        )
        GROUP BY dp.SKU, dp.ProductName, dp.GravityScore, wo.GravityZoneAssigned
    )
    SELECT 
        SKU,
        ProductName,
        GravityScore,
        CurrentZone,
        RecommendedZone,
        OperationCount,
        AvgDwellTime,
        CASE 
            WHEN CurrentZone != RecommendedZone THEN 'Reassignment Needed'
            ELSE 'Optimal'
        END AS RecommendationStatus
    FROM CurrentAssignments
    WHERE CurrentZone != RecommendedZone
    ORDER BY OperationCount DESC, GravityScore DESC;
END
GO
```

## Power BI Setup

### Step 1: Connect Power BI to SQL Server

1. Open Power BI Desktop
2. Get Data → SQL Server
3. Server: `${SQL_SERVER_HOST}`
4. Database: `LogiFleetPulse`
5. Data Connectivity mode: **DirectQuery** (for real-time) or **Import** (for performance)
6. Select tables: All Dim* and Fact* tables plus `vw_CrossFactLogistics`

### Step 2: Define Relationships

Power BI should auto-detect relationships. Verify these exist:

```
DimTime (TimeKey) → FactWarehouseOperations (TimeKey)
DimTime (TimeKey) → FactFleetTrips (DepartureTimeKey) [role-playing]
DimTime (TimeKey) → FactFleetTrips (ArrivalTimeKey) [role-playing]
DimGeography (GeographyKey) → FactWarehouseOperations (GeographyKey)
DimGeography (GeographyKey) → FactFleetTrips (OriginGeographyKey) [role-playing]
DimGeography (GeographyKey) → FactFleetTrips (DestinationGeographyKey) [role-playing]
DimProductGravity (ProductKey) → FactWarehouseOperations (ProductKey)
```

### Step 3: Create DAX Measures

```dax
// Total Warehouse Dwell Time (hours)
TotalDwellTimeHours = 
    SUM(FactWarehouseOperations[DwellTimeMinutes]) / 60

// Fleet Idle Percentage
FleetIdlePercentage = 
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[IdleTimeMinutes]) + 
        SUMX(
            FactFleetTrips,
            DATEDIFF(
                RELATED(DimTime[DateTimeStamp]),
                RELATED(DimTime[DateTimeStamp]),
                MINUTE
            )
        )
    ) * 100

// Average Gravity Score by Zone
AvgGravityScoreByZone = 
    AVERAGEX(
        SUMMARIZE(
            FactWarehouseOperations,
            FactWarehouseOperations[GravityZoneAssigned],
            DimProductGravity[GravityScore]
        ),
        DimProductGravity[GravityScore]
    )

// Cross-Fact: Dwell Time Impact on Fleet Efficiency
DwellImpactOnFleet = 
    VAR HighDwellProducts = 
        CALCULATETABLE(
            VALUES(DimProductGravity[SKU]),
            FactWarehouseOperations[DwellTimeMinutes] > 72 * 60
        )
    VAR FleetTripsWithHighDwell = 
        CALCULATE(
            COUNTROWS(FactFleetTrips),
            FILTER(
                FactWarehouseOperations,
                DimProductGravity[SKU] IN HighDwellProducts
            )
        )
    RETURN
        DIVIDE(FleetTripsWithHighDwell, COUNTROWS(FactFleetTrips)) * 100

// Bottleneck Risk Score
BottleneckRiskScore = 
    VAR DwellWeight = 0.4
    VAR IdleWeight = 0.3
    VAR DelayWeight = 0.3
    VAR NormalizedDwell = 
        DIVIDE(
            AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
            MAX(FactWarehouseOperations[DwellTimeMinutes])
        )
    VAR NormalizedIdle = 
        DIVIDE(
            AVERAGE(FactFleetTrips[IdleTimeMinutes]),
            MAX(FactFleetTrips[IdleTimeMinutes])
        )
    VAR NormalizedDelay = 
        DIVIDE(
            AVERAGE(FactFleetTrips[TrafficDelayMinutes]),
            MAX(FactFleetTrips[TrafficDelayMinutes])
        )
    RETURN
        (NormalizedDwell * DwellWeight + 
         NormalizedIdle * IdleWeight + 
         NormalizedDelay * DelayWeight) * 100

// Fuel Cost Estimation (assuming cost per liter)
EstimatedFuelCost = 
    SUMX(
        FactFleetTrips,
        FactFleetTrips[FuelConsumedLiters] * 1.50  -- Replace with actual fuel price
    )
```

### Step 4: Build Key Visualizations

**Dashboard 1: Executive Overview**
- KPI Cards: Total Trips, Total Operations, Avg Dwell Time, Fleet Idle %
- Line Chart: Daily Operations Count (Time.DateTimeStamp × Count of Operations)
- Stacked Bar: Fleet Fuel Consumption by Vehicle (VehicleID × FuelConsumedLiters)
- Map: Geographic distribution of warehouses (Geography.Latitude/Longitude)

**Dashboard 2: Warehouse Gravity Zones**
- Matrix: SKU × GravityZoneAssigned × AvgDwellTime
- Heatmap: StorageZone × OperationType × Count
- Scatter Plot: GravityScore (X) × DwellTimeMinutes (Y), sized by OperationCount

**Dashboard 3: Fleet Efficiency**
- Clustered Column: Vehicle × IdleTimePercentage × TrafficDelayMinutes
- Gauge: Fleet Idle % against target threshold (15%)
- Table: Top 10 routes by fuel efficiency with origin/destination

## Common Patterns & Workflows

### Pattern 1: Daily ETL Execution

```sql
-- SQL Agent Job or scheduled task to run daily at 02:00
DECLARE @LastLoad DATETIME2
SELECT @LastLoad = MAX(LoadTimestamp) FROM ETLControlTable WHERE TableName = 'FactWarehouseOperations'

EXEC usp_LoadWarehouseOperations @LastLoadTimestamp = @LastLoad

UPDATE ETLControlTable 
SET LoadTimestamp = GETDATE(), RowsLoaded = @@ROWCOUNT
WHERE TableName = 'FactWarehouseOperations'
```

### Pattern 2: Real-Time Alert Setup

```sql
-- Create alert for high dwell time
CREATE PROCEDURE usp_AlertHighDwellTime
AS
BEGIN
    DECLARE @AlertThresholdMinutes INT = 4320 -- 72 hours
    
    SELECT 
        wo.OperationID,
        dp.SKU,
        dp.ProductName,
        wo.DwellTimeMinutes,
        dg.LocationName,
        dt.DateTimeStamp
    INTO #HighDwellAlerts
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    INNER JOIN DimGeography dg ON wo.GeographyKey = dg.GeographyKey
    INNER JOIN DimTime dt ON wo.TimeKey = dt.TimeKey
    WHERE wo.DwellTimeMinutes > @AlertThresholdMinutes
        AND dt.DateTimeStamp >= DATEADD(HOUR, -24, GETDATE())
    
    IF EXISTS (SELECT 1 FROM #HighDwellAlerts)
    BEGIN
        -- Send email using Database Mail
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleet Alerts',
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'High Dwell Time Alert - LogiFleet Pulse',
            @body = 'Critical dwell time threshold exceeded. See attached details.',
            @query = 'SELECT * FROM #HighDwellAlerts',
            @attach_query_result_as_file = 1,
            @query_attachment_filename = 'HighDwellAlerts.csv'
    END
    
    DROP TABLE #HighDwellAlerts
END
GO
```

### Pattern 3: Gravity Zone Recalculation

```sql
-- Monthly batch job to update gravity scores based on recent activity
CREATE PROCEDURE usp_RecalculateGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH RecentActivity AS (
        SELECT 
            dp.ProductKey,
            dp.SKU,
            COUNT(*) AS OperationFrequency,
            AVG(wo.PickRateUnitsPerHour) AS AvgPickRate,
            AVG(wo.DwellTimeMinutes) AS AvgDwellTime
        FROM FactWarehouseOperations wo
        INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
        WHERE wo.TimeKey IN (
            SELECT TimeKey FROM DimTime 
            WHERE DateTimeStamp >= DATEADD(MONTH, -1, GETDATE())
        )
        GROUP BY dp.ProductKey, dp.SKU
    ),
    NewGravityScores AS (
        SELECT 
            ra.ProductKey,
            ra.SKU,
            -- Gravity = (frequency_percentile × 0.5) + (pick_rate_percentile × 0.3) + (1/dwell_percentile × 0.2) × 10
            (
                PERCENT_RANK() OVER (ORDER BY ra.OperationFrequency) * 0.5 +
                PERCENT_RANK() OVER (ORDER BY ra.AvgPickRate) * 0.3 +
                (1 - PERCENT_RANK() OVER (ORDER BY ra.AvgDwellTime)) * 0.2
            ) * 10 AS NewGravityScore
        FROM RecentActivity ra
    )
    UPDATE dp
    SET 
        dp.GravityScore = ngs.NewGravityScore,
        dp.VelocityTier = CASE 
            WHEN ngs.NewGravityScore > 7.5 THEN 'Fast'
            WHEN ngs.NewGravityScore > 4.5 THEN 'Medium'
            ELSE 'Slow'
        END
    FROM DimProductGravity dp
    INNER JOIN NewGravityScores ngs ON dp.ProductKey = ngs.ProductKey
END
GO
```

### Pattern 4: Cross-Fact Analysis Query

```sql
-- Find products with high warehouse dwell AND high associated fleet idle time
SELECT 
    dp.SKU,
    dp.ProductName,
    dp.GravityScore,
    AVG(wo.DwellTimeMinutes) AS AvgWarehouseDwellMin,
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMin,
    COUNT(DISTINCT wo.OperationID) AS WarehouseOps,
    COUNT(DISTINCT ft.TripID) AS FleetTrips,
    (AVG(wo.DwellTimeMinutes) * 0.6 + AVG(ft.IdleTimeMinutes) * 0.4) AS CompositeInefficiencyScore
FROM DimProductGravity dp
INNER JOIN FactWarehouseOperations wo ON dp.ProductKey = wo.ProductKey
INNER JOIN DimTime dt_wo ON wo.TimeKey = dt_wo.TimeKey
LEFT JOIN FactFleetTrips ft ON wo.GeographyKey = ft.OriginGeographyKey
INNER JOIN DimTime dt_ft ON ft.DepartureTimeKey = dt_ft.TimeKey
WHERE dt_wo.DateTimeStamp >= DATEADD(MONTH, -3, GETDATE())
    AND ABS(DATEDIFF(HOUR, dt_wo.DateTimeStamp, dt_ft.DateTimeStamp)) <= 48
GROUP BY dp.SKU, dp.ProductName, dp.GravityScore
HAVING AVG(wo.DwellTimeMinutes) > 2880 -- > 48 hours
    AND AVG(ft.IdleTimeMinutes) > 45
ORDER BY CompositeInefficiencyScore DESC
```

## Configuration & Customization

### Row-Level Security (RLS) Setup

```sql
-- Create security table
CREATE TABLE SecurityRoles (
    UserEmail VARCHAR(200),
    RoleName VARCHAR(50),
    GeographyAccess VARCHAR(100), -- 'All', 'North America', 'Warehouse-01', etc.
    ProductAccess VARCHAR(100) -- 'All', 'Electronics', 'Perishables', etc.
)
GO

-- Create RLS predicate function
CREATE FUNCTION dbo.fn_SecurityPredicate(@GeographyKey INT, @ProductKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    FROM dbo.SecurityRoles sr
    INNER JOIN dbo.DimGeography dg ON sr.GeographyAccess = dg.Region OR sr.GeographyAccess = 'All'
    INNER JOIN dbo.DimProductGravity dp ON sr.ProductAccess = dp.ProductCategory OR sr.ProductAccess = 'All'
    WHERE sr.UserEmail = USER_NAME()
        AND dg.GeographyKey = @GeographyKey
        AND dp.ProductKey = @ProductKey
)
GO

-- Apply security policy
CREATE SECURITY POLICY WarehouseSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(GeographyKey, ProductKey) ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(OriginGeographyKey, NULL) ON dbo.FactFleetTrips
WITH (STATE = ON)
GO
```

### Custom Time Buckets for 15-Minute Granularity

```sql
-- Populate DimTime with 15-minute intervals
DECLARE @StartDate DATETIME2 = '2024-01-01 00:00:00'
DECLARE @EndDate DATETIME2 = '2027-12-31 23:45:00'
DECLARE @CurrentDate DATETIME2 = @StartDate

WHILE @CurrentDate <= @EndDate
BEGIN
    INSERT INTO DimTime (
        DateTimeStamp, HourOfDay, DayOfWeek, DayOfMonth, 
        WeekOfYear, MonthOfYear, QuarterOfYear, FiscalYear, 
        FiscalPeriod, IsWeekend, IsHoliday
    )
    VALUES (
        @CurrentDate,
        DATEPART(HOUR, @CurrentDate),
        DATEPART(WEEKDAY, @CurrentDate),
        DATEPART(DAY, @CurrentDate),
        DATEPART(WEEK, @CurrentDate),
        DATEPART(MONTH, @CurrentDate),
        DATEPART(QUARTER, @CurrentDate),
        YEAR(@CurrentDate),
        (DATEPART(MONTH, @CurrentDate) - 1) / 3 + 1,
        CASE WHEN DATEPART(WEEKDAY, @CurrentDate) IN (1, 7) THEN 1 ELSE 0 END,
        0 -- Update with holiday logic
    )
    
    SET @CurrentDate = DATEADD(MINUTE, 15, @CurrentDate)
END
GO
```

## Troubleshooting

### Issue: Power BI refresh fails with timeout

**Solution**: Switch from DirectQuery to Import mode, or optimize queries with indexed views:

```sql
CREATE VIEW vw_WarehouseOperationsSummary
WITH SCHEMABINDING
AS
SELECT 
    wo.GeographyKey,
    wo.ProductKey,
    wo.TimeKey,
    COUNT_BIG(*) AS OperationCount,
    SUM(wo.DwellTimeMinutes) AS TotalDwellTime,
    AVG(wo.PickRateUnitsPerHour) AS AvgPickRate
FROM dbo.FactWarehouseOperations wo
GROUP BY wo.GeographyKey, wo.ProductKey, wo.TimeKey
GO

CREATE
