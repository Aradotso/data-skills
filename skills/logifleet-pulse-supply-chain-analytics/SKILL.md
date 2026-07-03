---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing and fleet logistics intelligence platform for real-time supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy multi-fact star schema for warehouse operations"
  - "create fleet telemetry data model"
  - "build supply chain KPI dashboard"
  - "implement warehouse gravity zone analytics"
  - "troubleshoot Power BI logistics visualization"
  - "optimize SQL Server logistics data warehouse"
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is a multi-modal logistics intelligence engine that combines warehouse operations, fleet telemetry, inventory management, and external data signals into a unified Power BI analytical platform. It uses MS SQL Server for transactional integrity with a custom multi-fact star schema architecture.

## What It Does

- **Unified Data Model**: Multi-fact star schema linking warehouse, fleet, and supply chain data
- **Real-Time Dashboards**: Power BI visualizations with 15-minute refresh cycles
- **Cross-Fact KPIs**: Harmonized metrics across warehouse operations and fleet performance
- **Predictive Analytics**: Bottleneck detection and fleet maintenance prioritization
- **Warehouse Gravity Zones**: Spatial optimization based on pick frequency and item characteristics
- **Temporal Modeling**: Time-phased simulation scenarios for capacity planning

## Installation & Setup

### Prerequisites

- MS SQL Server 2019+ (Standard or Enterprise edition recommended)
- Power BI Desktop (latest version)
- SQLCMD or SQL Server Management Studio (SSMS)
- Database permissions: CREATE DATABASE, CREATE TABLE, CREATE PROCEDURE

### Step 1: Deploy SQL Schema

Clone the repository and deploy the database schema:

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

Execute the main schema script:

```sql
-- Run in SSMS or via sqlcmd
-- Assumes SQL Server instance is localhost and database will be LogiFleetDB

USE master;
GO

CREATE DATABASE LogiFleetDB;
GO

USE LogiFleetDB;
GO

-- Create time dimension (15-minute granularity)
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME NOT NULL,
    [Date] DATE NOT NULL,
    [Year] INT NOT NULL,
    [Quarter] INT NOT NULL,
    [Month] INT NOT NULL,
    [Week] INT NOT NULL,
    [Day] INT NOT NULL,
    [Hour] INT NOT NULL,
    [Minute] INT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalQuarter INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);
GO

CREATE INDEX IX_DimTime_Date ON DimTime([Date]);
CREATE INDEX IX_DimTime_FiscalYear ON DimTime(FiscalYear, FiscalQuarter);
GO

-- Create geography dimension
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID NVARCHAR(50) NOT NULL UNIQUE,
    LocationName NVARCHAR(200) NOT NULL,
    LocationType NVARCHAR(50) NOT NULL, -- Warehouse, DistributionCenter, RouteNode
    [Address] NVARCHAR(500),
    City NVARCHAR(100),
    StateProvince NVARCHAR(100),
    Country NVARCHAR(100),
    Continent NVARCHAR(50),
    PostalCode NVARCHAR(20),
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone NVARCHAR(50)
);
GO

CREATE INDEX IX_DimGeography_LocationType ON DimGeography(LocationType);
CREATE INDEX IX_DimGeography_Country ON DimGeography(Country);
GO

-- Create product dimension with gravity scoring
CREATE TABLE DimProduct (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU NVARCHAR(50) NOT NULL UNIQUE,
    ProductName NVARCHAR(200) NOT NULL,
    Category NVARCHAR(100),
    SubCategory NVARCHAR(100),
    Brand NVARCHAR(100),
    UnitCost DECIMAL(18,2),
    UnitPrice DECIMAL(18,2),
    Weight DECIMAL(10,2), -- kg
    Volume DECIMAL(10,2), -- cubic meters
    IsFragile BIT DEFAULT 0,
    RequiresRefrigeration BIT DEFAULT 0,
    ShelfLifeDays INT,
    GravityScore DECIMAL(5,2) -- Calculated: velocity * value / fragility factor
);
GO

CREATE INDEX IX_DimProduct_Category ON DimProduct(Category, SubCategory);
CREATE INDEX IX_DimProduct_GravityScore ON DimProduct(GravityScore DESC);
GO

-- Create warehouse operations fact table
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProduct(ProductKey),
    OperationType NVARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OrderID NVARCHAR(50),
    QuantityHandled INT NOT NULL,
    DwellTimeMinutes INT, -- Time in staging area
    CycleTimeMinutes INT, -- Time to complete operation
    ZoneID NVARCHAR(50), -- Storage zone identifier
    EmployeeID NVARCHAR(50),
    ErrorCount INT DEFAULT 0,
    OperationCost DECIMAL(18,2)
);
GO

CREATE INDEX IX_FactWH_Time ON FactWarehouseOperations(TimeKey);
CREATE INDEX IX_FactWH_Product ON FactWarehouseOperations(ProductKey);
CREATE INDEX IX_FactWH_OpType ON FactWarehouseOperations(OperationType);
GO

-- Create fleet operations fact table
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TripID NVARCHAR(50) NOT NULL,
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID NVARCHAR(50) NOT NULL,
    DriverID NVARCHAR(50),
    DistanceKM DECIMAL(10,2),
    DurationMinutes INT,
    IdleTimeMinutes INT,
    FuelConsumedLiters DECIMAL(10,2),
    LoadWeight DECIMAL(10,2), -- kg
    OnTimeDelivery BIT,
    DelayMinutes INT DEFAULT 0,
    WeatherCondition NVARCHAR(50),
    TrafficLevel NVARCHAR(20), -- Low, Medium, High
    TripCost DECIMAL(18,2)
);
GO

CREATE INDEX IX_FactFleet_Time ON FactFleetTrips(TimeKey);
CREATE INDEX IX_FactFleet_Vehicle ON FactFleetTrips(VehicleID);
CREATE INDEX IX_FactFleet_Origin ON FactFleetTrips(OriginGeographyKey);
GO
```

