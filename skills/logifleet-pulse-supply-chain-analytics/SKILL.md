---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI multi-fact star schema for logistics, warehouse operations, and fleet management analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - configure warehouse and fleet logistics dashboard
  - implement multi-fact star schema for logistics
  - create power bi logistics intelligence dashboard
  - deploy sql server warehouse operations data model
  - build cross-modal supply chain analytics system
  - integrate fleet telemetry with warehouse data
  - setup logistics kpi harmonization platform
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform built on MS SQL Server and Power BI. It provides:

- **Multi-fact star schema** data warehouse connecting warehouse operations, fleet telemetry, inventory, and external data sources
- **Cross-fact KPI harmonization** linking inventory turnover with fleet metrics through shared dimensions
- **Real-time dashboards** with 15-minute refresh cycles for operational awareness
- **Predictive analytics** for bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones** spatial optimization based on pick frequency and item characteristics
- **Temporal elasticity modeling** for simulation scenarios

The platform integrates data from WMS, telematics/GPS, supplier portals, weather/traffic APIs, and customer order systems into a unified semantic layer.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (2022 recommended)
- Power BI Desktop (latest version)
- Database administration privileges
- Access to source systems (WMS, TMS, ERP, telemetry feeds)

### Step 1: Deploy SQL Schema

