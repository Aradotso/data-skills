---
name: commercevault-edd-digital-commerce
description: EDD MCP server for Easy Digital Downloads sales analytics, product management, and order orchestration
triggers:
  - "connect to Easy Digital Downloads API"
  - "fetch EDD sales data and analytics"
  - "query digital product sales statistics"
  - "retrieve EDD customer orders and revenue"
  - "analyze Easy Digital Downloads metrics"
  - "integrate with WordPress EDD store"
  - "pull digital commerce transaction data"
  - "monitor EDD subscription and license keys"
---

# CommerceVault EDD Digital Commerce Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CommerceVault (mcp-edd-analytics-vantage) is a Model Context Protocol (MCP) server that provides a unified API layer for Easy Digital Downloads (EDD) WordPress installations. It enables programmatic access to sales data, product catalogs, customer analytics, order lifecycle management, and license/entitlement tracking through a secure, standardized REST interface.

## What It Does

- **Sales Analytics**: Query revenue metrics, GMV, AOV, cohort analysis, and time-series sales data
- **Product Catalog Management**: Sync product listings, pricing tiers, digital downloads, and variable products
- **Order Processing**: Create, retrieve, update, and refund orders with full lifecycle tracking
- **Customer Segmentation**: Access customer profiles, LTV calculations, purchase history, and behavioral cohorts
- **License Management**: Generate and validate license keys for digital products and SaaS offerings
- **Real-time Webhooks**: Receive event-driven notifications for order completions, refunds, and customer updates

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- Easy Digital Downloads WordPress installation with REST API enabled
- Valid EDD API credentials (consumer key and consumer secret)

### MCP Server Setup

Install as an MCP server for Claude Desktop, Cursor, or other MCP-compatible tools:

```bash
# Clone the repository
git clone https://github.com/dhapat3927/mcp-edd-analytics-vantage.git
cd mcp-edd-analytics-vantage

# Install dependencies
npm install
# or
pip install -r requirements.txt

# Configure environment variables
cp .env.example .env
```

### Environment Configuration

Edit `.env` with your EDD credentials:

```bash
EDD_SITE_URL=https://your-wordpress-site.com
EDD_CONSUMER_KEY=ck_your_consumer_key_here
EDD_CONSUMER_SECRET=cs_your_consumer_secret_here
EDD_API_VERSION=v2
CACHE_ENABLED=true
CACHE_TTL_SECONDS=300
LOG_LEVEL=info
WEBHOOK_SECRET=your_webhook_signing_secret
```

### MCP Configuration

