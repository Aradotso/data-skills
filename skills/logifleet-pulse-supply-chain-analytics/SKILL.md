---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI data warehousing platform for multi-modal supply chain logistics intelligence with cross-fact KPI harmonization
triggers:
  - "set up LogiFleet Pulse analytics"
  - "configure supply chain data warehouse"
  - "deploy logistics Power BI dashboards"
  - "implement warehouse gravity zones"
  - "create fleet telemetry analytics"
  - "build cross-modal logistics KPIs"
  - "setup supply chain star schema"
  - "integrate WMS with Power BI"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is an advanced data warehousing and business intelligence platform for supply chain and logistics operations. It provides:

- **Multi-fact star schema** linking warehouse operations, fleet telemetry, inventory, and external data sources
- **Cross-modal KPI harmonization** that correlates metrics across warehouse, fleet, and supplier dimensions
- **Real-time Power BI dashboards** with 15-minute refresh cycles
- **Warehouse Gravity Zones™** for optimizing storage based on pick frequency, value, and fragility
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Fleet triage engine** for proactive maintenance prioritization

The platform is built on MS SQL Server (2019+) with Power BI as the visualization layer, designed for 3PL operators, retail chains, and logistics-heavy organizations.

## Installation & Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Access to source systems: WMS, TMS, telematics APIs
- SQL Server Management Studio (SSMS) recommended

### Step 1: Clone Repository

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

### Step 2: Deploy SQL Schema

Connect to your SQL Server instance and execute the schema deployment:

```sql
-- Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute schema script (included in repository)
-- Run schema/01_create_tables.sql
-- Run schema/02_create_dimensions.sql
-- Run schema/03_create_facts.sql
-- Run schema/04_create_relationships.sql
-- Run schema/05_create_indexes.sql
-- Run schema/06_create_stored_procedures.sql
```

### Step 3: Configure Data Sources

Update `config_sample.json` with your connection details:

```json
{
  "connections": {
    "wms_api": {
      "endpoint": "${WMS_API_ENDPOINT}",
      "auth_token": "${WMS_API_TOKEN}"
    },
    "fleet_telemetry": {
      "endpoint": "${FLEET_TELEMETRY_ENDPOINT}",
      "api_key": "${FLEET_API_KEY}"
    },
    "erp_connection": {
      "server": "${ERP_SQL_SERVER}",
      "database": "${ERP_DATABASE}",
      "username": "${ERP_USERNAME}",
      "password": "${ERP_PASSWORD}"
    }
  },
  "refresh_interval_minutes": 15,
  "alert_thresholds": {
    "dwell_time_hours": 72,
    "fleet_idle_percentage": 15,
    "warehouse_capacity_percentage": 95
  }
}
```

### Step 4: Import Power BI Template

1. Open Power BI Desktop
2. File → Import → Power BI Template
3. Select `LogiFleet_Pulse_Master.pbit`
4. Enter SQL Server connection details when prompted
5. Configure row-level security roles

## Key SQL Schema Components

### Core Fact Tables

#### FactWarehouseOperations

```sql
-- Track warehouse micro-operations
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    LocationKey INT NOT NULL,
    OperationType VARCHAR(50) NOT NULL, -- 'Receiving', 'Putaway', 'Picking', 'Packing', 'Shipping'
    StartTimestamp DATETIME2 NOT NULL,
    EndTimestamp DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, StartTimestamp, EndTimestamp),
    QuantityHandled INT,
    EmployeeKey INT,
    EquipmentKey INT,
    DwellTimeHours DECIMAL(10,2),
    GravityZoneKey INT,
    CONSTRAINT FK_WarehouseOps_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_WarehouseOps_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey),
    CONSTRAINT FK_WarehouseOps_Location FOREIGN KEY (LocationKey) REFERENCES DimLocation(LocationKey)
);

-- Columnstore index for analytics
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_WarehouseOps 
ON FactWarehouseOperations (TimeKey, ProductKey, LocationKey, DurationMinutes, DwellTimeHours);
```

#### FactFleetTrips

