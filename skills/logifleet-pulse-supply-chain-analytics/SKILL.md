---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing engine for logistics, fleet management, and warehouse operations analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "deploy logistics data warehouse schema"
  - "create Power BI dashboard for fleet management"
  - "configure warehouse gravity zones"
  - "build multi-fact star schema for logistics"
  - "implement cross-modal supply chain analytics"
  - "set up real-time fleet optimization dashboard"
  - "create warehouse operations KPI tracking"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced MS SQL Server and Power BI data warehousing solution designed for logistics and supply chain analytics. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, and inventory data
- **Cross-modal analytics** that harmonize warehouse velocity with fleet performance
- **Real-time dashboards** with 15-minute refresh intervals
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Warehouse Gravity Zones™** for optimal storage placement based on velocity and value
- **Adaptive fleet triage** prioritizing maintenance by revenue impact

The platform ingests data from WMS, telematics, supplier portals, weather APIs, and customer orders into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Express, Standard, or Enterprise)
- Power BI Desktop (latest version)
- SQL Server Management Studio (SSMS) or Azure Data Studio
- Read/write access to SQL Server instance

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance and create a new database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    TimeOfDay TIME,
    HourOfDay TINYINT,
    DayOfWeek TINYINT,
    DayName VARCHAR(10),
    DayOfMonth TINYINT,
    MonthOfYear TINYINT,
    MonthName VARCHAR(10),
    Quarter TINYINT,
    Year SMALLINT,
    FiscalPeriod VARCHAR(10),
    IsWeekend BIT,
    IsHoliday BIT
);

CREATE CLUSTERED INDEX IX_DimTime_FullDateTime ON DimTime(FullDateTime);
GO

-- Create geography dimension (hierarchical)
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) UNIQUE NOT NULL,
    LocationName VARCHAR(200),
    LocationType VARCHAR(50), -- 'Warehouse', 'Route Node', 'Customer Site'
    Address VARCHAR(500),
    City VARCHAR(100),
    Region VARCHAR(100),
    Country VARCHAR(100),
    Continent VARCHAR(50),
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    TimeZone VARCHAR(50),
    EffectiveDate DATE,
    ExpirationDate DATE,
    IsCurrent BIT DEFAULT 1
);

CREATE NONCLUSTERED INDEX IX_DimGeography_Country_Region ON DimGeography(Country, Region);
GO

-- Create product gravity dimension
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) UNIQUE NOT NULL,
    ProductName VARCHAR(500),
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT,
    IsPerishable BIT,
    ShelfLifeDays INT,
    UnitCost DECIMAL(18,2),
    UnitPrice DECIMAL(18,2),
    GravityScore DECIMAL(5,2), -- Computed: velocity * value / lead_time
    OptimalZone VARCHAR(20), -- 'High', 'Medium', 'Low', 'Bulk'
    LastGravityUpdate DATETIME2
);

CREATE NONCLUSTERED INDEX IX_DimProductGravity_GravityScore ON DimProductGravity(GravityScore DESC);
GO

-- Create supplier reliability dimension
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(100) UNIQUE NOT NULL,
    SupplierName VARCHAR(300),
    Country VARCHAR(100),
    AvgLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectPercentage DECIMAL(5,3),
    OnTimeDeliveryRate DECIMAL(5,3),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier VARCHAR(20), -- 'Platinum', 'Gold', 'Silver', 'Bronze', 'Watch'
    LastReviewDate DATE
);
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    SupplierKey INT FOREIGN KEY REFERENCES DimSupplierReliability(SupplierKey),
    OperationType VARCHAR(50), -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    OrderID VARCHAR(100),
    BatchNumber VARCHAR(100),
    QuantityUnits INT,
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    StorageZone VARCHAR(50),
    AssignedGravityZone VARCHAR(20),
    OperatorID VARCHAR(50),
    EquipmentID VARCHAR(50),
    QualityCheckPassed BIT,
    ExceptionCode VARCHAR(50),
    TotalCost DECIMAL(18,2),
    LoadTimestamp DATETIME2 DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey);
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Geography ON FactWarehouseOperations(GeographyKey);
GO

