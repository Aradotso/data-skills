---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server and Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - set up logifleet pulse supply chain analytics
  - deploy logistics data warehouse with power bi
  - configure multi-fact star schema for fleet management
  - integrate warehouse and fleet data in sql server
  - create supply chain analytics dashboard
  - implement logicore logistics intelligence
  - build cross-modal supply chain reporting
  - optimize warehouse and fleet operations with powerbi
---

# LogiFleet Pulse Supply Chain Analytics

> Skill by [ara.so](https://ara.so) — Data Skills collection

LogiFleet Pulse is an advanced MS SQL Server and Power BI-based logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data sources into a single semantic layer. It uses a multi-fact star schema architecture with time-phased dimensions to enable cross-functional KPI analysis and predictive bottleneck detection.

## Core Capabilities

- **Multi-fact star schema** linking warehouse operations, fleet trips, cross-dock transfers, and external data
- **Time-phased dimensions** with 15-minute granularity for real-time operational visibility
- **Cross-fact KPI harmonization** connecting inventory metrics with fleet performance
- **Warehouse Gravity Zones** for spatial optimization based on pick frequency and item characteristics
- **Predictive bottleneck detection** using temporal elasticity modeling
- **Power BI dashboards** with role-based access and natural language querying

## Installation and Setup

### Prerequisites

- MS SQL Server 2019 or later
- Power BI Desktop (latest version)
- Administrative access to deploy database schema
- Source system credentials (WMS, TMS, telemetry APIs)

### Database Deployment

1. **Clone the repository:**

```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy the SQL schema:**

```sql
-- Connect to your SQL Server instance
-- Execute the main schema script
USE master;
GO

CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Execute the schema creation scripts in order:
-- 1. Create dimension tables
-- 2. Create fact tables
-- 3. Create relationships and indexes
-- 4. Create views and stored procedures
```

3. **Configure data source connections:**

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
  }
}
```

## Data Model Architecture

### Dimension Tables

**DimTime** - 15-minute granularity time dimension:

```sql
CREATE TABLE DimTime (
    TimeKey INT PRIMARY KEY IDENTITY(1,1),
    FullDateTime DATETIME2 NOT NULL,
    Date DATE NOT NULL,
    TimeOfDay TIME NOT NULL,
    HourOfDay INT NOT NULL,
    QuarterHour INT NOT NULL, -- 0-3 for each hour
    DayOfWeek INT NOT NULL,
    DayName VARCHAR(20) NOT NULL,
    WeekOfYear INT NOT NULL,
    MonthNumber INT NOT NULL,
    MonthName VARCHAR(20) NOT NULL,
    Quarter INT NOT NULL,
    FiscalYear INT NOT NULL,
    FiscalPeriod INT NOT NULL,
    IsWeekend BIT NOT NULL,
    IsHoliday BIT NOT NULL
);

-- Index for optimal join performance
CREATE CLUSTERED INDEX IX_DimTime_FullDateTime ON DimTime(FullDateTime);
CREATE NONCLUSTERED INDEX IX_DimTime_Date ON DimTime(Date);
```

**DimGeography** - Hierarchical location dimension:

```sql
CREATE TABLE DimGeography (
    GeographyKey INT PRIMARY KEY IDENTITY(1,1),
    LocationID VARCHAR(50) NOT NULL UNIQUE,
    LocationName VARCHAR(200) NOT NULL,
    LocationType VARCHAR(50) NOT NULL, -- Warehouse, RouteNode, CrossDock
    AddressLine1 VARCHAR(200),
    City VARCHAR(100) NOT NULL,
    StateProvince VARCHAR(100),
    PostalCode VARCHAR(20),
    Country VARCHAR(100) NOT NULL,
    Region VARCHAR(100) NOT NULL,
    Continent VARCHAR(50) NOT NULL,
    Latitude DECIMAL(9,6),
    Longitude DECIMAL(9,6),
    TimeZone VARCHAR(50),
    IsActive BIT NOT NULL DEFAULT 1
);
```

**DimProductGravity** - Product classification with gravity scoring:

```sql
CREATE TABLE DimProductGravity (
    ProductKey INT PRIMARY KEY IDENTITY(1,1),
    SKU VARCHAR(100) NOT NULL UNIQUE,
    ProductName VARCHAR(500) NOT NULL,
    Category VARCHAR(100),
    SubCategory VARCHAR(100),
    UnitWeight DECIMAL(10,3),
    UnitVolume DECIMAL(10,3),
    IsFragile BIT NOT NULL DEFAULT 0,
    IsPerishable BIT NOT NULL DEFAULT 0,
    ShelfLifeDays INT,
    AveragePickFrequency DECIMAL(10,2), -- Updated nightly
    AverageValue DECIMAL(15,2),
    ReplenishmentLeadTimeDays INT,
    GravityScore DECIMAL(5,2), -- Calculated: (PickFrequency * Value) / (Weight * LeadTime)
    RecommendedZoneType VARCHAR(50), -- HighGravity, MediumGravity, LowGravity
    LastUpdated DATETIME2 NOT NULL DEFAULT GETDATE()
);
```

**DimSupplierReliability** - Supplier performance metrics:

```sql
CREATE TABLE DimSupplierReliability (
    SupplierKey INT PRIMARY KEY IDENTITY(1,1),
    SupplierID VARCHAR(50) NOT NULL UNIQUE,
    SupplierName VARCHAR(200) NOT NULL,
    Country VARCHAR(100),
    AverageLeadTimeDays DECIMAL(5,1),
    LeadTimeVariance DECIMAL(5,2),
    DefectRate DECIMAL(5,4), -- Percentage
    OnTimeDeliveryRate DECIMAL(5,4), -- Percentage
    ComplianceScore DECIMAL(5,2), -- 0-100
    RiskCategory VARCHAR(20), -- Low, Medium, High
    LastAssessmentDate DATE NOT NULL
);
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:

```sql
CREATE TABLE FactWarehouseOperations (
    OperationKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    GeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    OperationType VARCHAR(50) NOT NULL, -- Receiving, Putaway, Picking, Packing, Shipping
    OperationID VARCHAR(100) NOT NULL,
    OrderID VARCHAR(100),
    Quantity INT NOT NULL,
    UnitOfMeasure VARCHAR(20),
    OperationStartTime DATETIME2 NOT NULL,
    OperationEndTime DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, OperationStartTime, OperationEndTime) PERSISTED,
    EmployeeID VARCHAR(50),
    EquipmentID VARCHAR(50), -- Forklift, picker, etc.
    StorageZone VARCHAR(50),
    ZoneGravityType VARCHAR(50),
    DwellTimeHours DECIMAL(10,2), -- For inventory items
    QualityCheckPassed BIT,
    ExceptionOccurred BIT DEFAULT 0,
    ExceptionType VARCHAR(100),
    Notes VARCHAR(MAX)
);