```sql
-- Track fleet movements and telemetry
CREATE TABLE FactFleetTrips (
    TripKey BIGINT IDENTITY(1,1) PRIMARY KEY,
    TimeKey INT NOT NULL,
    VehicleKey INT NOT NULL,
    RouteKey INT NOT NULL,
    DriverKey INT NOT NULL,
    StartLocationKey INT,
    EndLocationKey INT,
    DepartureTimestamp DATETIME2,
    ArrivalTimestamp DATETIME2,
    TripDurationMinutes AS DATEDIFF(MINUTE, DepartureTimestamp, ArrivalTimestamp),
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,2),
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    CargoWeightLbs DECIMAL(10,2),
    WeatherConditionKey INT,
    MaintenanceAlertFlag BIT DEFAULT 0,
    CONSTRAINT FK_FleetTrips_Time FOREIGN KEY (TimeKey) REFERENCES DimTime(TimeKey),
    CONSTRAINT FK_FleetTrips_Vehicle FOREIGN KEY (VehicleKey) REFERENCES DimVehicle(VehicleKey)
);

CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FleetTrips 
ON FactFleetTrips (TimeKey, VehicleKey, RouteKey, IdleTimeMinutes, FuelConsumedGallons);
```

### Dimension Tables

#### DimTime (15-minute granularity)

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY,
    FullDateTime DATETIME2 NOT NULL,
    Year INT,
    Quarter INT,
    Month INT,
    Week INT,
    DayOfYear INT,
    DayOfMonth INT,
    DayOfWeek INT,
    Hour INT,
    Minute15Bucket INT, -- 0, 15, 30, 45
    IsWeekend BIT,
    FiscalPeriod INT,
    FiscalQuarter INT,
    ShiftType VARCHAR(20) -- 'Morning', 'Afternoon', 'Night'
);

-- Generate time dimension data
CREATE PROCEDURE sp_PopulateTimeDimension
    @StartDate DATETIME2,
    @EndDate DATETIME2
AS
BEGIN
    DECLARE @CurrentDateTime DATETIME2 = @StartDate;
    
    WHILE @CurrentDateTime <= @EndDate
    BEGIN
        INSERT INTO DimTime (TimeKey, FullDateTime, Year, Quarter, Month, Week, 
                             DayOfYear, DayOfMonth, DayOfWeek, Hour, Minute15Bucket, IsWeekend)
        VALUES (
            CAST(FORMAT(@CurrentDateTime, 'yyyyMMddHHmm') AS INT),
            @CurrentDateTime,
            YEAR(@CurrentDateTime),
            DATEPART(QUARTER, @CurrentDateTime),
            MONTH(@CurrentDateTime),
            DATEPART(WEEK, @CurrentDateTime),
            DATEPART(DAYOFYEAR, @CurrentDateTime),
            DAY(@CurrentDateTime),
            DATEPART(WEEKDAY, @CurrentDateTime),
            DATEPART(HOUR, @CurrentDateTime),
            (DATEPART(MINUTE, @CurrentDateTime) / 15) * 15,
            CASE WHEN DATEPART(WEEKDAY, @CurrentDateTime) IN (1, 7) THEN 1 ELSE 0 END
        );
        
        SET @CurrentDateTime = DATEADD(MINUTE, 15, @CurrentDateTime);
    END