```sql
-- Connect to your SQL Server instance
-- Create the LogiFleet database
CREATE DATABASE LogiFleetPulse
GO

USE LogiFleetPulse
GO

-- Deploy dimension tables first (time-phased dimensions)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    DateTimeValue DATETIME NOT NULL,
    [Date] DATE NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Day] INT NOT NULL,
    DayOfWeek INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT DEFAULT 0
)
GO

-- Create clustered columnstore index for optimal compression
CREATE CLUSTERED COLUMNSTORE INDEX IX_DimTime ON DimTime
GO

-- Geography dimension with hierarchy
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    ContinentCode NVARCHAR(10),
    ContinentName NVARCHAR(100),
    CountryCode NVARCHAR(10),
    CountryName NVARCHAR(100),
    RegionCode NVARCHAR(20),
    RegionName NVARCHAR(100),
    LocationCode NVARCHAR(50) NOT NULL,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50), -- Warehouse, DistributionCenter, CrossDock, RouteNode
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50),
    IsActive BIT DEFAULT 1,
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE
)
GO

CREATE NONCLUSTERED INDEX IX_DimGeography_LocationCode ON DimGeography(LocationCode)
CREATE NONCLUSTERED INDEX IX_DimGeography_Active ON DimGeography(IsActive) WHERE IsActive = 1
GO

-- Product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    ProductSKU NVARCHAR(50) NOT NULL,
    ProductName NVARCHAR(200),
    ProductCategory NVARCHAR(100),
    ProductSubcategory NVARCHAR(100),
    UnitWeight DECIMAL(10,4),
    UnitVolume DECIMAL(10,4),
    UnitValue DECIMAL(18,2),
    IsFragile BIT DEFAULT 0,
    IsPerishable BIT DEFAULT 0,
    RequiresTemperatureControl BIT DEFAULT 0,
    OptimalTemperatureMin DECIMAL(5,2),
    OptimalTemperatureMax DECIMAL(5,2),
    GravityScore DECIMAL(10,4), -- Calculated: velocity * value / (fragility_factor * volume)
    VelocityClass NVARCHAR(20), -- Fast, Medium, Slow, Dead
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE
)
GO

CREATE UNIQUE NONCLUSTERED INDEX IX_DimProduct_SKU ON DimProduct(ProductSKU, EffectiveDate)
CREATE NONCLUSTERED INDEX IX_DimProduct_Gravity ON DimProduct(GravityScore DESC)
GO

-- Supplier reliability dimension
CREATE TABLE DimSupplier (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierCode NVARCHAR(50) NOT NULL,
    SupplierName NVARCHAR(200),
    SupplierCategory NVARCHAR(100),
    AverageLeadTimeDays DECIMAL(10,2),
    LeadTimeVariance DECIMAL(10,4),
    DefectRatePercentage DECIMAL(5,4),
    ComplianceScore DECIMAL(5,2), -- 0-100
    ReliabilityTier NVARCHAR(20), -- Platinum, Gold, Silver, Bronze, Risk
    EffectiveDate DATE NOT NULL,
    ExpirationDate DATE
)
GO

-- Fact table: Warehouse Operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL,
    GeographyKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    BatchNumber NVARCHAR(100),
    OrderNumber NVARCHAR(100),
    QuantityUnits INT,
    DurationMinutes DECIMAL(10,2),
    DwellTimeHours DECIMAL(10,2),
    ZoneCode NVARCHAR(50),
    GravityZoneAssignment NVARCHAR(20), -- High, Medium, Low
    StaffMemberID NVARCHAR(50),
    EquipmentID NVARCHAR(50),
    TemperatureAtOperation DECIMAL(5,2),
    IsQualityIssue BIT DEFAULT 0,
    IssueCategory NVARCHAR(100),
    FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKey) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO

-- Partitioning strategy for large fact tables
CREATE PARTITION FUNCTION PF_OperationsByMonth (INT)
AS RANGE RIGHT FOR VALUES (
    202401, 202402, 202403, 202404, 202405, 202406,
    202407, 202408, 202409, 202410, 202411, 202412
)
GO

CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time ON FactWarehouseOperations(TimeKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Product ON FactWarehouseOperations(ProductKey)
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Zone ON FactWarehouseOperations(ZoneCode)
GO

-- Fact table: Fleet Trips
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKeyStart INT NOT NULL,
    TimeKeyEnd INT NOT NULL,
    GeographyKeyOrigin INT NOT NULL,
    GeographyKeyDestination INT NOT NULL,
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    RouteCode NVARCHAR(100),
    DistanceKM DECIMAL(10,2),
    DurationMinutes DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    FuelConsumedLiters DECIMAL(10,4),
    LoadWeightKG DECIMAL(10,2),
    LoadVolumeM3 DECIMAL(10,4),
    LoadValueUSD DECIMAL(18,2),
    AverageSpeedKMH DECIMAL(6,2),
    MaxSpeedKMH DECIMAL(6,2),
    HarshBrakingEvents INT DEFAULT 0,
    HarshAccelerationEvents INT DEFAULT 0,
    TemperatureAlerts INT DEFAULT 0,
    IsDelayed BIT DEFAULT 0,
    DelayReasonCategory NVARCHAR(100),
    FOREIGN KEY (TimeKeyStart) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (TimeKeyEnd) REFERENCES DimTime(TimeKey),
    FOREIGN KEY (GeographyKeyOrigin) REFERENCES DimGeography(GeographyKey),
    FOREIGN KEY (GeographyKeyDestination) REFERENCES DimGeography(GeographyKey)
)
GO

CREATE NONCLUSTERED INDEX IX_FactFleet_TimeStart ON FactFleetTrips(TimeKeyStart)
CREATE NONCLUSTERED INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID)
CREATE NONCLUSTERED INDEX IX_FactFleet_Route ON FactFleetTrips(RouteCode)
GO

-- Bridge table for many-to-many: Routes to Products
CREATE TABLE BridgeRouteProducts (
    TripKey BIGINT NOT NULL,
    ProductKey INT NOT NULL,
    QuantityUnits INT,
    PRIMARY KEY (TripKey, ProductKey),
    FOREIGN KEY (TripKey) REFERENCES FactFleetTrips(TripKey),
    FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
)
GO
```

### Step 2: Populate Time Dimension

```sql
-- Stored procedure to populate time dimension
CREATE PROCEDURE sp_PopulateTimeDimension
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @CurrentDateTime DATETIME = @StartDate;
    DECLARE @EndDateTime DATETIME = DATEADD(DAY, 1, @EndDate);
    
    -- Populate in 15-minute increments
    WHILE @CurrentDateTime < @EndDateTime
    BEGIN
        INSERT INTO DimTime (
            TimeKey, DateTimeValue, [Date], [Year], [Quarter], [Month], 
            [Day], DayOfWeek, [Hour], [Minute], FiscalYear, FiscalQuarter, IsWeekend
        )
        VALUES (
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            CAST(@CurrentDateTime AS DATE),
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            DATEPART(MINUTE, @CurrentDateTime),
            -- Fiscal year logic (example: July 1 start)
            CASE 
                WHEN MONTH(@CurrentDateTime) >= 7 THEN YEAR(@CurrentDateTime) + 1
                ELSE YEAR(@CurrentDateTime)
            END,
            CASE 
                WHEN MONTH(@CurrentDateTime) >= 7 THEN ((MONTH(@CurrentDateTime) - 7) / 3) + 1
                ELSE ((MONTH(@CurrentDateTime) + 5) / 3) + 1
            END,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END
GO

-- Execute to populate 2 years of time data
EXEC sp_PopulateTimeDimension '2026-01-01', '2027-12-31'
GO
```