### Step 2: Configure Data Sources

Create a configuration file for data source connections:

```json
{
  "connections": {
    "sqlServer": {
      "server": "${SQL_SERVER_HOST}",
      "database": "LogiFleetDB",
      "authentication": "integrated",
      "connectionTimeout": 30
    },
    "dataSources": {
      "wms": {
        "type": "REST_API",
        "endpoint": "${WMS_API_ENDPOINT}",
        "authToken": "${WMS_API_TOKEN}",
        "refreshInterval": 15
      },
      "telemetry": {
        "type": "REST_API",
        "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
        "authToken": "${FLEET_API_TOKEN}",
        "refreshInterval": 5
      },
      "weather": {
        "type": "REST_API",
        "endpoint": "https://api.openweathermap.org/data/2.5",
        "apiKey": "${WEATHER_API_KEY}",
        "refreshInterval": 60
      }
    }
  },
  "powerBI": {
    "workspaceId": "${POWERBI_WORKSPACE_ID}",
    "refreshSchedule": "*/15 * * * *"
  }
}
```

### Step 3: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template (`.pbit`)
3. Select `LogiFleet_Pulse_Master.pbit` from the repository
4. Enter connection parameters when prompted:
   - SQL Server instance name
   - Database name: `LogiFleetDB`
   - Authentication method

The template includes pre-built relationships and measures.

## Key SQL Stored Procedures

### ETL: Load Warehouse Operations