END;
```

#### DimProductGravity (Warehouse Gravity Zones)

```sql
CREATE TABLE DimProductGravity (
    ProductGravityKey INT IDENTITY(1,1) PRIMARY KEY,
    ProductKey INT NOT NULL,
    GravityScore DECIMAL(5,2), -- Calculated score 0-100
    PickFrequencyRank INT,
    ValueDensity DECIMAL(10,2), -- Value per cubic foot
    FragilityIndex INT, -- 1-10 scale
    ReplenishmentLeadTimeDays INT,
    GravityZone VARCHAR(20), -- 'High', 'Medium', 'Low'
    RecommendedStorageDistance INT, -- Distance from shipping dock in feet
    LastRecalculatedDate DATETIME2,
    CONSTRAINT FK_ProductGravity_Product FOREIGN KEY (ProductKey) REFERENCES DimProduct(ProductKey)
);
```

## Stored Procedures & ETL Patterns

### Incremental Load Pattern

```sql
CREATE PROCEDURE sp_IncrementalLoadWarehouseOps
    @LastLoadTimestamp DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Load new warehouse operations from source system
    INSERT INTO FactWarehouseOperations (
        TimeKey, ProductKey, LocationKey, OperationType,
        StartTimestamp, EndTimestamp, QuantityHandled, 
        EmployeeKey, EquipmentKey, DwellTimeHours, GravityZoneKey
    )
    SELECT 
        CAST(FORMAT(wms.OperationStart, 'yyyyMMddHHmm') AS INT) AS TimeKey,
        p.ProductKey,
        l.LocationKey,
        wms.OperationType,
        wms.OperationStart,
        wms.OperationEnd,
        wms.Quantity,
        e.EmployeeKey,
        eq.EquipmentKey,
        DATEDIFF(HOUR, wms.OperationStart, wms.OperationEnd) AS DwellTimeHours,
        pg.ProductGravityKey
    FROM ExternalWMS.dbo.Operations wms
    INNER JOIN DimProduct p ON wms.SKU = p.SKU
    INNER JOIN DimLocation l ON wms.LocationCode = l.LocationCode
    LEFT JOIN DimEmployee e ON wms.EmployeeID = e.EmployeeID
    LEFT JOIN DimEquipment eq ON wms.EquipmentID = eq.EquipmentID
    LEFT JOIN DimProductGravity pg ON p.ProductKey = pg.ProductKey
    WHERE wms.OperationStart > @LastLoadTimestamp
        AND wms.OperationEnd IS NOT NULL;
    
    -- Update last load timestamp in control table
    UPDATE ETL_Control
    SET LastLoadTimestamp = GETDATE()
    WHERE TableName = 'FactWarehouseOperations';
END;
```

### Calculate Warehouse Gravity Zones

```sql
CREATE PROCEDURE sp_RecalculateGravityZones
AS
BEGIN
    -- Calculate gravity scores based on velocity, value, and fragility
    WITH ProductMetrics AS (
        SELECT 
            p.ProductKey,
            COUNT(DISTINCT wo.OperationKey) AS PickFrequency,
            AVG(p.UnitPrice / NULLIF(p.CubicFeet, 0)) AS ValueDensity,
            p.FragilityRating,
            AVG(s.LeadTimeDays) AS AvgLeadTime
        FROM DimProduct p
        LEFT JOIN FactWarehouseOperations wo 
            ON p.ProductKey = wo.ProductKey 
            AND wo.OperationType = 'Picking'
            AND wo.StartTimestamp >= DATEADD(DAY, -90, GETDATE())
        LEFT JOIN DimSupplier s ON p.SupplierKey = s.SupplierKey
        GROUP BY p.ProductKey, p.UnitPrice, p.CubicFeet, p.FragilityRating
    ),
    ScoredProducts AS (
        SELECT 
            ProductKey,
            PickFrequency,
            ValueDensity,
            FragilityRating,
            AvgLeadTime,
            -- Weighted gravity score calculation
            (
                (PickFrequency * 0.4) + 
                (ValueDensity * 0.3) + 
                (FragilityRating * 10 * 0.2) + 
                ((30 - AvgLeadTime) * 0.1)
            ) AS GravityScore,
            RANK() OVER (ORDER BY PickFrequency DESC) AS FrequencyRank
        FROM ProductMetrics
    )
    MERGE DimProductGravity AS target
    USING ScoredProducts AS source ON target.ProductKey = source.ProductKey
    WHEN MATCHED THEN
        UPDATE SET 
            GravityScore = source.GravityScore,
            PickFrequencyRank = source.FrequencyRank,
            ValueDensity = source.ValueDensity,
            FragilityIndex = source.FragilityRating,
            ReplenishmentLeadTimeDays = source.AvgLeadTime,
            GravityZone = CASE 
                WHEN source.GravityScore >= 70 THEN 'High'
                WHEN source.GravityScore >= 40 THEN 'Medium'
                ELSE 'Low'
            END,
            RecommendedStorageDistance = CASE
                WHEN source.GravityScore >= 70 THEN 50  -- 50 feet from dock
                WHEN source.GravityScore >= 40 THEN 150
                ELSE 300
            END,
            LastRecalculatedDate = GETDATE()
    WHEN NOT MATCHED THEN
        INSERT (ProductKey, GravityScore, PickFrequencyRank, ValueDensity, 
                FragilityIndex, ReplenishmentLeadTimeDays, GravityZone, 
                RecommendedStorageDistance, LastRecalculatedDate)
        VALUES (source.ProductKey, source.GravityScore, source.FrequencyRank,
                source.ValueDensity, source.FragilityRating, source.AvgLeadTime,
                CASE WHEN source.GravityScore >= 70 THEN 'High'
                     WHEN source.GravityScore >= 40 THEN 'Medium'
                     ELSE 'Low' END,
                CASE WHEN source.GravityScore >= 70 THEN 50
                     WHEN source.GravityScore >= 40 THEN 150
                     ELSE 300 END,
                GETDATE());