Add to your MCP settings file (e.g., `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "commercevault-edd": {
      "command": "node",
      "args": ["/path/to/mcp-edd-analytics-vantage/index.js"],
      "env": {
        "EDD_SITE_URL": "https://your-store.com",
        "EDD_CONSUMER_KEY": "${EDD_CONSUMER_KEY}",
        "EDD_CONSUMER_SECRET": "${EDD_CONSUMER_SECRET}"
      }
    }
  }
}
```

## Key API Methods

### Fetching Sales Analytics

```javascript
// Get sales summary for last 30 days
const salesData = await mcp.call('edd.getSalesSummary', {
  start_date: '2026-06-01',
  end_date: '2026-06-30',
  granularity: 'daily'
});

console.log(salesData);
// {
//   total_revenue: 45230.50,
//   total_orders: 342,
//   average_order_value: 132.25,
//   refund_rate: 0.02,
//   daily_breakdown: [...]
// }
```

### Querying Products

```javascript
// Retrieve product catalog with filters
const products = await mcp.call('edd.getProducts', {
  status: 'publish',
  category: 'software',
  page: 1,
  per_page: 50
});

products.forEach(product => {
  console.log(`${product.name} - $${product.price}`);
});
```

### Creating Orders

```javascript
// Create new order programmatically
const newOrder = await mcp.call('edd.createOrder', {
  customer_email: 'customer@example.com',
  products: [
    { id: 123, quantity: 1, price: 49.99 },
    { id: 456, quantity: 2, price: 29.99 }
  ],
  payment_method: 'stripe',
  status: 'completed'
});

console.log(`Order created: ${newOrder.id}`);
```

### Customer Lifetime Value Analysis

```javascript
// Get customer analytics with LTV calculation
const customerData = await mcp.call('edd.getCustomerAnalytics', {
  customer_id: 789,
  include_ltv: true,
  include_purchase_history: true
});

console.log(`Customer LTV: $${customerData.lifetime_value}`);
console.log(`Total Purchases: ${customerData.total_orders}`);
console.log(`Average Purchase Frequency: ${customerData.purchase_frequency} days`);
```

### License Key Management

```javascript
// Generate license key for digital product
const license = await mcp.call('edd.generateLicense', {
  product_id: 101,
  customer_email: 'customer@example.com',
  activation_limit: 3,
  expiration_date: '2027-06-30'
});

console.log(`License Key: ${license.key}`);
console.log(`Activations Remaining: ${license.activations_left}`);

// Validate license key
const validation = await mcp.call('edd.validateLicense', {
  license_key: license.key,
  product_id: 101,
  url: 'https://customer-site.com'
});

console.log(`Valid: ${validation.valid}`);
console.log(`Status: ${validation.status}`);
```

## Python SDK Usage

```python
from commercevault import EDDClient
import os

# Initialize client
client = EDDClient(
    site_url=os.getenv('EDD_SITE_URL'),
    consumer_key=os.getenv('EDD_CONSUMER_KEY'),
    consumer_secret=os.getenv('EDD_CONSUMER_SECRET')
)

# Fetch revenue report
revenue = client.analytics.get_revenue(
    start_date='2026-06-01',
    end_date='2026-06-30',
    group_by='product'
)

for product_revenue in revenue:
    print(f"{product_revenue['product_name']}: ${product_revenue['total']}")

# Export customer segment to CSV
segment = client.customers.get_segment(
    filter='ltv_gt_1000',
    recency_days=90
)

client.export.to_csv(segment, 'high_value_customers.csv')
```

## Common Patterns

### Delta Sync for Product Catalog

```javascript
// Only fetch products modified since last sync
const lastSync = await cache.get('last_product_sync');

const updatedProducts = await mcp.call('edd.getProducts', {
  modified_after: lastSync || '2026-01-01',
  per_page: 100
});

console.log(`${updatedProducts.length} products updated since last sync`);

// Update sync timestamp
await cache.set('last_product_sync', new Date().toISOString());
```

### Webhook Event Processing

```javascript
// Handle incoming EDD webhooks
app.post('/webhooks/edd', async (req, res) => {
  const signature = req.headers['x-edd-signature'];
  const payload = req.body;
  
  // Verify webhook authenticity
  const isValid = await mcp.call('edd.verifyWebhook', {
    signature,
    payload,
    secret: process.env.WEBHOOK_SECRET
  });
  
  if (!isValid) {
    return res.status(401).send('Invalid signature');
  }
  
  // Process event
  switch (payload.event) {
    case 'order.completed':
      await processNewOrder(payload.data);
      break;
    case 'refund.issued':
      await handleRefund(payload.data);
      break;
    case 'subscription.renewed':
      await updateSubscription(payload.data);
      break;
  }
  
  res.status(200).send('OK');
});
```

### Cohort Analysis with Aggregation

```javascript
// Analyze customer cohorts by acquisition month
const cohorts = await mcp.call('edd.getCohortAnalysis', {
  start_date: '2025-01-01',
  end_date: '2026-06-30',
  cohort_by: 'acquisition_month',
  metric: 'revenue'
});

cohorts.forEach(cohort => {
  console.log(`${cohort.month}: ${cohort.customer_count} customers, $${cohort.total_revenue}`);
  console.log(`  Retention Rate: ${cohort.retention_rate}%`);
  console.log(`  Repeat Purchase Rate: ${cohort.repeat_rate}%`);
});
```

### Batch Order Updates

```javascript
// Update multiple orders efficiently
const orderUpdates = [
  { id: 1001, status: 'completed', tracking_number: 'TRACK001' },
  { id: 1002, status: 'completed', tracking_number: 'TRACK002' },
  { id: 1003, status: 'refunded', refund_reason: 'Customer request' }
];

const results = await mcp.call('edd.batchUpdateOrders', {
  updates: orderUpdates
});

console.log(`Updated ${results.success_count} orders`);
if (results.errors.length > 0) {
  console.error('Errors:', results.errors);
}
```

## Advanced Configuration

### Rate Limiting Configuration

```javascript
// config/rate-limits.json
{
  "adaptive_throttling": true,
  "max_requests_per_minute": 60,
  "burst_allowance": 10,
  "retry_strategy": {
    "max_retries": 3,
    "backoff_multiplier": 2,
    "initial_delay_ms": 1000
  },
  "queue_overflow_strategy": "drop_oldest"
}
```

### Cache Layer Setup (Redis)

```bash
# .env additions for caching
CACHE_ADAPTER=redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=${REDIS_PASSWORD}
REDIS_DB=0
CACHE_KEY_PREFIX=commercevault:
```

```javascript
// Use cached responses for frequently accessed data
const products = await mcp.call('edd.getProducts', {
  use_cache: true,
  cache_ttl: 600 // 10 minutes
});
```

### Middleware Pipeline Configuration

```javascript
// config/middleware.json
{
  "pipeline": [
    {
      "name": "geo-fencing",
      "enabled": true,
      "config": {
        "allowed_countries": ["US", "CA", "GB"],
        "blocked_products": [101, 102]
      }
    },
    {
      "name": "fraud-scoring",
      "enabled": true,
      "config": {
        "threshold": 0.75,
        "hold_suspicious_orders": true
      }
    },
    {
      "name": "currency-normalizer",
      "enabled": true,
      "config": {
        "base_currency": "USD"
      }
    }
  ]
}
```

## Troubleshooting

### Authentication Errors

```bash
# Test API credentials
curl -u "${EDD_CONSUMER_KEY}:${EDD_CONSUMER_SECRET}" \
  "${EDD_SITE_URL}/wp-json/edd/v2/products?per_page=1"

# Common issue: URL encoding of consumer secret
# Ensure special characters are properly escaped
```

### Rate Limit 429 Errors

```javascript
// Enable adaptive rate limiting
const config = {
  adaptive_throttling: true,
  observe_retry_after_header: true
};

// Check current rate limit status
const status = await mcp.call('edd.getRateLimitStatus');
console.log(`Remaining: ${status.remaining}/${status.limit}`);
console.log(`Reset at: ${status.reset_at}`);
```

### Webhook Signature Verification Failures

```javascript
// Debug webhook signature issues
const debugWebhook = async (req) => {
  const receivedSig = req.headers['x-edd-signature'];
  const payload = JSON.stringify(req.body);
  const secret = process.env.WEBHOOK_SECRET;
  
  const expectedSig = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  console.log('Received:', receivedSig);
  console.log('Expected:', expectedSig);
  console.log('Match:', receivedSig === expectedSig);
};
```

### Data Sync Lag

```javascript
// Force cache invalidation
await mcp.call('edd.invalidateCache', {
  scope: 'products', // or 'orders', 'customers', 'all'
  cascade: true
});

// Verify data freshness
const metadata = await mcp.call('edd.getDataFreshness');
console.log(`Last product sync: ${metadata.products.last_sync}`);
console.log(`Staleness: ${metadata.products.staleness_seconds}s`);
```

### Memory Leaks in Long-Running Processes

```javascript
// Configure connection pooling
const client = new EDDClient({
  pool: {
    max: 10,
    min: 2,
    idle_timeout_ms: 30000,
    acquire_timeout_ms: 5000
  }
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  await client.close();
  process.exit(0);
});
```

## Performance Optimization

### Batch Operations for Large Datasets

```javascript
// Process large order sets in batches
async function* batchOrders(startDate, endDate, batchSize = 100) {
  let page = 1;
  while (true) {
    const orders = await mcp.call('edd.getOrders', {
      start_date: startDate,
      end_date: endDate,
      page,
      per_page: batchSize
    });
    
    if (orders.length === 0) break;
    
    yield orders;
    page++;
  }
}

// Usage
for await (const orderBatch of batchOrders('2026-01-01', '2026-06-30')) {
  await processOrders(orderBatch);
}
```

### Materialized Views for Analytics

```javascript
// Pre-compute expensive analytics queries
await mcp.call('edd.createMaterializedView', {
  name: 'monthly_revenue_by_product',
  query: {
    aggregate: 'revenue',
    group_by: ['month', 'product_id'],
    date_range: 'last_12_months'
  },
  refresh_schedule: '0 0 * * *' // Daily at midnight
});

// Query the materialized view (much faster)
const revenue = await mcp.call('edd.queryMaterializedView', {
  view: 'monthly_revenue_by_product',
  filters: { product_id: 101 }
});
```