### Step 3: Create Data Loading Stored Procedures

```sql
-- Incremental load procedure for warehouse operations
CREATE PROCEDURE sp_LoadWarehouseOperations
    @LastLoadDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert new records from staging table (assumes external data source)
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        BatchNumber, OrderNumber, QuantityUnits, DurationMinutes,
        DwellTimeHours, ZoneCode, GravityZoneAssignment
    )
    SELECT 
        t.TimeKey,
        g.GeographyKey,
        p.ProductKey,
        stg.OperationType,
        stg.BatchNumber,
        stg.OrderNumber,
        stg.QuantityUnits,
        stg.DurationMinutes,
        stg.DwellTimeHours,
        stg.ZoneCode,
        CASE 
            WHEN p.GravityScore >= 8.0 THEN 'High'
            WHEN p.GravityScore >= 4.0 THEN 'Medium'
            ELSE 'Low'
        END
    FROM StagingWarehouseOperations stg
    INNER JOIN DimTime t ON CAST(FORMAT(stg.OperationDateTime, 'yyyyMMddHHmm') AS INT) = t.TimeKey
    INNER JOIN DimGeography g ON stg.LocationCode = g.LocationCode AND g.IsActive = 1
    INNER JOIN DimProduct p ON stg.ProductSKU = p.ProductSKU 
        AND stg.OperationDateTime BETWEEN p.EffectiveDate AND ISNULL(p.ExpirationDate, '9999-12-31')
    WHERE stg.OperationDateTime > @LastLoadDateTime
        AND stg.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOperations
    SET IsProcessed = 1
    WHERE OperationDateTime > @LastLoadDateTime
        AND IsProcessed = 0;
END
GO
```

### Step 4: Configure Power BI Connection

Open the `LogiFleet_Pulse_Master.pbit` template and configure the connection:

```powerquery
// Power Query M connection code
let
    Source = Sql.Database(
        "${ENV:SQL_SERVER_HOST}", 
        "LogiFleetPulse",
        [
            Query="SELECT * FROM FactWarehouseOperations WHERE TimeKey >= " & 
                  Text.From(Number.From(DateTime.LocalNow() - #duration(90,0,0,0)))
        ]
    )
in
    Source
```

### Step 5: Create Calculated Measures in Power BI

```dax
// DAX measures for cross-fact KPI harmonization

// Total Dwell Time Hours
TotalDwellTime = SUM(FactWarehouseOperations[DwellTimeHours])

// Average Pick Rate per Hour
PickRate = 
DIVIDE(
    CALCULATE(SUM(FactWarehouseOperations[QuantityUnits]), 
              FactWarehouseOperations[OperationType] = "Picking"),
    CALCULATE(SUM(FactWarehouseOperations[DurationMinutes]) / 60,
              FactWarehouseOperations[OperationType] = "Picking"),
    0
)

// Fleet Utilization Percentage
FleetUtilization = 
DIVIDE(
    SUM(FactFleetTrips[DurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[DurationMinutes]),
    0
) * 100

// Cross-Fact: Cost per SKU Moved
CostPerSKUMoved = 
VAR TotalFuelCost = SUM(FactFleetTrips[FuelConsumedLiters]) * 1.45 // USD per liter
VAR TotalSKUsMoved = SUM(FactWarehouseOperations[QuantityUnits])
RETURN DIVIDE(TotalFuelCost, TotalSKUsMoved, 0)

// Gravity Zone Efficiency Score
GravityZoneEfficiency = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DurationMinutes]),
    FactWarehouseOperations[GravityZoneAssignment] = "High"
) / 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DurationMinutes]),
    FactWarehouseOperations[GravityZoneAssignment] = "Low"
)

// Predictive Bottleneck Index (simplified)
BottleneckIndex = 
VAR AvgDwell = AVERAGE(FactWarehouseOperations[DwellTimeHours])
VAR StdDevDwell = STDEV.P(FactWarehouseOperations[DwellTimeHours])
VAR FleetIdleRate = DIVIDE(SUM(FactFleetTrips[IdleTimeMinutes]), 
                            SUM(FactFleetTrips[DurationMinutes]))
RETURN 
    (AvgDwell / NULLIF(StdDevDwell, 0)) * FleetIdleRate * 100
```

