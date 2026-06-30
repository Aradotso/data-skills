---
name: logifleet-pulse-supply-chain-analytics
description: MS SQL Server & Power BI logistics intelligence platform with multi-fact star schema for warehouse, fleet, and supply chain analytics
triggers:
  - "set up LogiFleet Pulse supply chain analytics"
  - "configure Power BI logistics dashboard"
  - "deploy multi-fact star schema for warehouse and fleet data"
  - "implement cross-modal supply chain analytics"
  - "create logistics KPI dashboard with Power BI"
  - "build warehouse gravity zone optimization"
  - "set up real-time fleet tracking analytics"
  - "configure SQL Server data warehouse for logistics"
---

# LogiFleet Pulse Supply Chain Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## What This Project Does

LogiFleet Pulse is a comprehensive logistics intelligence platform that unifies warehouse operations, fleet telemetry, inventory management, and external data signals into a single semantic layer. Built on MS SQL Server with Power BI visualizations, it provides:

- **Multi-fact star schema** linking warehouse operations, fleet trips, and cross-dock activities
- **Time-phased dimensions** with 15-minute granularity for real-time analytics
- **Warehouse Gravity Zones™** for spatial optimization based on pick frequency and item value
- **Adaptive Fleet Triage Engine** for proactive maintenance prioritization
- **Cross-fact KPI harmonization** correlating inventory turnover with fleet efficiency
- **Predictive bottleneck detection** using temporal elasticity modeling

## Installation & Setup

### Prerequisites

```bash
# Required software
# - MS SQL Server 2019+ (Standard or Enterprise edition)
# - Power BI Desktop (latest version)
# - SQL Server Management Studio (SSMS)
```

### Database Schema Deployment

1. **Clone the repository**:
```bash
git clone https://github.com/Empty5i/LogiCore-Analytics-Adaptive-Supply-Chain-Pulse.git
cd LogiCore-Analytics-Adaptive-Supply-Chain-Pulse
```

2. **Deploy SQL schema**:
```sql
-- Connect to your SQL Server instance and execute
-- Schema creates: tables, views, stored procedures, indexes

-- Step 1: Create database
CREATE DATABASE LogiFleetPulse;
GO

USE LogiFleetPulse;
GO

-- Step 2: Execute main schema script
-- Run the provided SQL scripts in order:
-- 01_create_dimensions.sql
-- 02_create_facts.sql
-- 03_create_bridge_tables.sql
-- 04_create_views.sql
-- 05_create_stored_procedures.sql
```

3. **Configure data source connections**:
```json
// config.json (create from config_sample.json)
{
  "sqlServer": {
    "server": "${SQL_SERVER_HOST}",
    "database": "LogiFleetPulse",
    "authentication": "ActiveDirectoryIntegrated"
  },
  "dataSources": {
    "wms": {
      "type": "REST_API",
      "endpoint": "${WMS_API_ENDPOINT}",
      "apiKey": "${WMS_API_KEY}"
    },
    "telematics": {
      "type": "STREAMING",
      "endpoint": "${TELEMETRY_STREAM_URL}",
      "token": "${TELEMETRY_TOKEN}"
    },
    "weather": {
      "type": "REST_API",
      "endpoint": "https://api.weatherapi.com/v1",
      "apiKey": "${WEATHER_API_KEY}"
    }
  },
  "refreshInterval": 900
}
```

## Core SQL Schema Components

### Dimension Tables

**DimTime** - Time dimension with 15-minute granularity:
```sql
-- Query time dimension
SELECT 
    TimeKey,
    FullDateTime,
    HourOfDay,
    DayOfWeek,
    FiscalPeriod,
    IsWeekend,
    IsHoliday
FROM DimTime
WHERE FiscalPeriod = '2026-Q2';
```

**DimGeography** - Hierarchical location dimension:
```sql
-- Geography hierarchy query
SELECT 
    GeographyKey,
    Continent,
    Country,
    Region,
    WarehouseCode,
    Latitude,
    Longitude
FROM DimGeography
WHERE Country = 'USA'
ORDER BY Region, WarehouseCode;
```