-- Partitioning by date for performance
CREATE PARTITION FUNCTION PF_WarehouseOps_Date (DATE)
AS RANGE RIGHT FOR VALUES 
('2026-01-01', '2026-02-01', '2026-03-01', '2026-04-01', '2026-05-01', 
 '2026-06-01', '2026-07-01', '2026-08-01', '2026-09-01', '2026-10-01', 
 '2026-11-01', '2026-12-01');

CREATE PARTITION SCHEME PS_WarehouseOps_Date
AS PARTITION PF_WarehouseOps_Date ALL TO ([PRIMARY]);
```

**FactFleetTrips** - Fleet telemetry and performance:

```sql
CREATE TABLE FactFleetTrips (
    TripKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    OriginGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    DestinationGeographyKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    VehicleID VARCHAR(50) NOT NULL,
    DriverID VARCHAR(50) NOT NULL,
    TripID VARCHAR(100) NOT NULL UNIQUE,
    RouteID VARCHAR(100),
    DepartureTime DATETIME2 NOT NULL,
    ArrivalTime DATETIME2,
    PlannedDurationMinutes INT,
    ActualDurationMinutes AS DATEDIFF(MINUTE, DepartureTime, ArrivalTime) PERSISTED,
    DistanceMiles DECIMAL(10,2),
    FuelConsumedGallons DECIMAL(10,3),
    FuelEfficiencyMPG AS (DistanceMiles / NULLIF(FuelConsumedGallons, 0)) PERSISTED,
    IdleTimeMinutes DECIMAL(10,2),
    LoadingTimeMinutes DECIMAL(10,2),
    UnloadingTimeMinutes DECIMAL(10,2),
    LoadWeightLbs DECIMAL(10,2),
    LoadValueUSD DECIMAL(15,2),
    WeatherCondition VARCHAR(50),
    TrafficDelayMinutes DECIMAL(10,2),
    MaintenanceIssueDetected BIT DEFAULT 0,
    MaintenanceIssueSeverity VARCHAR(20), -- Low, Medium, High, Critical
    OnTimeDelivery BIT,
    CustomerSignatureReceived BIT,
    ExceptionNotes VARCHAR(MAX)
);
```

**FactCrossDock** - Cross-dock transfer operations:

```sql
CREATE TABLE FactCrossDock (
    CrossDockKey BIGINT PRIMARY KEY IDENTITY(1,1),
    TimeKey INT NOT NULL FOREIGN KEY REFERENCES DimTime(TimeKey),
    CrossDockFacilityKey INT NOT NULL FOREIGN KEY REFERENCES DimGeography(GeographyKey),
    ProductKey INT NOT NULL FOREIGN KEY REFERENCES DimProductGravity(ProductKey),
    InboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    OutboundTripKey BIGINT FOREIGN KEY REFERENCES FactFleetTrips(TripKey),
    CrossDockID VARCHAR(100) NOT NULL,
    InboundArrivalTime DATETIME2 NOT NULL,
    OutboundDepartureTime DATETIME2,
    DwellTimeMinutes AS DATEDIFF(MINUTE, InboundArrivalTime, OutboundDepartureTime) PERSISTED,
    Quantity INT NOT NULL,
    InspectionRequired BIT,
    InspectionPassed BIT,
    TemperatureCompliant BIT, -- For perishables
    ActualTempFahrenheit DECIMAL(5,2),
    RepackagingRequired BIT,
    UrgencyLevel VARCHAR(20) -- Standard, Expedited, Critical
);
```

## Key Stored Procedures

### Incremental Data Load

```sql
CREATE PROCEDURE usp_LoadWarehouseOperationsIncremental
    @LastLoadDateTime DATETIME2,
    @CurrentLoadDateTime DATETIME2
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- Load from staging table populated by ETL process
        INSERT INTO FactWarehouseOperations (
            TimeKey, GeographyKey, ProductKey, OperationType, OperationID,
            OrderID, Quantity, UnitOfMeasure, OperationStartTime, OperationEndTime,
            EmployeeID, EquipmentID, StorageZone, ZoneGravityType, DwellTimeHours,
            QualityCheckPassed, ExceptionOccurred, ExceptionType, Notes
        )
        SELECT 
            dt.TimeKey,
            dg.GeographyKey,
            dp.ProductKey,
            stg.OperationType,
            stg.OperationID,
            stg.OrderID,
            stg.Quantity,
            stg.UnitOfMeasure,
            stg.OperationStartTime,
            stg.OperationEndTime,
            stg.EmployeeID,
            stg.EquipmentID,
            stg.StorageZone,
            stg.ZoneGravityType,
            stg.DwellTimeHours,
            stg.QualityCheckPassed,
            stg.ExceptionOccurred,
            stg.ExceptionType,
            stg.Notes
        FROM StagingWarehouseOperations stg
        INNER JOIN DimTime dt ON CAST(stg.OperationStartTime AS DATETIME2) = dt.FullDateTime
        INNER JOIN DimGeography dg ON stg.LocationID = dg.LocationID
        INNER JOIN DimProductGravity dp ON stg.SKU = dp.SKU
        WHERE stg.OperationStartTime >= @LastLoadDateTime
          AND stg.OperationStartTime < @CurrentLoadDateTime
          AND stg.IsProcessed = 0;
        
        -- Mark staging records as processed
        UPDATE StagingWarehouseOperations
        SET IsProcessed = 1, ProcessedDateTime = GETDATE()
        WHERE OperationStartTime >= @LastLoadDateTime
          AND OperationStartTime < @CurrentLoadDateTime;
        
        COMMIT TRANSACTION;
        
        PRINT 'Incremental load completed successfully.';
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        
        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH
END;
GO
```

### Gravity Score Calculation

```sql
CREATE PROCEDURE usp_UpdateProductGravityScores
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Calculate gravity scores based on recent activity
    WITH RecentActivity AS (
        SELECT 
            ProductKey,
            COUNT(*) AS PickCount,
            AVG(CAST(DurationMinutes AS DECIMAL(10,2))) AS AvgPickTime
        FROM FactWarehouseOperations
        WHERE OperationType = 'Picking'
          AND OperationStartTime >= DATEADD(DAY, -30, GETDATE())
        GROUP BY ProductKey
    ),
    ProductMetrics AS (
        SELECT 
            dp.ProductKey,
            dp.SKU,
            ISNULL(ra.PickCount, 0) / 30.0 AS DailyPickFrequency,
            dp.AverageValue,
            dp.UnitWeight,
            dp.ReplenishmentLeadTimeDays,
            dp.IsFragile,
            dp.IsPerishable
        FROM DimProductGravity dp
        LEFT JOIN RecentActivity ra ON dp.ProductKey = ra.ProductKey
    )
    UPDATE dp
    SET 
        AveragePickFrequency = pm.DailyPickFrequency,
        GravityScore = CASE 
            WHEN pm.UnitWeight > 0 AND pm.ReplenishmentLeadTimeDays > 0 THEN
                ((pm.DailyPickFrequency * pm.AverageValue) / (pm.UnitWeight * pm.ReplenishmentLeadTimeDays))
                * CASE WHEN pm.IsFragile = 1 THEN 1.5 ELSE 1.0 END
                * CASE WHEN pm.IsPerishable = 1 THEN 1.3 ELSE 1.0 END
            ELSE 0
        END,
        RecommendedZoneType = CASE 
            WHEN ((pm.DailyPickFrequency * pm.AverageValue) / NULLIF(pm.UnitWeight * pm.ReplenishmentLeadTimeDays, 0)) > 50 THEN 'HighGravity'
            WHEN ((pm.DailyPickFrequency * pm.AverageValue) / NULLIF(pm.UnitWeight * pm.ReplenishmentLeadTimeDays, 0)) BETWEEN 10 AND 50 THEN 'MediumGravity'
            ELSE 'LowGravity'
        END,
        LastUpdated = GETDATE()
    FROM DimProductGravity dp
    INNER JOIN ProductMetrics pm ON dp.ProductKey = pm.ProductKey;
    
    PRINT 'Gravity scores updated successfully.';