## Key Configuration Files

### config.json (Database Connection)

```json
{
  "database": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "windows",
    "connectionTimeout": 30,
    "commandTimeout": 300
  },
  "dataSources": {
    "wms": {
      "type": "rest_api",
      "endpoint": "${WMS_API_ENDPOINT}",
      "authHeader": "Authorization: Bearer ${WMS_API_TOKEN}",
      "refreshIntervalMinutes": 15
    },
    "telematics": {
      "type": "sql_external_table",
      "connectionString": "Server=${TELEMATICS_DB_HOST};Database=FleetData;",
      "refreshIntervalMinutes": 5
    },
    "weather": {
      "type": "rest_api",
      "endpoint": "https://api.openweathermap.org/data/2.5/",
      "apiKey": "${WEATHER_API_KEY}",
      "refreshIntervalMinutes": 60
    }
  },
  "alerts": {
    "enabled": true,
    "emailServer": "${SMTP_SERVER}",
    "emailFrom": "logifleet-alerts@${COMPANY_DOMAIN}",
    "teamsWebhook": "${TEAMS_WEBHOOK_URL}"
  }
}
```

### Row-Level Security Configuration

```sql
-- Create role-based security
CREATE TABLE SecurityRoles (
    RoleID INT PRIMARY KEY IDENTITY(1,1),
    RoleName NVARCHAR(50),
    CanViewFleetCosts BIT DEFAULT 0,
    CanViewSupplierData BIT DEFAULT 0,
    RegionRestriction NVARCHAR(100), -- NULL = all regions
    WarehouseRestriction NVARCHAR(100) -- NULL = all warehouses
)
GO

CREATE TABLE UserRoles (
    UserEmail NVARCHAR(255) PRIMARY KEY,
    RoleID INT NOT NULL,
    FOREIGN KEY (RoleID) REFERENCES SecurityRoles(RoleID)
)
GO

-- Power BI RLS DAX filter
-- Apply to FactWarehouseOperations and FactFleetTrips
[GeographyKey] IN (
    CALCULATETABLE(
        VALUES(DimGeography[GeographyKey]),
        FILTER(
            DimGeography,
            DimGeography[RegionCode] = LOOKUPVALUE(
                SecurityRoles[RegionRestriction],
                UserRoles[UserEmail], USERPRINCIPALNAME(),
                UserRoles[RoleID], SecurityRoles[RoleID]
            )
        )
    )
)
```

## Common Usage Patterns

### Pattern 1: Cross-Fact Analysis Query

```sql
-- Find products with high dwell time AND associated with delayed fleet trips
WITH HighDwellProducts AS (
    SELECT 
        p.ProductSKU,
        p.ProductName,
        AVG(w.DwellTimeHours) AS AvgDwellTime,
        COUNT(*) AS OperationCount
    FROM FactWarehouseOperations w
    INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
    WHERE w.TimeKey >= 202607010000 -- Last quarter
    GROUP BY p.ProductSKU, p.ProductName
    HAVING AVG(w.DwellTimeHours) > 48
),
DelayedRoutes AS (
    SELECT 
        bp.ProductKey,
        COUNT(DISTINCT f.TripKey) AS DelayedTripCount
    FROM FactFleetTrips f
    INNER JOIN BridgeRouteProducts bp ON f.TripKey = bp.TripKey
    WHERE f.IsDelayed = 1
        AND f.TimeKeyStart >= 202607010000
    GROUP BY bp.ProductKey
)
SELECT 
    hdp.ProductSKU,
    hdp.ProductName,
    hdp.AvgDwellTime,
    hdp.OperationCount,
    ISNULL(dr.DelayedTripCount, 0) AS DelayedTripCount,
    CAST(dr.DelayedTripCount AS FLOAT) / hdp.OperationCount AS DelayImpactRatio
FROM HighDwellProducts hdp
LEFT JOIN DelayedRoutes dr ON hdp.ProductSKU = (
    SELECT ProductSKU FROM DimProduct WHERE ProductKey = dr.ProductKey
)
ORDER BY DelayImpactRatio DESC;
```