END;
```

### Cross-Fact KPI: Dwell Time vs Fleet Idling

```sql
CREATE VIEW vw_DwellTimeFleetImpact AS
SELECT 
    t.Year,
    t.Month,
    t.Week,
    p.ProductCategory,
    l.WarehouseName,
    -- Warehouse metrics
    AVG(wo.DwellTimeHours) AS AvgDwellTimeHours,
    SUM(wo.QuantityHandled) AS TotalUnitsHandled,
    -- Fleet metrics (for shipments originating from same warehouse/product)
    AVG(ft.IdleTimeMinutes) AS AvgFleetIdleMinutes,
    AVG(ft.FuelConsumedGallons) AS AvgFuelPerTrip,
    -- Cross-fact KPI
    (AVG(wo.DwellTimeHours) * AVG(ft.IdleTimeMinutes)) / NULLIF(AVG(ft.TripDurationMinutes), 0) 
        AS DwellIdleImpactRatio
FROM FactWarehouseOperations wo
INNER JOIN DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
INNER JOIN DimLocation l ON wo.LocationKey = l.LocationKey
LEFT JOIN FactFleetTrips ft 
    ON ft.StartLocationKey = wo.LocationKey
    AND CAST(ft.DepartureTimestamp AS DATE) = CAST(wo.EndTimestamp AS DATE)
WHERE wo.OperationType = 'Shipping'
    AND t.FullDateTime >= DATEADD(MONTH, -6, GETDATE())