```sql
CREATE PROCEDURE sp_LoadWarehouseOperations
    @StartDateTime DATETIME,
    @EndDateTime DATETIME
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert operations from staging table
    INSERT INTO FactWarehouseOperations (
        TimeKey, GeographyKey, ProductKey, OperationType,
        OrderID, QuantityHandled, DwellTimeMinutes, 
        CycleTimeMinutes, ZoneID, EmployeeID, ErrorCount, OperationCost
    )
    SELECT 
        dt.TimeKey,
        dg.GeographyKey,
        dp.ProductKey,
        s.OperationType,
        s.OrderID,
        s.Quantity,
        DATEDIFF(MINUTE, s.StagingStartTime, s.OperationStartTime) AS DwellTimeMinutes,
        DATEDIFF(MINUTE, s.OperationStartTime, s.OperationEndTime) AS CycleTimeMinutes,
        s.ZoneID,
        s.EmployeeID,
        s.ErrorCount,
        s.LaborCost + s.EquipmentCost AS OperationCost
    FROM StagingWarehouseOperations s
    INNER JOIN DimTime dt ON CAST(s.OperationStartTime AS DATETIME) = dt.FullDateTime
    INNER JOIN DimGeography dg ON s.LocationID = dg.LocationID
    INNER JOIN DimProduct dp ON s.SKU = dp.SKU
    WHERE s.OperationStartTime >= @StartDateTime 
      AND s.OperationStartTime < @EndDateTime
      AND s.IsProcessed = 0;
    
    -- Mark staging records as processed
    UPDATE StagingWarehouseOperations
    SET IsProcessed = 1, ProcessedDate = GETDATE()
    WHERE OperationStartTime >= @StartDateTime 
      AND OperationStartTime < @EndDateTime;
END;
GO
```

### Calculate Warehouse Gravity Scores

```sql
CREATE PROCEDURE sp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity score based on velocity, value, and fragility
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            p.SKU,
            COUNT(DISTINCT f.OperationKey) AS PickCount,
            AVG(p.UnitPrice) AS AvgPrice,
            CASE 
                WHEN p.IsFragile = 1 THEN 0.7
                WHEN p.RequiresRefrigeration = 1 THEN 0.8
                ELSE 1.0
            END AS FragilityFactor
        FROM DimProduct p
        LEFT JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey
        WHERE f.OperationType = 'Picking'
          AND f.TimeKey >= (SELECT MAX(TimeKey) - 720 FROM DimTime) -- Last 30 days
        GROUP BY p.ProductKey, p.SKU, p.UnitPrice, p.IsFragile, p.RequiresRefrigeration
    )
    UPDATE p
    SET p.GravityScore = (
        (pm.PickCount * 0.5 + pm.AvgPrice * 0.3) * pm.FragilityFactor
    )
    FROM DimProduct p
    INNER JOIN ProductMetrics pm ON p.ProductKey = pm.ProductKey;
END;
GO
```

### Automated Alerting

```sql
CREATE PROCEDURE sp_CheckFleetIdleAlerts
    @ThresholdPercent DECIMAL(5,2) = 15.0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Find trips with excessive idle time in last 24 hours
    SELECT 
        f.TripID,
        f.VehicleID,
        f.DriverID,
        og.LocationName AS Origin,
        dg.LocationName AS Destination,
        f.DurationMinutes,
        f.IdleTimeMinutes,
        CAST((f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) AS DECIMAL(5,2)) AS IdlePercent,
        f.FuelConsumedLiters,
        dt.FullDateTime AS TripTime
    FROM FactFleetTrips f
    INNER JOIN DimTime dt ON f.TimeKey = dt.TimeKey
    INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
    INNER JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey
    WHERE dt.FullDateTime >= DATEADD(HOUR, -24, GETDATE())
      AND (f.IdleTimeMinutes * 100.0 / NULLIF(f.DurationMinutes, 0)) > @ThresholdPercent
    ORDER BY IdlePercent DESC;
END;
GO
```

## Power BI DAX Measures

### Cross-Fact: Dwell Time per Fuel Consumed

```dax
Dwell Time per Liter = 
VAR TotalDwellMinutes = SUM(FactWarehouseOperations[DwellTimeMinutes])
VAR TotalFuelLiters = 
    CALCULATE(
        SUM(FactFleetTrips[FuelConsumedLiters]),
        USERELATIONSHIP(DimTime[TimeKey], FactFleetTrips[TimeKey])
    )
RETURN
    DIVIDE(TotalDwellMinutes, TotalFuelLiters, 0)
```

### Fleet Utilization Rate

```dax
Fleet Utilization % = 
VAR TotalDuration = SUM(FactFleetTrips[DurationMinutes])
VAR TotalIdle = SUM(FactFleetTrips[IdleTimeMinutes])
RETURN
    DIVIDE(TotalDuration - TotalIdle, TotalDuration, 0) * 100
```

### Warehouse Efficiency Score