### Pattern 2: Gravity Zone Optimization Recommendation

```sql
-- Identify products that should be moved to higher gravity zones
SELECT 
    p.ProductSKU,
    p.ProductName,
    p.GravityScore AS CurrentGravityScore,
    w.GravityZoneAssignment AS CurrentZone,
    COUNT(*) AS PickOperations,
    AVG(w.DurationMinutes) AS AvgPickDuration,
    CASE 
        WHEN p.GravityScore >= 8.0 AND w.GravityZoneAssignment != 'High' 
            THEN 'Move to High'
        WHEN p.GravityScore >= 4.0 AND p.GravityScore < 8.0 
            AND w.GravityZoneAssignment = 'Low' 
            THEN 'Move to Medium'
        ELSE 'Optimal'
    END AS Recommendation
FROM FactWarehouseOperations w
INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
WHERE w.OperationType = 'Picking'
    AND w.TimeKey >= 202606010000 -- Last month
GROUP BY p.ProductSKU, p.ProductName, p.GravityScore, w.GravityZoneAssignment
HAVING COUNT(*) > 100 -- Significant pick volume
ORDER BY AvgPickDuration DESC;
```

### Pattern 3: Fleet Maintenance Priority Queue

```sql
-- Generate maintenance priority based on vehicle health and load value
WITH VehicleMetrics AS (
    SELECT 
        VehicleID,
        SUM(LoadValueUSD) AS TotalValueTransported,
        AVG(HarshBrakingEvents + HarshAccelerationEvents) AS AvgDrivingRisk,
        SUM(DistanceKM) AS TotalDistanceKM,
        COUNT(*) AS TripCount
    FROM FactFleetTrips
    WHERE TimeKeyStart >= 202606010000
    GROUP BY VehicleID
)
SELECT 
    vm.VehicleID,
    vm.TotalValueTransported,
    vm.AvgDrivingRisk,
    vm.TotalDistanceKM,
    vm.TripCount,
    -- Weighted priority score
    (vm.TotalValueTransported / 1000000) * 0.4 + -- Value weight
    vm.AvgDrivingRisk * 0.3 + -- Risk weight
    (vm.TotalDistanceKM / 10000) * 0.3 AS MaintenancePriority
FROM VehicleMetrics vm
WHERE vm.TotalDistanceKM > 5000 -- Threshold for maintenance consideration
ORDER BY MaintenancePriority DESC;
```

## Alert Configuration

### Automated Alert Stored Procedure

```sql
CREATE PROCEDURE sp_CheckKPIThresholds
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check 1: Fleet idle time exceeds 15%
    DECLARE @IdlePercent DECIMAL(5,2);
    SELECT @IdlePercent = 
        (SUM(IdleTimeMinutes) * 100.0) / NULLIF(SUM(DurationMinutes), 0)
    FROM FactFleetTrips
    WHERE TimeKeyStart >= CAST(FORMAT(DATEADD(HOUR, -4, GETDATE()), 'yyyyMMddHHmm') AS INT);
    
    IF @IdlePercent > 15.0
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time at ' + 
                           CAST(@IdlePercent AS NVARCHAR(10)) + 
                           '% (threshold: 15%)';
        
        -- Insert into alert log
        INSERT INTO AlertLog (AlertType, AlertMessage, AlertDateTime, Severity)
        VALUES ('Fleet Idle', @AlertMessage, GETDATE(), 'High');
        
        -- Send email via Database Mail (configure separately)
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${OPERATIONS_EMAIL}',
            @subject = 'LogiFleet Pulse: Fleet Idle Alert',
            @body = @AlertMessage;
    END
    
    -- Check 2: Cold storage temperature excursion
    IF EXISTS (
        SELECT 1 FROM FactWarehouseOperations w
        INNER JOIN DimProduct p ON w.ProductKey = p.ProductKey
        WHERE p.RequiresTemperatureControl = 1
            AND w.TimeKey >= CAST(FORMAT(DATEADD(MINUTE, -30, GETDATE()), 'yyyyMMddHHmm') AS INT)
            AND (w.TemperatureAtOperation < p.OptimalTemperatureMin 
                 OR w.TemperatureAtOperation > p.OptimalTemperatureMax)
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Temperature-controlled products out of range in last 30 minutes';
        
        INSERT INTO AlertLog (AlertType, AlertMessage, AlertDateTime, Severity)
        VALUES ('Temperature', @AlertMessage, GETDATE(), 'Critical');
        
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'LogiFleetAlerts',
            @recipients = '${WAREHOUSE_MANAGER_EMAIL}',
            @subject = 'LogiFleet Pulse: Temperature Excursion CRITICAL',
            @body = @AlertMessage;
    END
END
GO

-- Schedule this to run every 15 minutes via SQL Agent
```