**DimProductGravity** - Product classification with gravity scoring:
```sql
-- Products by gravity zone
SELECT 
    ProductKey,
    SKU,
    ProductName,
    GravityScore,
    VelocityClass,
    FragilityIndex,
    OptimalZoneID
FROM DimProductGravity
WHERE GravityScore > 80  -- High-gravity (fast-moving) items
ORDER BY GravityScore DESC;
```

**DimSupplierReliability** - Supplier performance metrics:
```sql
-- Supplier reliability ranking
SELECT 
    SupplierKey,
    SupplierName,
    LeadTimeVariance,
    DefectPercentage,
    ComplianceScore,
    ReliabilityRating
FROM DimSupplierReliability
WHERE ComplianceScore >= 90
ORDER BY ReliabilityRating DESC;
```

### Fact Tables

**FactWarehouseOperations** - Warehouse activity tracking:
```sql
-- Warehouse operations with dwell time analysis
SELECT 
    w.OperationKey,
    t.FullDateTime,
    p.SKU,
    g.WarehouseCode,
    w.OperationType,  -- 'RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING'
    w.DwellTimeMinutes,
    w.ProcessingTimeMinutes,
    w.UnitsProcessed
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey
JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
WHERE w.OperationType = 'PICKING'
  AND w.DwellTimeMinutes > 4320  -- Items with >72 hours dwell time
ORDER BY w.DwellTimeMinutes DESC;
```

**FactFleetTrips** - Fleet telemetry and trip data:
```sql
-- Fleet performance with idle time analysis
SELECT 
    f.TripKey,
    t.FullDateTime AS DepartureTime,
    g.WarehouseCode AS Origin,
    f.VehicleID,
    f.RouteSegment,
    f.DistanceKM,
    f.FuelConsumedLiters,
    f.IdleTimeMinutes,
    f.LoadWeightKG,
    f.TripDurationMinutes,
    (f.IdleTimeMinutes * 100.0 / f.TripDurationMinutes) AS IdlePercentage
FROM FactFleetTrips f
JOIN DimTime t ON f.DepartureTimeKey = t.TimeKey
JOIN DimGeography g ON f.OriginKey = g.GeographyKey
WHERE f.IdleTimeMinutes > (f.TripDurationMinutes * 0.15)  -- >15% idle time
ORDER BY IdlePercentage DESC;
```

**FactCrossDock** - Cross-dock transfer operations:
```sql
-- Cross-dock efficiency analysis
SELECT 
    c.CrossDockKey,
    tin.FullDateTime AS InboundTime,
    tout.FullDateTime AS OutboundTime,
    p.SKU,
    c.UnitsTransferred,
    c.TransferDurationMinutes,
    c.TemperatureCompliant,
    c.QualityCheckPassed
FROM FactCrossDock c
JOIN DimTime tin ON c.InboundTimeKey = tin.TimeKey
JOIN DimTime tout ON c.OutboundTimeKey = tout.TimeKey
JOIN DimProductGravity p ON c.ProductKey = p.ProductKey
WHERE c.TransferDurationMinutes < 30  -- Fast cross-dock operations
  AND c.TemperatureCompliant = 1
ORDER BY c.TransferDurationMinutes;
```

## Key Stored Procedures

### Incremental Data Loading

```sql
-- Load warehouse operations data
EXEC sp_LoadWarehouseOperations 
    @StartDateTime = '2026-06-01 00:00:00',
    @EndDateTime = '2026-06-30 23:59:59',
    @WarehouseCode = 'WH-CENTRAL-01';

-- Load fleet trip data
EXEC sp_LoadFleetTrips 
    @StartDateTime = '2026-06-01 00:00:00',
    @EndDateTime = '2026-06-30 23:59:59',
    @RegionFilter = 'MIDWEST';

-- Refresh gravity scores (recalculates based on recent activity)
EXEC sp_RefreshProductGravityScores;
```

### KPI Calculation Procedures

```sql
-- Calculate cross-fact KPIs
EXEC sp_CalculateCrossFactKPIs 
    @ReportingPeriod = '2026-06',
    @OutputTable = 'KPI_Snapshot_June2026';

-- Generate predictive bottleneck index
EXEC sp_GenerateBottleneckIndex 
    @ForecastHorizonHours = 72,
    @ConfidenceThreshold = 0.75;

-- Fleet maintenance priority queue
EXEC sp_GenerateFleetTriageQueue 
    @CriticalityThreshold = 7.5,
    @OutputTable = 'FleetMaintenancePriority';
```