-- Create fleet trips fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(100),
    DriverID VARCHAR(100),
    RouteID VARCHAR(100),
    TripDurationMinutes INT,
    IdleTimeMinutes INT,
    LoadingTimeMinutes INT,
    UnloadingTimeMinutes INT,
    DistanceKM DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,2),
    AvgSpeedKPH DECIMAL(5,2),
    MaxSpeedKPH DECIMAL(5,2),
    LoadWeightKG DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,2),
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes INT,
    OnTimeDelivery BIT,
    TripCost DECIMAL(18,2),
    RevenuePotential DECIMAL(18,2),
    MaintenanceAlertFlag BIT,
    LoadTimestamp DATETIME2 DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(RouteID);
GO

-- Create cross-dock fact table
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    TransferTimeMinutes INT,
    QuantityTransferred INT,
    DirectTransfer BIT, -- TRUE = no storage, FALSE = temporary holding
    QualityCheckRequired BIT,
    TemperatureControlled BIT,
    LoadTimestamp DATETIME2 DEFAULT GETDATE()
);

CREATE NONCLUSTERED INDEX IX_FactCrossDock_Time ON FactCrossDock(TimeKey);
GO
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate time dimension with 15-minute intervals
CREATE PROCEDURE PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    DECLARE @TimeKey INT;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        SET @TimeKey = CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT);
        
        INSERT INTO DimTime (
            TimeKey, FullDateTime, TimeOfDay, HourOfDay, DayOfWeek, DayName,
            DayOfMonth, MonthOfYear, MonthName, Quarter, Year, FiscalPeriod,
            IsWeekend, IsHoliday
        )
        VALUES (
            @TimeKey,
            @CurrentDateTime,
            CAST(@CurrentDateTime AS TIME),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATENAME(WEEKDAY, @CurrentDateTime),
            DATEPART(DAY, @CurrentDateTime),
            DATEPART(MONTH, @CurrentDateTime),
            DATENAME(MONTH, @CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            DATEPART(YEAR, @CurrentDateTime),
            CONCAT('FY', DATEPART(YEAR, @CurrentDateTime), '-Q', DATEPART(QUARTER, @CurrentDateTime)),
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END,
            0 -- Update holiday logic as needed
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
GO

-- Execute to populate 2 years of time data
EXEC PopulateTimeDimension @StartDate = '2025-01-01', @EndDate = '2026-12-31';
GO
```

### Step 3: Create Cross-Fact Views

```sql
-- Unified view combining warehouse dwell time with fleet idle time
CREATE VIEW vw_CrossFactBottleneckAnalysis
AS
SELECT 
    t.FullDateTime,
    t.DayName,
    t.HourOfDay,
    g.LocationName AS WarehouseName,
    g.Region,
    p.SKU,
    p.ProductName,
    p.GravityScore,
    p.OptimalZone,
    -- Warehouse metrics
    AVG(w.DwellTimeMinutes) AS AvgDwellTimeMinutes,
    SUM(w.QuantityUnits) AS TotalUnitsProcessed,
    COUNT(DISTINCT w.OperationKey) AS OperationCount,
    -- Fleet metrics (trips originating from this warehouse)
    AVG(f.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(f.FuelConsumedLiters) AS AvgFuelConsumed,
    SUM(CASE WHEN f.OnTimeDelivery = 1 THEN 1 ELSE 0 END) * 100.0 / NULLIF(COUNT(f.TripKey), 0) AS OnTimeDeliveryRate,
    -- Combined bottleneck score
    (AVG(w.DwellTimeMinutes) * 0.4 + AVG(f.IdleTimeMinutes) * 0.6) AS BottleneckScore
FROM FactWarehouseOperations w
INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
INNER JOIN DimGeography g ON w.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
LEFT JOIN FactFleetTrips f ON f.OriginGeographyKey = w.GeographyKey 
    AND f.TimeKey = w.TimeKey
WHERE w.OperationType IN ('Putaway', 'Picking')
GROUP BY 
    t.FullDateTime, t.DayName, t.HourOfDay, g.LocationName, g.Region,
    p.SKU, p.ProductName, p.GravityScore, p.OptimalZone;
GO

-- Supplier impact on warehouse and fleet performance
CREATE VIEW vw_SupplierPerformanceImpact
AS
SELECT 
    s.SupplierName,
    s.Country AS SupplierCountry,
    s.ReliabilityTier,
    s.OnTimeDeliveryRate AS SupplierOTD,
    -- Warehouse impact
    AVG(w.DwellTimeMinutes) AS AvgWarehouseDwellTime,
    AVG(w.ProcessingTimeMinutes) AS AvgProcessingTime,
    SUM(CASE WHEN w.QualityCheckPassed = 0 THEN 1 ELSE 0 END) * 100.0 / NULLIF(COUNT(*), 0) AS DefectRate,
    -- Fleet impact (correlation)
    AVG(f.TrafficDelayMinutes) AS AvgTrafficDelay,
    SUM(f.TripCost) AS TotalFleetCost,
    SUM(f.RevenuePotential) AS TotalRevenuePotential
FROM DimSupplierReliability s
INNER JOIN FactWarehouseOperations w ON s.SupplierKey = w.SupplierKey
LEFT JOIN FactFleetTrips f ON f.TimeKey = w.TimeKey
GROUP BY s.SupplierName, s.Country, s.ReliabilityTier, s.OnTimeDeliveryRate;
GO
```

### Step 4: Create Automated Alerting

```sql
-- Stored procedure for real-time alert generation
CREATE PROCEDURE GenerateLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Create alerts table if not exists
    IF NOT EXISTS (SELECT * FROM sys.tables WHERE name = 'AlertLog')
    BEGIN
        CREATE TABLE AlertLog (
            AlertID BIGINT PRIMARY KEY IDENTITY(1,1),
            AlertTimestamp DATETIME2 DEFAULT GETDATE(),
            AlertType VARCHAR(50),
            Severity VARCHAR(20), -- 'Critical', 'High', 'Medium', 'Low'
            Message NVARCHAR(1000),
            AffectedEntity VARCHAR(200),
            RecommendedAction NVARCHAR(1000),
            IsAcknowledged BIT DEFAULT 0,
            AcknowledgedBy VARCHAR(100),
            AcknowledgedAt DATETIME2
        );
    END
    
    -- Alert 1: High dwell time for high-gravity products
    INSERT INTO AlertLog (AlertType, Severity, Message, AffectedEntity, RecommendedAction)
    SELECT 
        'High Dwell Time',
        'High',
        CONCAT('SKU ', p.SKU, ' has avg dwell time of ', CAST(AVG(w.DwellTimeMinutes) AS VARCHAR), ' minutes (threshold: 120)'),
        p.SKU,
        'Reassign to higher gravity zone or increase picking staff'
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -4, GETDATE())
        AND p.GravityScore > 70
        AND w.OperationType = 'Picking'
    GROUP BY p.SKU, p.ProductName
    HAVING AVG(w.DwellTimeMinutes) > 120;
    
    -- Alert 2: Fleet idle time exceeds 15% of trip duration
    INSERT INTO AlertLog (AlertType, Severity, Message, AffectedEntity, RecommendedAction)
    SELECT 
        'High Fleet Idle Time',
        'Critical',
        CONCAT('Vehicle ', f.VehicleID, ' idle time is ', CAST((f.IdleTimeMinutes * 100.0 / NULLIF(f.TripDurationMinutes, 0)) AS VARCHAR), '% of trip'),
        f.VehicleID,
        'Review route optimization and driver behavior; check for mechanical issues'
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -2, GETDATE())
        AND (f.IdleTimeMinutes * 100.0 / NULLIF(f.TripDurationMinutes, 0)) > 15;
    
    -- Alert 3: Maintenance flag on high-revenue trips
    INSERT INTO AlertLog (AlertType, Severity, Message, AffectedEntity, RecommendedAction)
    SELECT 
        'Maintenance Alert on High-Revenue Trip',
        'Critical',
        CONCAT('Vehicle ', f.VehicleID, ' requires maintenance and is assigned to $', CAST(f.RevenuePotential AS VARCHAR), ' revenue trip'),
        f.VehicleID,
        'Reassign to backup vehicle immediately; schedule urgent maintenance'
    FROM FactFleetTrips f
    INNER JOIN DimTime t ON f.TimeKey = t.TimeKey
    WHERE t.FullDateTime >= DATEADD(HOUR, -1, GETDATE())
        AND f.MaintenanceAlertFlag = 1
        AND f.RevenuePotential > 5000;
        
END;
GO

-- Schedule this to run every 15 minutes via SQL Agent
```

### Step 5: Configure Power BI Connection

1. Download the repository and open `LogiFleet_Pulse_Master.pbit`
2. When prompted, enter your SQL Server connection details:
   - **Server**: `your-server-name.database.windows.net` or `localhost\SQLEXPRESS`
   - **Database**: `LogiFleetPulse`
   - **Data Connectivity Mode**: DirectQuery (for real-time) or Import (for performance)

3. Update the M query parameters if using environment-specific configs:

```powerquery
let
    ServerName = #"SQL_SERVER_HOST",
    DatabaseName = "LogiFleetPulse",
    Source = Sql.Database(ServerName, DatabaseName)
in
    Source
```

## Key Configuration Patterns

### Gravity Score Calculation

Update gravity scores based on recent velocity and value:

```sql
-- Update product gravity scores (run weekly or after major demand shifts)
UPDATE DimProductGravity
SET 
    GravityScore = (
        SELECT 
            (COUNT(*) * 1.0 / NULLIF(DATEDIFF(DAY, MIN(t.FullDateTime), MAX(t.FullDateTime)), 0)) -- velocity
            * UnitPrice -- value
            / NULLIF(s.AvgLeadTimeDays, 0) -- lead time
        FROM FactWarehouseOperations w
        INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
        LEFT JOIN DimSupplierReliability s ON w.SupplierKey = s.SupplierKey
        WHERE w.ProductKey = DimProductGravity.ProductKey
            AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
            AND w.OperationType IN ('Picking', 'Shipping')
    ),
    OptimalZone = CASE 
        WHEN GravityScore > 80 THEN 'High'
        WHEN GravityScore > 50 THEN 'Medium'
        WHEN GravityScore > 20 THEN 'Low'
        ELSE 'Bulk'
    END,
    LastGravityUpdate = GETDATE()
WHERE EXISTS (
    SELECT 1 FROM FactWarehouseOperations w
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE w.ProductKey = DimProductGravity.ProductKey
        AND t.FullDateTime >= DATEADD(DAY, -30, GETDATE())
);
```

### Data Ingestion from External Sources

#### Example: Ingest WMS data via CSV

```sql
-- Bulk insert from CSV export
BULK INSERT FactWarehouseOperations
FROM '\\network-share\wms-export\operations_20260701.csv'
WITH (
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    FIRSTROW = 2,
    TABLOCK,
    ERRORFILE = '\\network-share\errors\operations_errors.csv'
);
GO
```

#### Example: Link to external telemetry API via Polybase

```sql
-- Create external data source (requires Polybase enabled)
CREATE EXTERNAL DATA SOURCE FleetTelemetryAPI
WITH (
    TYPE = HADOOP,
    LOCATION = 'https://api.telemetryprovider.com/v2',
    CREDENTIAL = FleetAPICredential -- Created separately with API key
);

-- Create external table for streaming GPS data
CREATE EXTERNAL TABLE ext_FleetGPSStream (
    VehicleID VARCHAR(100),
    Timestamp DATETIME2,
    Latitude DECIMAL(10,7),
    Longitude DECIMAL(10,7),
    Speed DECIMAL(5,2),
    FuelLevel DECIMAL(5,2)
)
WITH (
    DATA_SOURCE = FleetTelemetryAPI,
    LOCATION = '/gps-stream',
    FILE_FORMAT = JSONFormat
);
```

### Row-Level Security for Multi-Tenant Access

```sql
-- Create security predicate function
CREATE FUNCTION fn_SecurityPredicate(@Region VARCHAR(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS AccessAllowed
WHERE @Region = USER_NAME() 
    OR IS_MEMBER('GlobalAdmin') = 1
    OR IS_MEMBER('ExecutiveView') = 1;
GO

-- Apply security policy to geography dimension
CREATE SECURITY POLICY RegionalAccessPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(Region) ON dbo.DimGeography,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(Region) ON dbo.FactWarehouseOperations,
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(Region) ON dbo.FactFleetTrips
WITH (STATE = ON);
GO
```

## Power BI DAX Measures

### Cross-Fact KPI Measures

```dax
// Bottleneck Index (combines dwell time and idle time)
Bottleneck Index = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeMinutes])
VAR AvgIdle = AVERAGE(FactFleetTrips[IdleTimeMinutes])
VAR NormalizedDwell = DIVIDE(AvgDwell, 180, 0) // Normalize to 3-hour baseline
VAR NormalizedIdle = DIVIDE(AvgIdle, 60, 0) // Normalize to 1-hour baseline
RETURN (NormalizedDwell * 0.4) + (NormalizedIdle * 0.6)

// Fleet Utilization Rate
Fleet Utilization % = 
DIVIDE(
    SUMX(FactFleetTrips, FactFleetTrips[TripDurationMinutes] - FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// Warehouse Gravity Compliance
Gravity Compliance % = 
VAR TotalOps = COUNTROWS(FactWarehouseOperations)
VAR CompliantOps = CALCULATE(
    COUNTROWS(FactWarehouseOperations),
    FactWarehouseOperations[StorageZone] = FactWarehouseOperations[AssignedGravityZone]
)
RETURN DIVIDE(CompliantOps, TotalOps, 0) * 100

// Cross-Dock Efficiency
Cross-Dock Efficiency = 
DIVIDE(
    CALCULATE(
        COUNTROWS(FactCrossDock),
        FactCrossDock[DirectTransfer] = TRUE()
    ),
    COUNTROWS(FactCrossDock),
    0
) * 100

// Supplier-Driven Delay Impact
Supplier Delay Impact Hours = 
SUMX(
    FactWarehouseOperations,
    VAR SupplierOTD = RELATED(DimSupplierReliability[OnTimeDeliveryRate])
    RETURN IF(SupplierOTD < 0.85, FactWarehouseOperations[DwellTimeMinutes] / 60, 0)
)
```

### Time Intelligence

```dax
// Rolling 30-Day Average Dwell Time
Dwell Time 30D MA = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    DATESINPERIOD(DimTime[FullDateTime], LASTDATE(DimTime[FullDateTime]), -30, DAY)
)

// Year-over-Year Fuel Efficiency
Fuel Efficiency YoY % = 
VAR CurrentYearFuel = SUM(FactFleetTrips[FuelConsumedLiters])
VAR PriorYearFuel = CALCULATE(
    SUM(FactFleetTrips[FuelConsumedLiters]),
    SAMEPERIODLASTYEAR(DimTime[FullDateTime])
)
RETURN DIVIDE(CurrentYearFuel - PriorYearFuel, PriorYearFuel, 0) * 100
```

## Common Workflows

### Workflow 1: Daily Operations Dashboard Setup

1. Create a new Power BI report page
2. Add a **Card visual** for each key metric:
   - Total Operations (COUNT of FactWarehouseOperations)
   - Fleet Utilization % (measure above)
   - Bottleneck Index (measure above)
3. Add a **Line chart** with:
   - X-axis: DimTime[HourOfDay]
   - Y-axis: Average Dwell Time, Average Idle Time
   - Legend: DimGeography[Region]
4. Add a **Matrix visual** for product gravity:
   - Rows: DimProductGravity[Category], DimProductGravity[SKU]
   - Values: Gravity Score, Optimal Zone, Avg Dwell Time
5. Set refresh to every 15 minutes via Power BI Service

### Workflow 2: Predictive Bottleneck Alert

```sql
-- Identify future bottlenecks based on trending patterns
WITH TrendAnalysis AS (
    SELECT 
        GeographyKey,
        ProductKey,
        AVG(DwellTimeMinutes) AS CurrentAvgDwell,
        AVG(DwellTimeMinutes) OVER (
            PARTITION BY GeographyKey, ProductKey 
            ORDER BY TimeKey 
            ROWS BETWEEN 96 PRECEDING AND CURRENT ROW -- 24 hours at 15-min intervals
        ) AS TrendingAvgDwell
    FROM FactWarehouseOperations
    WHERE TimeKey >= (SELECT MAX(TimeKey) - 672 FROM FactWarehouseOperations) -- Last week
)
SELECT 
    g.LocationName,
    p.SKU,
    p.ProductName,
    t.CurrentAvgDwell,
    t.TrendingAvgDwell,
    ((t.TrendingAvgDwell - t.CurrentAvgDwell) * 100.0 / NULLIF(t.CurrentAvgDwell, 0)) AS TrendPercentChange,
    CASE 
        WHEN ((t.TrendingAvgDwell - t.CurrentAvgDwell) * 100.0 / NULLIF(t.CurrentAvgDwell, 0)) > 20 
        THEN 'HIGH RISK - Accelerating Dwell'
        WHEN ((t.TrendingAvgDwell - t.CurrentAvgDwell) * 100.0 / NULLIF(t.CurrentAvgDwell, 0)) > 10 
        THEN 'MEDIUM RISK'
        ELSE 'LOW RISK'
    END AS RiskLevel
FROM TrendAnalysis t
INNER JOIN DimGeography g ON t.GeographyKey = g.GeographyKey
INNER JOIN DimProductGravity p ON t.ProductKey = p.ProductKey
WHERE ((t.TrendingAvgDwell - t.CurrentAvgDwell) * 100.0 / NULLIF(t.CurrentAvgDwell, 0)) > 10
ORDER BY TrendPercentChange DESC;
```

### Workflow 3: Warehouse Zone Rebalancing Recommendation

```sql
-- Generate recommendations for moving products to different gravity zones
WITH ZoneAnalysis AS (
    SELECT 
        p.ProductKey,
        p.SKU,
        p.ProductName,
        p.GravityScore,
        p.OptimalZone,
        w.StorageZone AS CurrentZone,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(*) AS PickFrequency,
        SUM(w.QuantityUnits) AS TotalUnitsMovement
    FROM FactWarehouseOperations w
    INNER JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    INNER JOIN DimTime t ON w.TimeKey = t.TimeKey
    WHERE w.OperationType IN ('Picking', 'Putaway')
        AND t.FullDateTime >= DATEADD(DAY, -14, GETDATE())
    GROUP BY p.ProductKey, p.SKU, p.ProductName, p.GravityScore, p.OptimalZone, w.StorageZone
)
SELECT 
    SKU,
    ProductName,
    GravityScore,
    CurrentZone,
    OptimalZone,
    AvgDwellTime,
    PickFrequency,
    TotalUnitsMovement,
    CASE 
        WHEN CurrentZone <> OptimalZone AND PickFrequency > 50 
        THEN CONCAT('MOVE to ', OptimalZone, ' zone - High priority')
        WHEN CurrentZone <> OptimalZone AND PickFrequency > 20 
        THEN CONCAT('CONSIDER moving to ', OptimalZone, ' zone')
        ELSE 'Current placement acceptable'
    END AS Recommendation,
    (