## Power BI Dashboard Patterns

### Composite Visual: Dwell Time vs Fleet Idle Correlation

```dax
// Measure for scatter plot X-axis
AvgDwellByProduct = 
CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeHours]),
    ALLEXCEPT(DimProduct, DimProduct[ProductSKU])
)

// Measure for scatter plot Y-axis
AvgFleetIdleForProduct = 
VAR CurrentProduct = SELECTEDVALUE(DimProduct[ProductKey])
RETURN
CALCULATE(
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[DurationMinutes])
    ) * 100,
    FILTER(
        BridgeRouteProducts,
        BridgeRouteProducts[ProductKey] = CurrentProduct
    )
)

// Size bubble by total value transported
TotalProductValue = 
SUMX(
    FILTER(
        BridgeRouteProducts,
        BridgeRouteProducts[ProductKey] = SELECTEDVALUE(DimProduct[ProductKey])
    ),
    RELATED(FactFleetTrips[LoadValueUSD])
)
```

## Troubleshooting

### Issue: Slow Query Performance on Fact Tables

**Solution**: Implement columnstore indexing and partitioning

```sql
-- Convert to columnstore for analytical queries
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactWarehouse 
ON FactWarehouseOperations 
WITH (DROP_EXISTING = OFF);

-- Partition by month for better query pruning
ALTER TABLE FactWarehouseOperations
SWITCH PARTITION $PARTITION.PF_OperationsByMonth(202607)
TO StagingFactWarehouseOperations PARTITION $PARTITION.PF_OperationsByMonth(202607);
```

### Issue: Power BI Refresh Timeout

**Solution**: Use incremental refresh policy

```powerquery
// Power Query parameter for incremental refresh
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetPulse"),
    FilteredRows = Table.SelectRows(
        Source, 
        each [TimeKey] >= Number.From(RangeStart) 
         and [TimeKey] < Number.From(RangeEnd)
    )
in
    FilteredRows
```

Configure in Power BI Desktop: Modeling → Incremental Refresh → Store 2 years, Refresh last 7 days

### Issue: Row-Level Security Not Filtering Correctly

**Diagnostic Query**:

```sql
-- Verify user role assignments
SELECT 
    ur.UserEmail,
    sr.RoleName,
    sr.RegionRestriction,
    COUNT(DISTINCT g.GeographyKey) AS AccessibleLocations
FROM UserRoles ur
INNER JOIN SecurityRoles sr ON ur.RoleID = sr.RoleID
LEFT JOIN DimGeography g ON 
    (sr.RegionRestriction IS NULL OR g.RegionCode = sr.RegionRestriction)
    AND g.IsActive = 1
WHERE ur.UserEmail = 'user@company.com'
GROUP BY ur.UserEmail, sr.RoleName, sr.RegionRestriction;
```

### Issue: Missing Time Dimension Records

```sql
-- Find gaps in time dimension
WITH TimeSequence AS (
    SELECT 
        TimeKey,
        LAG(TimeKey) OVER (ORDER BY TimeKey) AS PrevTimeKey
    FROM DimTime
)
SELECT 
    TimeKey,
    PrevTimeKey,
    TimeKey - PrevTimeKey AS Gap
FROM TimeSequence
WHERE TimeKey - PrevTimeKey > 15 -- More than 15 minutes gap
ORDER BY TimeKey;

-- Regenerate missing periods
EXEC sp_PopulateTimeDimension '2026-05-15', '2026-05-20';
```

## Advanced Pattern: Temporal Elasticity Simulation

```sql
-- Simulate warehouse capacity increase impact on fleet efficiency
WITH CapacityScenario AS (
    SELECT 
        'Current' AS Scenario,
        AVG(DwellTimeHours) AS Avg