END;
GO
```

### Automated Alerting

```sql
CREATE PROCEDURE usp_GenerateLogisticsAlerts
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Create temp table for alerts
    CREATE TABLE #Alerts (
        AlertType VARCHAR(100),
        Severity VARCHAR(20),
        EntityID VARCHAR(100),
        Message VARCHAR(MAX),
        Value DECIMAL(18,2),
        Threshold DECIMAL(18,2)
    );
    
    -- Alert 1: High fleet idling time
    INSERT INTO #Alerts
    SELECT 
        'FleetIdleTime' AS AlertType,
        'Medium' AS Severity,
        VehicleID AS EntityID,
        'Vehicle ' + VehicleID + ' has excessive idle time: ' + 
            CAST(ROUND(AVG(IdleTimeMinutes), 1) AS VARCHAR) + ' minutes average in last 7 days' AS Message,
        AVG(IdleTimeMinutes) AS Value,
        15.0 AS Threshold
    FROM FactFleetTrips
    WHERE DepartureTime >= DATEADD(DAY, -7, GETDATE())
      AND ActualDurationMinutes > 0
    GROUP BY VehicleID
    HAVING (AVG(IdleTimeMinutes) / AVG(ActualDurationMinutes)) * 100 > 15;
    
    -- Alert 2: Excessive warehouse dwell time
    INSERT INTO #Alerts
    SELECT 
        'WarehouseDwellTime' AS AlertType,
        CASE WHEN AVG(DwellTimeHours) > 120 THEN 'High' ELSE 'Medium' END AS Severity,
        dp.SKU AS EntityID,
        'SKU ' + dp.SKU + ' (' + dp.ProductName + ') has excessive dwell time: ' + 
            CAST(ROUND(AVG(wo.DwellTimeHours), 1) AS VARCHAR) + ' hours average' AS Message,
        AVG(wo.DwellTimeHours) AS Value,
        72.0 AS Threshold
    FROM FactWarehouseOperations wo
    INNER JOIN DimProductGravity dp ON wo.ProductKey = dp.ProductKey
    WHERE wo.OperationType = 'Putaway'
      AND wo.OperationStartTime >= DATEADD(DAY, -14, GETDATE())
      AND wo.DwellTimeHours IS NOT NULL
    GROUP BY dp.SKU, dp.ProductName
    HAVING AVG(wo.DwellTimeHours) > 72;
    
    -- Alert 3: Critical maintenance issues
    INSERT INTO #Alerts
    SELECT 
        'FleetMaintenance' AS AlertType,
        'High' AS Severity,
        VehicleID AS EntityID,
        'Vehicle ' + VehicleID + ' has critical maintenance issues detected in ' + 
            CAST(COUNT(*) AS VARCHAR) + ' recent trips' AS Message,
        COUNT(*) AS Value,
        1.0 AS Threshold
    FROM FactFleetTrips
    WHERE DepartureTime >= DATEADD(DAY, -3, GETDATE())
      AND MaintenanceIssueDetected = 1
      AND MaintenanceIssueSeverity = 'Critical'
    GROUP BY VehicleID
    HAVING COUNT(*) >= 1;
    
    -- Alert 4: Low on-time delivery rate
    INSERT INTO #Alerts
    SELECT 
        'OnTimeDelivery' AS AlertType,
        'Medium' AS Severity,
        DriverID AS EntityID,
        'Driver ' + DriverID + ' has low on-time delivery rate: ' + 
            CAST(ROUND(AVG(CAST(OnTimeDelivery AS DECIMAL(5,2))) * 100, 1) AS VARCHAR) + '%' AS Message,
        AVG(CAST(OnTimeDelivery AS DECIMAL(5,2))) * 100 AS Value,
        85.0 AS Threshold
    FROM FactFleetTrips
    WHERE DepartureTime >= DATEADD(DAY, -30, GETDATE())
      AND ArrivalTime IS NOT NULL
    GROUP BY DriverID
    HAVING COUNT(*) >= 10 -- Minimum trips for statistical relevance
       AND AVG(CAST(OnTimeDelivery AS DECIMAL(5,2))) < 0.85;
    
    -- Insert into permanent alerts table with notification flag
    INSERT INTO LogisticsAlerts (AlertType, Severity, EntityID, Message, Value, Threshold, GeneratedDateTime, NotificationSent)
    SELECT AlertType, Severity, EntityID, Message, Value, Threshold, GETDATE(), 0
    FROM #Alerts;
    
    -- Return summary
    SELECT 
        Severity,
        COUNT(*) AS AlertCount
    FROM #Alerts
    GROUP BY Severity
    ORDER BY CASE Severity WHEN 'High' THEN 1 WHEN 'Medium' THEN 2 ELSE 3 END;
    
    DROP TABLE #Alerts;