GROUP BY t.Year, t.Month, t.Week, p.ProductCategory, l.WarehouseName;
```

### Automated Alerting System

```sql
CREATE PROCEDURE sp_GenerateLogisticsAlerts
AS
BEGIN
    DECLARE @AlertTable TABLE (
        AlertType VARCHAR(50),
        Severity VARCHAR(20),
        EntityID INT,
        EntityName VARCHAR(100),
        MetricValue DECIMAL(10,2),
        ThresholdValue DECIMAL(10,2),
        AlertMessage VARCHAR(500)
    );
    
    -- Alert 1: Excessive dwell time
    INSERT INTO @AlertTable
    SELECT 
        'DwellTime' AS AlertType,
        'High' AS Severity,
        p.ProductKey,
        p.ProductName,
        AVG(wo.DwellTimeHours) AS MetricValue,
        72 AS ThresholdValue,
        'Product "' + p.ProductName + '" has avg dwell time of ' + 
        CAST(CAST(AVG(wo.DwellTimeHours) AS INT) AS VARCHAR) + ' hours (threshold: 72h)'
    FROM FactWarehouseOperations wo
    INNER JOIN DimProduct p ON wo.ProductKey = p.ProductKey
    WHERE wo.StartTimestamp >= DATEADD(DAY, -7, GETDATE())
    GROUP BY p.ProductKey, p.ProductName
    HAVING AVG(wo.DwellTimeHours) > 72;
    
    -- Alert 2: High fleet idle percentage
    INSERT INTO @AlertTable
    SELECT 
        'FleetIdle' AS AlertType,
        'Medium' AS Severity,
        v.VehicleKey,
        v.VehicleID,
        (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.TripDurationMinutes), 0)) AS MetricValue,
        15 AS ThresholdValue,
        'Vehicle "' + v.VehicleID + '" idle time at ' + 
        CAST(CAST((SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.TripDurationMinutes), 0)) AS INT) AS VARCHAR) + 
        '% (threshold: 15%)'
    FROM FactFleetTrips ft
    INNER JOIN DimVehicle v ON ft.VehicleKey = v.VehicleKey
    WHERE ft.DepartureTimestamp >= DATEADD(DAY, -7, GETDATE())
    GROUP BY v.VehicleKey, v.VehicleID
    HAVING (SUM(ft.IdleTimeMinutes) * 100.0 / NULLIF(SUM(ft.TripDurationMinutes), 0)) > 15;
    
    -- Alert 3: Warehouse capacity warning
    INSERT INTO @AlertTable
    SELECT 
        'WarehouseCapacity' AS AlertType,
        'Critical' AS Severity,
        l.LocationKey,
        l.WarehouseName,
        (COUNT(DISTINCT wo.ProductKey) * 100.0 / NULLIF(l.MaxSKUCapacity, 0)) AS MetricValue,
        95 AS ThresholdValue,
        'Warehouse "' + l.WarehouseName + '" at ' + 
        CAST(CAST((COUNT(DISTINCT wo.ProductKey) * 100.0 / NULLIF(l.MaxSKUCapacity, 0)) AS INT) AS VARCHAR) + 
        '% capacity (threshold: 95%)'
    FROM FactWarehouseOperations wo
    INNER JOIN DimLocation l ON wo.LocationKey = l.LocationKey
    WHERE wo.StartTimestamp >= DATEADD(DAY, -1, GETDATE())
        AND l.LocationType = 'Warehouse'
    GROUP BY l.LocationKey, l.WarehouseName, l.MaxSKUCapacity
    HAVING (COUNT(DISTINCT wo.ProductKey) * 100.0 / NULLIF(l.MaxSKUCapacity, 0)) > 95;
    
    -- Insert alerts into tracking table
    INSERT INTO LogisticsAlerts (AlertDate, AlertType, Severity, EntityID, EntityName, 
                                  MetricValue, ThresholdValue, AlertMessage, IsAcknowledged)
    SELECT GETDATE(), AlertType, Severity, EntityID, EntityName, 
           MetricValue, ThresholdValue, AlertMessage, 0
    FROM @AlertTable;
    
    -- Return alerts for immediate notification
    SELECT * FROM @AlertTable;
END;
```

## Power BI DAX Measures

### Key Performance Indicators

```dax
// Average Dwell Time (Warehouse)
Avg Dwell Time = 
AVERAGE(FactWarehouseOperations[DwellTimeHours])

// Fleet Utilization Percentage
Fleet Utilization % = 
DIVIDE(
    SUM(FactFleetTrips[TripDurationMinutes]) - SUM(FactFleetTrips[IdleTimeMinutes]),
    SUM(FactFleetTrips[TripDurationMinutes]),
    0
) * 100

// Cost per Mile (Fleet)
Cost per Mile = 
DIVIDE(
    SUM(FactFleetTrips[FuelConsumedGallons]) * [Avg Fuel Price],
    SUM(FactFleetTrips[DistanceMiles]),
    0
)

// Warehouse Throughput (Units per Hour)
Warehouse Throughput = 
DIVIDE(
    SUM(FactWarehouseOperations[QuantityHandled]),
    SUM(FactWarehouseOperations[DurationMinutes]) / 60,
    0
)