## Power BI Configuration

### Connecting to SQL Server

1. **Open Power BI Desktop**
2. **Get Data** → **SQL Server**
3. Configure connection:
```
Server: ${SQL_SERVER_HOST}
Database: LogiFleetPulse
Data Connectivity mode: DirectQuery (recommended for real-time)
```

4. **Import the template**:
   - Open `LogiFleet_Pulse_Master.pbit`
   - Enter connection parameters when prompted
   - Verify relationships auto-detect correctly

### Key DAX Measures

**Warehouse Gravity Score**:
```dax
// Average gravity score weighted by transaction volume
Weighted_Gravity_Score = 
SUMX(
    FactWarehouseOperations,
    FactWarehouseOperations[UnitsProcessed] * 
    RELATED(DimProductGravity[GravityScore])
) / SUM(FactWarehouseOperations[UnitsProcessed])
```

**Fleet Efficiency Index**:
```dax
// Fleet efficiency: actual vs. optimal time
Fleet_Efficiency_Index = 
VAR TotalDistance = SUM(FactFleetTrips[DistanceKM])
VAR TotalTime = SUM(FactFleetTrips[TripDurationMinutes])
VAR IdleTime = SUM(FactFleetTrips[IdleTimeMinutes])
VAR OptimalTime = TotalDistance / 60 * 60  // Assume 60 km/h optimal
RETURN
DIVIDE(OptimalTime, TotalTime - IdleTime, 0)
```

**Cross-Dock Velocity**:
```dax
// Average cross-dock processing speed
CrossDock_Velocity = 
DIVIDE(
    SUM(FactCrossDock[UnitsTransferred]),
    SUM(FactCrossDock[TransferDurationMinutes]) / 60,
    0
) // Units per hour
```

**Dwell Time Anomaly Detection**:
```dax
// Flag items with excessive dwell time
Dwell_Time_Anomaly = 
VAR AvgDwell = CALCULATE(
    AVERAGE(FactWarehouseOperations[DwellTimeMinutes]),
    ALL(FactWarehouseOperations)
)
VAR StdDev = STDEV.P(FactWarehouseOperations[DwellTimeMinutes])
VAR CurrentDwell = SUM(FactWarehouseOperations[DwellTimeMinutes])
RETURN
IF(CurrentDwell > AvgDwell + (2 * StdDev), "ANOMALY", "NORMAL")
```

### Setting Up Scheduled Refresh

```powershell
# Power BI Service configuration (via REST API)
# Requires Power BI Pro or Premium

$headers = @{
    "Authorization" = "Bearer ${POWERBI_ACCESS_TOKEN}"
    "Content-Type" = "application/json"
}

$refreshSchedule = @{
    value = @(
        @{
            days = @("Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday")
            times = @("00:15", "06:15", "12:15", "18:15")  # Every 6 hours + 15 minutes
        }
    )
} | ConvertTo-Json -Depth 3

Invoke-RestMethod -Method Patch `
    -Uri "https://api.powerbi.com/v1.0/myorg/groups/${WORKSPACE_ID}/datasets/${DATASET_ID}/refreshSchedule" `
    -Headers $headers `
    -Body $refreshSchedule
```

## Common Query Patterns

### Cross-Fact Analysis: Inventory Turnover vs Fleet Efficiency

```sql
-- Correlate slow-moving inventory with fleet utilization
WITH InventoryTurnover AS (
    SELECT 
        p.SKU,
        p.ProductName,
        AVG(w.DwellTimeMinutes) AS AvgDwellTime,
        COUNT(DISTINCT w.OperationKey) AS TotalOperations
    FROM FactWarehouseOperations w
    JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
    WHERE w.OperationType IN ('RECEIVING', 'SHIPPING')
    GROUP BY p.SKU, p.ProductName
),
FleetStats AS (
    SELECT 
        g.Region,
        AVG(f.IdleTimeMinutes * 100.0 / f.TripDurationMinutes) AS AvgIdlePercentage,
        AVG(f.FuelConsumedLiters / f.DistanceKM) AS AvgFuelEfficiency
    FROM FactFleetTrips f
    JOIN DimGeography g ON f.OriginKey = g.GeographyKey
    GROUP BY g.Region
)
SELECT 
    i.SKU,
    i.ProductName,
    i.AvgDwellTime,
    i.TotalOperations,
    fs.AvgIdlePercentage,
    fs.AvgFuelEfficiency