END;
GO
```

## Power BI Configuration

### Connecting to SQL Server

1. **Open Power BI Desktop** and load the `.pbit` template file

2. **Configure data source connection:**

```
Data Source Settings:
  Server: ${SQL_SERVER_HOSTNAME}
  Database: LogiFleetPulse
  Data Connectivity mode: DirectQuery (recommended for real-time) or Import
  
Authentication:
  Windows Authentication (if domain-joined)
  OR
  SQL Server Authentication:
    Username: ${SQL_USERNAME}
    Password: ${SQL_PASSWORD}
```

3. **Set refresh schedule** (for Import mode):

```
Power BI Service Settings:
  Scheduled refresh: Every 15 minutes (requires Pro license)
  Gateway: On-premises data gateway (if SQL Server not publicly accessible)
```

### Key DAX Measures

**Cross-Fact KPI: Dwell Time Impact on Fleet Efficiency**

```dax
DwellTime_FleetEfficiency_Correlation = 
VAR AvgDwellByProduct = 
    SUMMARIZE(
        FactWarehouseOperations,
        DimProductGravity[SKU],
        "AvgDwell", AVERAGE(FactWarehouseOperations[DwellTimeHours])
    )
VAR FleetEffByProduct =
    SUMMARIZE(
        RELATEDTABLE(FactFleetTrips),
        DimProductGravity[SKU],
        "AvgIdlePct", DIVIDE(
            AVERAGE(FactFleetTrips[IdleTimeMinutes]),
            AVERAGE(FactFleetTrips[ActualDurationMinutes]),
            0
        )
    )