// Cross-Modal KPI: Delivery Efficiency Score
Delivery Efficiency Score = 
VAR WarehouseScore = 100 - ([Avg Dwell Time] / 72 * 100)
VAR FleetScore = [Fleet Utilization %]
VAR OnTimeDelivery = DIVIDE(
    CALCULATE(COUNT(FactFleetTrips[TripKey]), FactFleetTrips[ArrivalTimestamp] <= FactFleetTrips[ScheduledArrival]),
    COUNT(FactFleetTrips[TripKey]),
    0
) * 100
RETURN
    (WarehouseScore * 0.3) + (FleetScore * 0.4) + (OnTimeDelivery * 0.3)

// Gravity Zone Optimization Score
Gravity Zone Score = 
VAR HighGravityInZone = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[GravityZone] = "High",
        DimLocation[DistanceFromDock] <= 50
    )
VAR TotalHighGravity = 
    CALCULATE(
        COUNTROWS(FactWarehouseOperations),
        DimProductGravity[GravityZone] = "High"
    )
RETURN
    DIVIDE(HighGravityInZone, TotalHighGravity, 0) * 100
```

### Time Intelligence Measures

```dax
// Year-over-Year Dwell Time Change
YoY Dwell Time Change = 
VAR CurrentPeriod = [Avg Dwell Time]
VAR PriorYear = 
    CALCULATE(
        [Avg Dwell Time],
        DATEADD(DimTime[FullDateTime], -1, YEAR)
    )
RETURN
    DIVIDE(CurrentPeriod - PriorYear, PriorYear, 0) * 100

// Rolling 7-Day Fleet Idle Percentage
Rolling 7D Idle % = 
CALCULATE(
    DIVIDE(
        SUM(FactFleetTrips[IdleTimeMinutes]),
        SUM(FactFleetTrips[TripDurationMinutes]),
        0
    ) * 100,
    DATESINPERIOD(
        DimTime[FullDateTime],
        LASTDATE(DimTime[FullDateTime]),
        -7,
        DAY
    )
)
```

## Common Integration Patterns

### Polybase External Table (Azure SQL/Synapse)

```sql
-- Create external data source for WMS API results stored in Azure Blob
CREATE EXTERNAL DATA SOURCE WMS_BlobStorage
WITH (
    TYPE = BLOB_STORAGE,
    LOCATION = 'https://${AZURE_STORAGE_ACCOUNT}.blob.core.windows.net/wms-exports'
);

-- Create external file format
CREATE EXTERNAL FILE FORMAT CSVFormat
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS (
        FIELD_TERMINATOR = ',',
        STRING_DELIMITER = '"',
        FIRST_ROW = 2
    )
);

-- Create external table
CREATE EXTERNAL TABLE ExternalWMS_Operations
(
    OperationID INT,
    OperationType VARCHAR(50),
    SKU VARCHAR(50),
    LocationCode VARCHAR(20),
    OperationStart DATETIME2,
    OperationEnd DATETIME2,
    Quantity INT,
    EmployeeID VARCHAR(20)
)
WITH (
    LOCATION = '/warehouse-ops/',
    DATA_SOURCE = WMS_BlobStorage,
    FILE_FORMAT = CSVFormat
);
```

### REST API Integration with Stored Procedure

```sql
-- Enable Ole Automation Procedures (run as admin)
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'Ole Automation Procedures', 1;
RECONFIGURE;

-- Procedure to fetch fleet telemetry via REST API
CREATE PROCEDURE sp_FetchFleetTelemetry
    @VehicleID VARCHAR(50)
AS
BEGIN
    DECLARE @url VARCHAR(500) = '${FLEET_API_ENDPOINT}/vehicles/' + @VehicleID + '/telemetry';
    DECLARE @authHeader VARCHAR(500) = 'Bearer ${FLEET_API_TOKEN}';
    DECLARE @response VARCHAR(MAX);
    DECLARE @obj INT;
    
    EXEC sp_OACreate 'MSXML2.ServerXMLHTTP', @obj OUT;
    EXEC sp_OAMethod @obj, 'open', NULL, 'GET', @url, false;
    EXEC sp_OAMethod @obj, 'setRequestHeader', NULL, 'Authorization', @authHeader;
    EXEC sp_OAMethod @obj, 'send';
    EXEC sp_OAGetProperty @obj, 'responseText', @response OUT;
    
    -- Parse JSON response and insert into staging table
    INSERT INTO StagingFleetTelemetry (VehicleID, TelemetryJSON, ReceivedTimestamp)
    VALUES (@VehicleID, @response, GETDATE());
    
    EXEC sp_OADestroy @obj;