FROM InventoryTurnover i
CROSS JOIN FleetStats fs
WHERE i.AvgDwellTime > 2880  -- >48 hours dwell time
ORDER BY i.AvgDwellTime DESC;
```

### Temporal Pattern Analysis: Peak Hours by Warehouse Zone

```sql
-- Identify warehouse activity patterns by hour and gravity zone
SELECT 
    t.HourOfDay,
    p.OptimalZoneID,
    p.VelocityClass,
    COUNT(*) AS OperationCount,
    AVG(w.ProcessingTimeMinutes) AS AvgProcessingTime,
    SUM(w.UnitsProcessed) AS TotalUnits
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey
JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
WHERE t.IsWeekend = 0
  AND w.OperationType = 'PICKING'
GROUP BY t.HourOfDay, p.OptimalZoneID, p.VelocityClass
ORDER BY t.HourOfDay, OperationCount DESC;
```

### Predictive Bottleneck Query

```sql
-- Forecast potential bottlenecks using rolling averages
WITH RollingMetrics AS (
    SELECT 
        w.WarehouseKey,
        t.FullDateTime,
        AVG(w.ProcessingTimeMinutes) OVER (
            PARTITION BY w.WarehouseKey 
            ORDER BY t.TimeKey 
            ROWS BETWEEN 95 PRECEDING AND CURRENT ROW
        ) AS Rolling24HrAvgTime,
        w.ProcessingTimeMinutes AS CurrentProcessingTime
    FROM FactWarehouseOperations w
    JOIN DimTime t ON w.TimeKey = t.TimeKey
)
SELECT 
    g.WarehouseCode,
    rm.FullDateTime,
    rm.Rolling24HrAvgTime,
    rm.CurrentProcessingTime,
    CASE 
        WHEN rm.CurrentProcessingTime > (rm.Rolling24HrAvgTime * 1.5) 
        THEN 'HIGH_RISK'
        WHEN rm.CurrentProcessingTime > (rm.Rolling24HrAvgTime * 1.2) 
        THEN 'MODERATE_RISK'
        ELSE 'NORMAL'
    END AS BottleneckRisk
FROM RollingMetrics rm
JOIN DimGeography g ON rm.WarehouseKey = g.GeographyKey
WHERE rm.CurrentProcessingTime > (rm.Rolling24HrAvgTime * 1.2)
ORDER BY rm.FullDateTime DESC;
```

## Automated Alerting Configuration

### SQL Server Agent Job for Threshold Alerts

```sql
-- Create stored procedure for automated alerts
CREATE PROCEDURE sp_CheckKPIThresholdsAndAlert
AS
BEGIN
    DECLARE @AlertMessage NVARCHAR(MAX);
    
    -- Check for excessive fleet idle time
    IF EXISTS (
        SELECT 1 
        FROM FactFleetTrips 
        WHERE DepartureTimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(hour, -2, GETDATE()))
          AND (IdleTimeMinutes * 100.0 / TripDurationMinutes) > 15
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Fleet idle time exceeded 15% threshold in last 2 hours';
        -- Send via email (configure Database Mail)
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Fleet Efficiency Alert',
            @body = @AlertMessage;
    END
    
    -- Check for warehouse dwell time anomalies
    IF EXISTS (
        SELECT 1
        FROM FactWarehouseOperations
        WHERE TimeKey >= (SELECT TimeKey FROM DimTime WHERE FullDateTime >= DATEADD(hour, -4, GETDATE()))
          AND DwellTimeMinutes > 4320  -- >72 hours
          AND OperationType = 'PUTAWAY'
    )
    BEGIN
        SET @AlertMessage = 'ALERT: Items with excessive dwell time (>72 hours) detected';
        EXEC msdb.dbo.sp_send_dbmail
            @recipients = '${ALERT_EMAIL_RECIPIENTS}',
            @subject = 'LogiFleet Pulse: Warehouse Dwell Time Alert',
            @body = @AlertMessage;
    END