```dax
Warehouse Efficiency = 
VAR ActualCycleTime = AVERAGE(FactWarehouseOperations[CycleTimeMinutes])
VAR BenchmarkCycleTime = 
    SWITCH(
        SELECTEDVALUE(FactWarehouseOperations[OperationType]),
        "Picking", 8,
        "Packing", 12,
        "Putaway", 10,
        15
    )
RETURN
    DIVIDE(BenchmarkCycleTime, ActualCycleTime, 0) * 100
```

### On-Time Delivery Rate

```dax
OTD % = 
VAR OnTime = CALCULATE(COUNTROWS(FactFleetTrips), FactFleetTrips[OnTimeDelivery] = TRUE())
VAR Total = COUNTROWS(FactFleetTrips)
RETURN
    DIVIDE(OnTime, Total, 0) * 100
```

## Common Patterns

### Pattern 1: Time-Intelligence Comparison

```dax
-- Compare current period to previous period
Dwell Time YoY = 
VAR CurrentPeriod = SUM(FactWarehouseOperations[DwellTimeMinutes])
VAR PreviousPeriod = 
    CALCULATE(
        SUM(FactWarehouseOperations[DwellTimeMinutes]),
        SAMEPERIODLASTYEAR(DimTime[Date])
    )
RETURN
    CurrentPeriod - PreviousPeriod
```

### Pattern 2: Dynamic Gravity Zone Assignment

```sql
-- Assign products to zones based on gravity score
WITH GravityRanks AS (
    SELECT 
        ProductKey,
        SKU,
        GravityScore,
        NTILE(5) OVER (ORDER BY GravityScore DESC) AS GravityTier
    FROM DimProduct
)
SELECT 
    SKU,
    CASE GravityTier
        WHEN 1 THEN 'Zone-A-High'
        WHEN 2 THEN 'Zone-B-MedHigh'
        WHEN 3 THEN 'Zone-C-Medium'
        WHEN 4 THEN 'Zone-D-MedLow'
        WHEN 5 THEN 'Zone-E-Low'
    END AS RecommendedZone,
    GravityScore
FROM GravityRanks
ORDER BY GravityScore DESC;
```

### Pattern 3: Route Optimization Query

```sql
-- Find most efficient routes by fuel efficiency
SELECT 
    og.LocationName AS Origin,
    dg.LocationName AS Destination,
    AVG(f.DistanceKM / NULLIF(f.FuelConsumedLiters, 0)) AS AvgKmPerLiter,
    AVG(f.DurationMinutes) AS AvgDurationMinutes,
    COUNT(*) AS TripCount,
    AVG(CAST(f.OnTimeDelivery AS INT)) * 100 AS OTDPercent
FROM FactFleetTrips f
INNER JOIN DimGeography og ON f.OriginGeographyKey = og.GeographyKey
INNER JOIN DimGeography dg ON f.DestinationGeographyKey = dg.GeographyKey
WHERE f.FuelConsumedLiters > 0
GROUP BY og.LocationName, dg.LocationName
HAVING COUNT(*) >= 10
ORDER BY AvgKmPerLiter DESC;
```

## Configuration Best Practices

### Row-Level Security (RLS)

Implement role-based access in Power BI:

```dax
-- Create role filter for regional managers
[Country] = USERNAME()

-- Or for warehouse supervisors
DimGeography[LocationID] IN {
    "WH-001", "WH-002", "WH-003"
}
```

Apply roles in Power BI Desktop:
1. Modeling tab → Manage Roles
2. Create role (e.g., "Regional_Manager")
3. Add filter to DimGeography table
4. Test with "View as" feature

### Incremental Refresh Setup

Configure incremental refresh for large fact tables:

1. Create RangeStart and RangeEnd parameters in Power Query:
```m
RangeStart = #datetime(2024, 1, 1, 0, 0, 0) meta [IsParameterQuery=true, Type="DateTime"]
RangeEnd = #datetime(2026, 12, 31, 23, 59, 59) meta [IsParameterQuery=true, Type="DateTime"]
```

2. Filter FactWarehouseOperations by date:
```m
= Table.SelectRows(
    Source, 
    each [OperationDate] >= RangeStart and [OperationDate] < RangeEnd
)
```