VAR JoinedData =
    NATURALLEFTOUTERJOIN(AvgDwellByProduct, FleetEffByProduct)
RETURN
    // Pearson correlation coefficient
    DIVIDE(
        SUMX(JoinedData, ([AvgDwell] - AVERAGE(AvgDwellByProduct[AvgDwell])) * ([AvgIdlePct] - AVERAGE(FleetEffByProduct[AvgIdlePct]))),
        SQRT(SUMX(JoinedData, ([AvgDwell] - AVERAGE(AvgDwellByProduct[AvgDwell]))^2)) * SQRT(SUMX(JoinedData, ([AvgIdlePct] - AVERAGE(FleetEffByProduct[AvgIdlePct]))^2)),
        BLANK()
    )
```

**Predictive Bottleneck Index**

```dax
Bottleneck_Risk_Score = 
VAR CurrentCapacity = 
    DIVIDE(
        CALCULATE(COUNT(FactWarehouseOperations[OperationKey]), FactWarehouseOperations[OperationType] = "Putaway"),
        [Max_Warehouse_Capacity],
        0
    )
VAR FleetUtilization =
    DIVIDE(
        CALCULATE(SUM(FactFleetTrips[ActualDurationMinutes])),
        CALCULATE(SUM(FactFleetTrips[PlannedDurationMinutes])),
        0
    )
VAR SupplierRisk =
    AVERAGE(DimSupplierReliability[RiskCategory])
VAR WeightedScore =
    (CurrentCapacity * 0.4) + 
    (FleetUtilization * 0.35) +
    (SupplierRisk * 0.25)
RETURN
    SWITCH(
        TRUE(),
        WeightedScore > 0.8, "Critical",
        WeightedScore > 0.6, "High",
        WeightedScore > 0.4, "Medium",
        "Low"
    )
```

**Warehouse Gravity Zone Efficiency**

```dax
Gravity_Zone_Efficiency = 
VAR HighGravityPicks =
    CALCULATE(
        COUNT(FactWarehouseOperations[OperationKey]),
        FactWarehouseOperations[ZoneGravityType] = "HighGravity",
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR HighGravityAvgTime =
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DurationMinutes]),
        FactWarehouseOperations[ZoneGravityType] = "HighGravity",
        FactWarehouseOperations[OperationType] = "Picking"
    )
VAR LowGravityAvgTime =
    CALCULATE(
        AVERAGE(FactWarehouseOperations[DurationMinutes]),
        FactWarehouseOperations[ZoneGravityType] = "LowGravity",
        FactWarehouseOperations[OperationType] = "Picking"
    )