END;
GO

-- Schedule the alert job (via SQL Server Agent)
-- Run every 15 minutes during business hours
```

### Power BI Data Alert Setup

```json
// Power BI alert configuration (via Power BI REST API)
{
  "alertTitle": "High Fleet Idle Time Alert",
  "dataset": "LogiFleetPulse",
  "measure": "Fleet_Idle_Percentage",
  "operator": "GreaterThan",
  "threshold": 15,
  "frequency": "Hourly",
  "recipients": ["${ALERT_EMAIL_RECIPIENTS}"]
}
```

## Role-Based Security Implementation

### Row-Level Security (RLS) Configuration

```sql
-- Create security roles in Power BI
-- RLS filters data based on user context

-- Create role: Regional Managers (see only their region)
CREATE ROLE RegionalManager;

-- Add filter predicate
ALTER ROLE RegionalManager ADD FILTER PREDICATE
ON DimGeography
WHERE Region = USERNAME();

-- Create role: Fleet Supervisors (see only fleet data, no warehouse details)
CREATE ROLE FleetSupervisor;

-- Hide warehouse operations for fleet supervisors
ALTER ROLE FleetSupervisor ADD FILTER PREDICATE
ON FactWarehouseOperations
WHERE 1 = 0;  -- Block all warehouse data

-- Create role: Executive (full access, aggregated view)
CREATE ROLE Executive;
-- No filters - full data access
```

### Active Directory Integration

```sql
-- Map AD groups to Power BI roles
-- Configure in Power BI Service > Dataset Settings > Security

-- Example mapping table
CREATE TABLE SecurityMapping (
    ADGroup NVARCHAR(255),
    PowerBIRole NVARCHAR(100),
    DataScope NVARCHAR(50)
);

INSERT INTO SecurityMapping VALUES
('DOMAIN\Logistics_Regional_Managers', 'RegionalManager', 'REGION_FILTERED'),
('DOMAIN\Fleet_Operations', 'FleetSupervisor', 'FLEET_ONLY'),
('DOMAIN\Executive_Team', 'Executive', 'FULL_ACCESS');
```

## Performance Optimization

### Indexing Strategy

```sql
-- Create columnstore indexes for fact tables (large volume)
CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactWarehouseOps_CS
ON FactWarehouseOperations (
    TimeKey, ProductKey, WarehouseKey, 
    DwellTimeMinutes, ProcessingTimeMinutes, UnitsProcessed
);

CREATE NONCLUSTERED COLUMNSTORE INDEX IX_FactFleetTrips_CS
ON FactFleetTrips (
    DepartureTimeKey, OriginKey, DestinationKey,
    DistanceKM, FuelConsumedLiters, IdleTimeMinutes
);

-- Create B-tree indexes for frequent join columns
CREATE NONCLUSTERED INDEX IX_FactWarehouse_Time
ON FactWarehouseOperations (TimeKey)
INCLUDE (ProductKey, WarehouseKey, UnitsProcessed);

CREATE NONCLUSTERED INDEX IX_FactFleet_Time
ON FactFleetTrips (DepartureTimeKey)
INCLUDE (OriginKey, VehicleID, RouteSegment);

-- Create filtered index for anomaly detection queries
CREATE NONCLUSTERED INDEX IX_FactWarehouse_HighDwell
ON FactWarehouseOperations (DwellTimeMinutes, TimeKey)
WHERE DwellTimeMinutes > 4320;  -- >72 hours
```

### Partitioning Strategy

```sql
-- Partition fact tables by month for better query performance
-- Create partition function
CREATE PARTITION FUNCTION PF_Monthly (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2026-01-01', '2026-02-01', '2026-03-01', 
    '2026-04-01', '2026-05-01', '2026-06-01',
    '2026-07-01', '2026-08-01', '2026-09-01',
    '2026-10-01', '2026-11-01', '2026-12-01'
);