3. Set incremental refresh policy:
   - Archive data starting 2 years before refresh date
   - Incrementally refresh data for last 7 days

### Performance Optimization

```sql
-- Create columnstore index for analytical queries
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactWarehouseOps
ON FactWarehouseOperations (
    TimeKey, GeographyKey, ProductKey, OperationType, 
    QuantityHandled, DwellTimeMinutes, CycleTimeMinutes
);

-- Partition large fact tables by date
CREATE PARTITION FUNCTION PF_ByMonth (INT)
AS RANGE RIGHT FOR VALUES (
    20260101, 20260201, 20260301, 20260401, 20260501, 20260601
);

CREATE PARTITION SCHEME PS_ByMonth
AS PARTITION PF_ByMonth
ALL TO ([PRIMARY]);
```

## Troubleshooting

### Issue: Power BI Refresh Failure

**Symptom**: "Couldn't refresh the entity because of an issue with the mashup document"

**Solution**:
1. Check SQL Server connectivity from Power BI service
2. Verify gateway is online and configured
3. Test connection string in Power BI Desktop:
```m
let
    Source = Sql.Database("${SQL_SERVER_HOST}", "LogiFleetDB", [
        Query="SELECT TOP 10 * FROM FactWarehouseOperations"
    ])
in
    Source
```
4. Enable detailed error logging in gateway settings

### Issue: Slow Dashboard Performance

**Symptom**: Visuals take >10 seconds to load

**Solution**:
1. Check query folding in Power Query Editor (View Native Query)
2. Replace complex DAX with SQL views:
```sql
CREATE VIEW vw_WarehouseKPIs AS
SELECT 
    dt.[Date],
    dg.LocationName,
    SUM(f.QuantityHandled) AS TotalQuantity,
    AVG(f.DwellTimeMinutes) AS AvgDwellTime,
    AVG(f.CycleTimeMinutes) AS AvgCycleTime
FROM FactWarehouseOperations f
INNER JOIN DimTime dt ON f.TimeKey = dt.TimeKey
INNER JOIN DimGeography dg ON f.GeographyKey = dg.GeographyKey
GROUP BY dt.[Date], dg.LocationName;
```
3. Import view instead of querying fact table directly
4. Enable query diagnostics to identify bottlenecks

### Issue: Incorrect Cross-Fact Calculations

**Symptom**: Measures return BLANK or unexpected values

**Solution**:
1. Verify relationship cardinality and cross-filter direction
2. Use USERELATIONSHIP for role-playing dimensions:
```dax
Fleet Trips in Same Period = 
CALCULATE(
    COUNTROWS(FactFleetTrips),
    USERELATIONSHIP(DimTime[TimeKey], FactFleetTrips[TimeKey])
)
```
3. Check for ambiguous paths using DAX Studio
4. Add explicit CROSSFILTER statements if needed

### Issue: Missing Data in Gravity Score

**Symptom**: GravityScore column is NULL for many products

**Solution**:
```sql
-- Manually trigger gravity score calculation
EXEC sp_UpdateProductGravityScores;

-- Check for products with no picking activity
SELECT p.SKU, p.ProductName, p.GravityScore
FROM DimProduct p
LEFT JOIN FactWarehouseOperations f ON p.ProductKey = f.ProductKey AND f.OperationType = 'Picking'
WHERE f.OperationKey IS NULL;

-- Assign default gravity score to new products
UPDATE DimProduct
SET GravityScore = 50.0
WHERE GravityScore IS NULL;
```

## Advanced Analytics: Predictive Bottleneck Detection

Create a machine learning model in SQL Server ML Services:

```sql
-- Train model using historical data
CREATE PROCEDURE sp_TrainBottleneckModel
AS
BEGIN
    -- Feature engineering
    DROP TABLE IF EXISTS #MLTrainingData;
    
    SELECT 
        f.ProductKey,
        f.GeographyKey,
        DATEPART(HOUR, dt.FullDateTime) AS HourOfDay,
        DATEPART(WEEKDAY, dt.FullDateTime) AS DayOfWeek,
        AVG(f.DwellTimeMinutes) AS AvgDwellTime,
        AVG(f.CycleTimeMinutes) AS AvgCycleTime,
        COUNT(*) AS OperationCount,
        CASE 
            WHEN AVG(f.DwellTimeMinutes) > 60 THEN 1 
            ELSE 0 
        END AS IsBottleneck
    INTO #MLTrainingData
    FROM FactWarehouseOperations f
    INNER JOIN DimTime dt ON f.TimeKey = dt.TimeKey
    WHERE dt.[Date] >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY f.ProductKey, f.GeographyKey, 
             DATEPART(HOUR, dt.FullDateTime), 
             DATEPART(WEEKDAY, dt.FullDateTime);
    
    -- Train model (requires SQL Server ML Services)
    EXEC sp_execute_external_script
        @language = N'R',
        @script = N'
            model <- glm(IsBottleneck ~ ., data = InputDataSet, family = binomial)
            trained_model <- data.frame(payload = as.raw(serialize(model, connection=NULL)))
        ',
        @input_data_1 = N'SELECT * FROM #MLTrainingData',
        @output_data_1_name = N'trained_model';
END;
GO
```

## Integration with External Systems

### Weather API Integration

```sql
CREATE PROCEDURE sp_EnrichFleetDataWithWeather
    @ApiKey NVARCHAR(100)
AS
BEGIN
    -- Pseudo-code for external API call
    -- In practice, use Azure Functions or SSIS to call API and insert results
    
    DECLARE @WeatherData TABLE (
        TripKey BIGINT,
        Temperature DECIMAL(5,2),
        Precipitation DECIMAL(5,2),
        WindSpeed DECIMAL(5,2),
        Condition NVARCHAR(50)
    );
    
    -- Update FactFleetTrips with weather conditions
    UPDATE f
    SET f.WeatherCondition = w.Condition
    FROM FactFleetTrips f
    INNER JOIN @WeatherData w ON f.TripKey = w.TripKey;
END;
GO
```

## Deployment Checklist

- [ ] SQL Server instance provisioned with adequate resources (4+ cores, 16GB+ RAM)
- [ ] Database created with appropriate recovery model (FULL for production)
- [ ] All dimension tables populated with master data
- [ ] Initial fact table load completed successfully
- [ ] Indexes and columnstore indexes created
- [ ] Stored procedures tested with sample data
- [ ] Power BI template imported and data sources configured
- [ ] Row-level security roles defined and tested
- [ ] Incremental refresh policies configured
- [ ] Gateway installed and connected (for Power BI service)
- [ ] Scheduled refresh configured (every 15 minutes recommended)
- [ ] Alerts configured with appropriate thresholds
- [ ] Documentation shared with business users
- [ ] Backup strategy implemented

## Sample End-to-End Workflow

```sql
-- 1. Load new warehouse operations (runs every 15 minutes)
EXEC sp_LoadWarehouseOperations 
    @StartDateTime = DATEADD(MINUTE, -15, GETDATE()),
    @EndDateTime = GETDATE();

-- 2. Update gravity scores (runs daily)
EXEC sp_UpdateProductGravityScores;

-- 3. Check for alerts (runs hourly)
EXEC sp_CheckFleetIdleAlerts @ThresholdPercent = 15.0;

-- 4. Generate executive summary
SELECT 
    'Warehouse Operations' AS Category,
    COUNT(*) AS TotalOperations,
    AVG(CycleTimeMinutes) AS AvgCycleTime,
    SUM(OperationCost) AS TotalCost
FROM FactWarehouseOperations
WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime) -- Last 24 hours

UNION ALL

SELECT 
    'Fleet Trips' AS Category,
    COUNT(*) AS TotalTrips,
    AVG(DurationMinutes) AS AvgDuration,
    SUM(TripCost) AS TotalCost
FROM FactFleetTrips
WHERE TimeKey >= (SELECT MAX(TimeKey) - 96 FROM DimTime);
```

This skill enables AI agents to help developers deploy, configure, and optimize the LogiFleet Pulse supply chain analytics platform using MS SQL Server and Power BI.