RETURN
    // Efficiency = ratio of high-gravity speed advantage
    DIVIDE(LowGravityAvgTime - HighGravityAvgTime, LowGravityAvgTime, 0)
```

### Row-Level Security (RLS)

```dax
-- Create role: WarehouseManager_Location
[Role: WarehouseManager_Location]
DAX Filter: DimGeography[LocationID] = USERPRINCIPALNAME()

-- Create role: FleetManager_Region
[Role: FleetManager_Region]
DAX Filter: DimGeography[Region] IN { 
    LOOKUPVALUE(
        UserRegionMapping[Region], 
        UserRegionMapping[UserEmail], 
        USERPRINCIPALNAME()
    )
}

-- Create role: Executive_AllAccess
[Role: Executive_AllAccess]
DAX Filter: 1=1  -- No restrictions
```

## Common Integration Patterns

### Real-Time Weather Correlation

```sql
-- Create external table for weather API data
CREATE EXTERNAL DATA SOURCE WeatherAPI
WITH (
    TYPE = REST,
    LOCATION = '${WEATHER_API_ENDPOINT}'
);

-- Stored procedure to enrich fleet trips with weather data
CREATE PROCEDURE usp_EnrichFleetTripsWithWeather
AS
BEGIN
    UPDATE ft
    SET WeatherCondition = wd.Condition
    FROM FactFleetTrips ft
    INNER JOIN ExternalWeatherData wd 
        ON CAST(ft.DepartureTime AS DATE) = wd.Date
        AND ft.OriginGeographyKey = wd.GeographyKey
    WHERE ft.WeatherCondition IS NULL
      AND ft.DepartureTime >= DATEADD(DAY, -7, GETDATE());
END;
GO
```

### Predictive Maintenance Scoring

```sql
-- Calculate vehicle health score based on telemetry
CREATE VIEW vw_VehicleHealthScore AS
SELECT 
    VehicleID,
    COUNT(CASE WHEN MaintenanceIssueDetected = 1 THEN 1 END) AS IssueCount,
    AVG(FuelEfficiencyMPG) AS AvgFuelEfficiency,
    AVG(IdleTimeMinutes / NULLIF(ActualDurationMinutes, 0)) AS AvgIdleRatio,
    MAX(DepartureTime) AS LastTripDate,
    CASE 
        WHEN COUNT(CASE WHEN MaintenanceIssueDetected = 1 AND MaintenanceIssueSeverity = 'Critical' THEN 1 END) > 0 THEN 0
        WHEN COUNT(CASE WHEN MaintenanceIssueDetected = 1 THEN 1 END) > 5 THEN 25
        WHEN AVG(FuelEfficiencyMPG) < 6.0 THEN 40
        WHEN AVG(IdleTimeMinutes / NULLIF(ActualDurationMinutes, 0)) > 0.20 THEN 60
        ELSE 85
    END AS HealthScore
FROM FactFleetTrips
WHERE DepartureTime >= DATEADD(DAY, -90, GETDATE())
GROUP BY VehicleID;
GO
```

### Cross-Dock Optimization Query

```sql
-- Identify products with optimal cross-dock dwell time
WITH CrossDockPerformance AS (
    SELECT 
        dp.SKU,
        dp.ProductName,
        dp.IsPerishable,
        AVG(fcd.DwellTimeMinutes) AS AvgDwellMinutes,
        STDEV(fcd.DwellTimeMinutes) AS StdDevDwell,
        COUNT(*) AS CrossDockCount,
        AVG(CASE WHEN fcd.InspectionRequired = 1 AND fcd.InspectionPassed = 0 THEN 1 ELSE 0 END) AS FailureRate
    FROM FactCrossDock fcd
    INNER JOIN DimProductGravity dp ON fcd.ProductKey = dp.ProductKey
    WHERE fcd.InboundArrivalTime >= DATEADD(DAY, -60, GETDATE())
    GROUP BY dp.SKU, dp.ProductName, dp.IsPerishable
    HAVING COUNT(*) >= 10
)
SELECT