-- Create partition scheme
CREATE PARTITION SCHEME PS_Monthly
AS PARTITION PF_Monthly
ALL TO ([PRIMARY]);

-- Apply to fact table (requires table rebuild)
CREATE TABLE FactWarehouseOperations_Partitioned (
    OperationKey BIGINT NOT NULL,
    TimeKey INT NOT NULL,
    ProductKey INT NOT NULL,
    WarehouseKey INT NOT NULL,
    OperationType VARCHAR(20),
    DwellTimeMinutes INT,
    ProcessingTimeMinutes INT,
    UnitsProcessed INT,
    LoadDateTime DATETIME NOT NULL
) ON PS_Monthly(LoadDateTime);
```

## Troubleshooting

### Common Issues and Solutions

**Issue: Power BI report refresh fails with timeout**
```sql
-- Solution: Verify DirectQuery mode is enabled
-- Check query execution time
SET STATISTICS TIME ON;
SELECT TOP 1000 * FROM FactWarehouseOperations;
SET STATISTICS TIME OFF;

-- If queries exceed 30 seconds, add aggregation views
CREATE VIEW vw_WarehouseOperations_Hourly
AS
SELECT 
    t.TimeKey,
    p.ProductKey,
    g.WarehouseCode,
    w.OperationType,
    AVG(w.DwellTimeMinutes) AS AvgDwellTime,
    SUM(w.UnitsProcessed) AS TotalUnits
FROM FactWarehouseOperations w
JOIN DimTime t ON w.TimeKey = t.TimeKey
JOIN DimProductGravity p ON w.ProductKey = p.ProductKey
JOIN DimGeography g ON w.WarehouseKey = g.GeographyKey
GROUP BY t.TimeKey, p.ProductKey, g.WarehouseCode, w.OperationType;
```

**Issue: Gravity scores not updating**
```sql
-- Solution: Manually trigger recalculation
EXEC sp_RefreshProductGravityScores;

-- Check last update timestamp
SELECT 
    MAX(LastUpdated) AS LastGravityScoreUpdate
FROM DimProductGravity;

-- Verify trigger is active
SELECT 
    name, 
    is_disabled
FROM sys.triggers
WHERE name = 'trg_UpdateGravityScores';
```

**Issue: Cross-fact KPI calculations showing NULL values**
```sql
-- Solution: Check for missing dimension relationships
SELECT 
    'FactWarehouseOperations' AS TableName,
    COUNT(*) AS TotalRows,
    COUNT(DISTINCT TimeKey) AS UniqueTimeKeys,
    COUNT(DISTINCT ProductKey) AS UniqueProductKeys
FROM FactWarehouseOperations

UNION ALL

SELECT 
    'FactFleetTrips',
    COUNT(*),
    COUNT(DISTINCT DepartureTimeKey),
    NULL
FROM FactFleetTrips;

-- Validate foreign key relationships
SELECT 
    f.name AS ForeignKey,
    OBJECT_NAME(f.parent_object_id) AS TableName,
    COL_NAME(fc.parent_object_id, fc.parent_column_id) AS ColumnName,
    OBJECT_NAME(f.referenced_object_id) AS ReferencedTable
FROM sys.foreign_keys AS f
INNER JOIN sys.foreign_key_columns AS fc 
    ON f.object_id = fc.constraint_object_id;
```

**Issue: Excessive memory usage in Power BI**
```dax
// Solution: Reduce cardinality in cross-joins
// Replace CROSSJOIN with explicit relationships

// Before (high memory):
CALCULATE(
    SUM(FactFleetTrips[FuelConsumedLiters]),
    CROSSJOIN(DimTime, DimGeography)
)

// After (optimized):
CALCULATE(
    SUM(FactFleetTrips[FuelConsumedLiters]),
    USERELATIONSHIP(FactFleetTrips[DepartureTimeKey], DimTime[TimeKey])
)
```

## Data Integration Examples

### Ingesting WMS Data via REST API

```python
# Python script for WMS data ingestion
import requests
import pyodbc
import os
from datetime import datetime