END;
```

## Row-Level Security Configuration

```sql
-- Create security predicates for role-based access
CREATE SCHEMA Security;
GO

-- Security function for warehouse managers
CREATE FUNCTION Security.fn_WarehouseSecurityPredicate(@LocationKey INT)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN (
    SELECT 1 AS AccessGranted
    FROM dbo.DimLocation l
    INNER JOIN dbo.UserWarehouseAccess uwa 
        ON l.LocationKey = uwa.LocationKey
    WHERE l.LocationKey = @LocationKey
        AND uwa.Username = USER_NAME()
);
GO

-- Apply security policy to warehouse operations
CREATE SECURITY POLICY WarehouseAccessPolicy
ADD FILTER PREDICATE Security.fn_WarehouseSecurityPredicate(LocationKey)
ON dbo.FactWarehouseOperations
WITH (STATE = ON);
```

## Troubleshooting

### Issue: Power BI Refresh Timeout

**Symptom**: Power BI dataset refresh fails with timeout errors

**Solution**: Implement incremental refresh in Power BI
1. Add RangeStart and RangeEnd parameters in Power Query
2. Filter fact tables using these parameters
3. Configure incremental refresh policy in dataset settings

```m
// Power Query M code for incremental refresh
let
    Source = Sql.Database("${SQL_SERVER}", "${DATABASE}"),
    FilteredRows = Table.SelectRows(
        Source, 
        each [StartTimestamp] >= RangeStart and [StartTimestamp] < RangeEnd
    )
in
    FilteredRows
```

### Issue: Slow Cross-Fact Queries

**Symptom**: Views joining multiple fact tables perform poorly

**Solution**: Create indexed materialized views

```sql
CREATE VIEW vw_MaterializedCrossFact
WITH SCHEMABINDING
AS
SELECT 
    t.TimeKey,
    l.LocationKey,
    p.ProductKey,
    SUM(wo.DwellTimeHours) AS TotalDwellTime,
    COUNT_BIG(*) AS OperationCount,
    AVG(ft.IdleTimeMinutes) AS AvgIdleTime
FROM dbo.FactWarehouseOperations wo
INNER JOIN dbo.DimTime t ON wo.TimeKey = t.TimeKey
INNER JOIN dbo.DimLocation l ON wo.LocationKey = l.LocationKey
INNER JOIN dbo.DimProduct p ON wo.ProductKey = p.ProductKey
LEFT JOIN dbo.FactFleetTrips ft ON ft.StartLocationKey = l.LocationKey
GROUP BY t.TimeKey, l.LocationKey, p.ProductKey;

-- Create unique clustered index to materialize
CREATE UNIQUE CLUSTERED INDEX IX_MaterializedCrossFact
ON vw_MaterializedCrossFact (TimeKey, LocationKey, ProductKey);
```

### Issue: Gravity Zone Calculation Performance

**Symptom**: `sp_RecalculateGravityZones` takes too long

**Solution**: Partition by product category and run in parallel

```sql
-- Create partition function
CREATE PARTITION FUNCTION pf_ProductCategory (VARCHAR(50))
AS RANGE RIGHT FOR VALUES ('Electronics', 'Apparel', 'Food', 'Industrial');

-- Apply to DimProductGravity
ALTER TABLE DimProductGravity
ADD ProductCategory VARCHAR(50);

-- Run calculation by partition
EXEC sp_RecalculateGravityZones @ProductCategory = 'Electronics';
EXEC sp_RecalculateGravityZones @ProductCategory = 'Apparel';
-- etc.
```

### Issue: Missing External Data Source Credentials

**Symptom**: External table queries fail with authentication errors

**Solution**: Create database scoped credential

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '${MASTER_KEY_PASSWORD}';

CREATE DATABASE SCOPED CREDENTIAL AzureBlobCredential
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = '${