# Configuration from environment variables
WMS_API_ENDPOINT = os.getenv('WMS_API_ENDPOINT')
WMS_API_KEY = os.getenv('WMS_API_KEY')
SQL_SERVER_HOST = os.getenv('SQL_SERVER_HOST')
SQL_DATABASE = 'LogiFleetPulse'

# Fetch warehouse operations from WMS API
def fetch_wms_operations(start_date, end_date):
    headers = {'Authorization': f'Bearer {WMS_API_KEY}'}
    params = {
        'start_date': start_date.isoformat(),
        'end_date': end_date.isoformat(),
        'operation_types': ['RECEIVING', 'PUTAWAY', 'PICKING', 'PACKING', 'SHIPPING']
    }
    
    response = requests.get(
        f'{WMS_API_ENDPOINT}/operations',
        headers=headers,
        params=params
    )
    response.raise_for_status()
    return response.json()

# Load data into SQL Server
def load_to_sql(operations):
    conn = pyodbc.connect(
        f'DRIVER={{ODBC Driver 17 for SQL Server}};'
        f'SERVER={SQL_SERVER_HOST};'
        f'DATABASE={SQL_DATABASE};'
        f'Trusted_Connection=yes;'
    )
    cursor = conn.cursor()
    
    insert_query = """
    INSERT INTO FactWarehouseOperations 
    (TimeKey, ProductKey, WarehouseKey, OperationType, 
     DwellTimeMinutes, ProcessingTimeMinutes, UnitsProcessed)
    VALUES (?, ?, ?, ?, ?, ?, ?)
    """
    
    for op in operations:
        cursor.execute(insert_query, (
            op['time_key'],
            op['product_key'],
            op['warehouse_key'],
            op['operation_type'],
            op['dwell_time_minutes'],
            op['processing_time_minutes'],
            op['units_processed']
        ))
    
    conn.commit()
    cursor.close()
    conn.close()

# Main execution
if __name__ == '__main__':
    start = datetime(2026, 6, 1)
    end = datetime(2026, 6, 30)
    
    operations = fetch_wms_operations(start, end)
    load_to_sql(operations)
    print(f'Loaded {len(operations)} warehouse operations')
```

### Streaming Telemetry Data Ingestion

```csharp
// C# service for real-time fleet telemetry ingestion
using System;
using System.Data.SqlClient;
using Microsoft.Azure.EventHubs;
using Newtonsoft.Json;

public class TelemetryIngestionService
{
    private readonly string _eventHubConnectionString = Environment.GetEnvironmentVariable("TELEMETRY_STREAM_URL");
    private readonly string _sqlConnectionString = $"Server={Environment.GetEnvironmentVariable("SQL_SERVER_HOST")};Database=LogiFleetPulse;Integrated Security=true;";
    
    public async Task StartIngestion()
    {
        var eventHubClient = EventHubClient.CreateFromConnectionString(_eventHubConnectionString);
        var runtimeInfo = await eventHubClient.GetRuntimeInformationAsync();
        
        foreach (var partitionId in runtimeInfo.PartitionIds)
        {
            var receiver = eventHubClient.CreateReceiver(
                "$Default", 
                partitionId, 
                EventPosition.FromEnd()
            );
            
            receiver.SetReceiveHandler(new TelemetryEventProcessor(_sqlConnectionString));
        }
    }
}

public class TelemetryEventProcessor : IPartitionReceiveHandler
{
    private readonly string _connectionString;
    
    public TelemetryEventProcessor(string connectionString)
    {
        _connectionString = connectionString;
    }
    
    public Task ProcessEventsAsync(IEnumerable<EventData> events)
    {
        using (var connection = new SqlConnection(_connectionString))
        {
            connection.Open();
            
            foreach (var eventData in events)
            {
                var telemetry = JsonConvert.DeserializeObject<FleetTelemetry>(
                    Encoding.UTF8.GetString(eventData.Body.Array)
                );
                
                InsertFleetTrip(connection, telemetry);
            }
        }
        
        return Task.CompletedTask;
    }
    
    private void InsertFleetTrip(SqlConnection connection, FleetTelemetry telemetry)
    {
        var command = new SqlCommand(@"
            INSERT INTO FactFleetTrips 
            (DepartureTimeKey, OriginKey, Destination
