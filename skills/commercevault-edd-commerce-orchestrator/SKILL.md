---
name: commercevault-edd-commerce-orchestrator
description: MCP server for Easy Digital Downloads API integration enabling sales analytics, product management, and order orchestration
triggers:
  - "connect to Easy Digital Downloads API"
  - "query EDD sales data and analytics"
  - "manage digital products and orders in EDD"
  - "set up CommerceVault MCP server"
  - "integrate WordPress EDD with MCP"
  - "retrieve customer data from Easy Digital Downloads"
  - "analyze revenue metrics from EDD store"
  - "configure EDD commerce orchestration"
---

# CommerceVault EDD Commerce Orchestrator

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CommerceVault (mcp-edd-analytics-vantage) is an MCP (Model Context Protocol) server that provides a unified interface for interacting with Easy Digital Downloads (EDD) WordPress e-commerce installations. It transforms raw EDD REST API responses into structured, queryable data for sales analytics, product catalog management, customer segmentation, and order lifecycle tracking.

The server implements hexagonal architecture with pluggable adapters, cryptographic payload verification, adaptive rate limiting, and built-in caching for high-throughput commerce operations.

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- Access to an Easy Digital Downloads WordPress installation
- EDD REST API credentials (Consumer Key and Consumer Secret)

### Install via npm

```bash
npm install -g mcp-edd-analytics-vantage
```

### Install via Python

```bash
pip install mcp-edd-analytics-vantage
```

### Clone from Source

```bash
git clone https://github.com/dhapat3927/mcp-edd-analytics-vantage.git
cd mcp-edd-analytics-vantage
npm install
# or
pip install -r requirements.txt
```

## Configuration

### Environment Variables

Create a `.env` file in your project root:

```bash
# Required - EDD API Credentials
EDD_SITE_URL=https://your-store.com
EDD_CONSUMER_KEY=ck_your_consumer_key_here
EDD_CONSUMER_SECRET=cs_your_consumer_secret_here

# Optional - Server Configuration
COMMERCEVAULT_PORT=3000
COMMERCEVAULT_CACHE_ENABLED=true
COMMERCEVAULT_CACHE_TTL=300
COMMERCEVAULT_RATE_LIMIT=100

# Optional - Security
COMMERCEVAULT_HMAC_SECRET=your_secret_key_for_payload_signing
COMMERCEVAULT_TLS_CERT_PATH=/path/to/cert.pem
COMMERCEVAULT_TLS_KEY_PATH=/path/to/key.pem

# Optional - Analytics
COMMERCEVAULT_METRICS_ENABLED=true
COMMERCEVAULT_RETENTION_DAYS=90
```

### MCP Settings Configuration

Add to your MCP client configuration (e.g., Claude Desktop config):

```json
{
  "mcpServers": {
    "commercevault": {
      "command": "npx",
      "args": ["mcp-edd-analytics-vantage"],
      "env": {
        "EDD_SITE_URL": "https://your-store.com",
        "EDD_CONSUMER_KEY": "${EDD_CONSUMER_KEY}",
        "EDD_CONSUMER_SECRET": "${EDD_CONSUMER_SECRET}"
      }
    }
  }
}
```

## Core API Operations

### Product Management

#### List All Products

```javascript
// Node.js / JavaScript
import { CommerceVault } from 'mcp-edd-analytics-vantage';

const vault = new CommerceVault({
  siteUrl: process.env.EDD_SITE_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET
});

// Retrieve products with filtering
const products = await vault.products.list({
  status: 'publish',
  per_page: 50,
  orderby: 'date',
  order: 'desc'
});

console.log(`Found ${products.length} products`);
products.forEach(product => {
  console.log(`${product.id}: ${product.name} - $${product.price}`);
});
```

```python
# Python
from commercevault import CommerceVault
import os

vault = CommerceVault(
    site_url=os.getenv('EDD_SITE_URL'),
    consumer_key=os.getenv('EDD_CONSUMER_KEY'),
    consumer_secret=os.getenv('EDD_CONSUMER_SECRET')
)

# Retrieve products with filtering
products = vault.products.list(
    status='publish',
    per_page=50,
    orderby='date',
    order='desc'
)

print(f"Found {len(products)} products")
for product in products:
    print(f"{product.id}: {product.name} - ${product.price}")
```

#### Get Single Product Details

```javascript
const productId = 123;
const product = await vault.products.get(productId);

console.log({
  id: product.id,
  name: product.name,
  price: product.price,
  downloads: product.sales_count,
  earnings: product.earnings,
  files: product.files.map(f => f.name)
});
```

#### Create or Update Product

```javascript
const newProduct = await vault.products.create({
  name: 'Premium WordPress Plugin',
  status: 'publish',
  type: 'download',
  price: 49.99,
  categories: ['wordpress', 'plugins'],
  files: [
    {
      name: 'Plugin ZIP',
      file: 'https://cdn.example.com/plugin.zip'
    }
  ]
});

console.log(`Created product ID: ${newProduct.id}`);
```

### Order Management

#### Retrieve Orders with Filters

```javascript
// Get orders from the last 30 days
const thirtyDaysAgo = new Date();
thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

const recentOrders = await vault.orders.list({
  status: 'complete',
  date_created_after: thirtyDaysAgo.toISOString(),
  per_page: 100
});

// Calculate revenue
const totalRevenue = recentOrders.reduce((sum, order) => {
  return sum + parseFloat(order.total);
}, 0);

console.log(`Total revenue (30 days): $${totalRevenue.toFixed(2)}`);
```

#### Get Order Details

```javascript
const orderId = 456;
const order = await vault.orders.get(orderId);

console.log({
  order_id: order.id,
  customer_email: order.billing.email,
  total: order.total,
  status: order.status,
  items: order.line_items.map(item => ({
    product: item.name,
    quantity: item.quantity,
    price: item.price
  }))
});
```

#### Update Order Status

```javascript
await vault.orders.update(orderId, {
  status: 'refunded',
  customer_note: 'Refund processed per customer request'
});
```

### Customer Management

#### List Customers with Segmentation

```javascript
const customers = await vault.customers.list({
  orderby: 'registered_date',
  order: 'desc',
  per_page: 50
});

// Segment by purchase count
const segments = {
  new: customers.filter(c => c.orders_count === 1),
  repeat: customers.filter(c => c.orders_count > 1 && c.orders_count <= 5),
  vip: customers.filter(c => c.orders_count > 5)
};

console.log('Customer Segments:');
console.log(`New: ${segments.new.length}`);
console.log(`Repeat: ${segments.repeat.length}`);
console.log(`VIP: ${segments.vip.length}`);
```

#### Calculate Customer Lifetime Value

```python
customers = vault.customers.list(per_page=100)

for customer in customers:
    ltv = sum(float(order.total) for order in customer.orders)
    avg_order_value = ltv / len(customer.orders) if customer.orders else 0
    
    print(f"{customer.email}:")
    print(f"  LTV: ${ltv:.2f}")
    print(f"  AOV: ${avg_order_value:.2f}")
    print(f"  Orders: {len(customer.orders)}")
```

### Analytics & Reporting

#### Revenue Analytics

```javascript
const analytics = await vault.analytics.revenue({
  start_date: '2026-01-01',
  end_date: '2026-06-30',
  interval: 'month'
});

analytics.periods.forEach(period => {
  console.log(`${period.month}: $${period.gross_revenue} (${period.order_count} orders)`);
});

console.log('\nSummary:');
console.log(`Total Revenue: $${analytics.total_revenue}`);
console.log(`Average Order Value: $${analytics.average_order_value}`);
console.log(`Total Orders: ${analytics.total_orders}`);
```

#### Product Performance Report

```javascript
const productAnalytics = await vault.analytics.products({
  start_date: '2026-01-01',
  end_date: '2026-06-30',
  orderby: 'revenue',
  order: 'desc',
  limit: 10
});

console.log('Top 10 Products by Revenue:');
productAnalytics.forEach((product, index) => {
  console.log(`${index + 1}. ${product.name}`);
  console.log(`   Revenue: $${product.revenue}`);
  console.log(`   Sales: ${product.sales_count}`);
  console.log(`   Avg Price: $${product.average_price}`);
});
```

#### Customer Cohort Analysis

```python
cohort_data = vault.analytics.cohorts({
    'cohort_type': 'month',
    'start_date': '2026-01-01',
    'metrics': ['retention', 'revenue']
})

for cohort in cohort_data:
    print(f"Cohort: {cohort.period}")
    print(f"  Initial Customers: {cohort.initial_size}")
    print(f"  Month 1 Retention: {cohort.retention_month_1}%")
    print(f"  Month 3 Retention: {cohort.retention_month_3}%")
    print(f"  Total Revenue: ${cohort.total_revenue:.2f}")
```

### License Management

#### Generate License Keys

```javascript
const license = await vault.licenses.create({
  product_id: 123,
  customer_id: 456,
  license_length: 'lifetime',
  activation_limit: 3
});

console.log(`License Key: ${license.key}`);
console.log(`Activations Remaining: ${license.activation_limit}`);
```

#### Validate and Activate License

```javascript
const validation = await vault.licenses.validate({
  license_key: 'XXXX-XXXX-XXXX-XXXX',
  product_id: 123,
  url: 'https://customer-site.com'
});

if (validation.valid) {
  const activation = await vault.licenses.activate({
    license_key: validation.key,
    url: 'https://customer-site.com'
  });
  
  console.log(`License activated. ${activation.activations_left} activations remaining.`);
} else {
  console.log(`Invalid license: ${validation.error}`);
}
```

## Common Patterns

### Delta Synchronization

Efficiently sync only changed records:

```javascript
// Store last sync timestamp
let lastSync = await vault.cache.get('last_product_sync');
if (!lastSync) {
  lastSync = new Date('2026-01-01').toISOString();
}

// Fetch only products modified since last sync
const updatedProducts = await vault.products.list({
  modified_after: lastSync,
  per_page: 100
});

console.log(`${updatedProducts.length} products updated since ${lastSync}`);

// Update local database/cache
for (const product of updatedProducts) {
  await localDb.products.upsert(product);
}

// Update sync timestamp
await vault.cache.set('last_product_sync', new Date().toISOString());
```

### Webhook Relay Setup

Configure webhook endpoints to receive real-time events:

```javascript
// Register webhook handler
vault.webhooks.register({
  topic: 'order.completed',
  delivery_url: 'https://your-app.com/webhooks/edd/order-completed',
  secret: process.env.WEBHOOK_SECRET
});

// Webhook handler endpoint
app.post('/webhooks/edd/order-completed', async (req, res) => {
  // Verify webhook signature
  const isValid = vault.webhooks.verify(
    req.body,
    req.headers['x-edd-signature'],
    process.env.WEBHOOK_SECRET
  );
  
  if (!isValid) {
    return res.status(401).send('Invalid signature');
  }
  
  const order = req.body.data;
  
  // Process order (send confirmation email, update inventory, etc.)
  await processNewOrder(order);
  
  res.status(200).send('OK');
});
```

### Batch Operations with Rate Limiting

```javascript
const productIds = [101, 102, 103, 104, 105, /* ... 500 more ... */];

// Process in batches to respect rate limits
const batchSize = 10;
const delay = ms => new Promise(resolve => setTimeout(resolve, ms));

for (let i = 0; i < productIds.length; i += batchSize) {
  const batch = productIds.slice(i, i + batchSize);
  
  const products = await Promise.all(
    batch.map(id => vault.products.get(id))
  );
  
  // Process batch
  await processBatch(products);
  
  // Respect rate limits
  if (i + batchSize < productIds.length) {
    await delay(1000); // 1 second between batches
  }
}
```

### Error Handling and Retry Logic

```python
from commercevault.exceptions import RateLimitError, APIError
import time

def fetch_with_retry(func, max_retries=3, backoff=2):
    for attempt in range(max_retries):
        try:
            return func()
        except RateLimitError as e:
            wait_time = backoff ** attempt
            print(f"Rate limited. Waiting {wait_time}s before retry {attempt + 1}/{max_retries}")
            time.sleep(wait_time)
        except APIError as e:
            if e.status_code >= 500:  # Server error, retry
                print(f"Server error. Retrying {attempt + 1}/{max_retries}")
                time.sleep(backoff ** attempt)
            else:  # Client error, don't retry
                raise
    
    raise Exception(f"Failed after {max_retries} retries")

# Usage
orders = fetch_with_retry(lambda: vault.orders.list(per_page=100))
```

## CLI Commands

If installed globally, CommerceVault provides CLI tools:

```bash
# Start MCP server
commercevault serve --port 3000

# Test connection
commercevault test-connection

# Export analytics report
commercevault export-analytics \
  --start-date 2026-01-01 \
  --end-date 2026-06-30 \
  --format csv \
  --output revenue-report.csv

# Sync products to local database
commercevault sync products \
  --incremental \
  --db-connection postgres://localhost/commerce

# Generate license keys in bulk
commercevault licenses generate \
  --product-id 123 \
  --count 100 \
  --output licenses.json
```

## Troubleshooting

### Authentication Errors

**Problem**: `401 Unauthorized` errors when calling API

**Solution**: Verify credentials and URL format:

```javascript
// Ensure URL does NOT have trailing slash
const vault = new CommerceVault({
  siteUrl: 'https://store.com',  // ✓ Correct
  // siteUrl: 'https://store.com/',  // ✗ Wrong
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET
});

// Test authentication
try {
  await vault.auth.verify();
  console.log('Authentication successful');
} catch (error) {
  console.error('Auth failed:', error.message);
}
```

### Rate Limit Exceeded

**Problem**: `429 Too Many Requests` errors

**Solution**: Enable adaptive rate limiting:

```javascript
const vault = new CommerceVault({
  siteUrl: process.env.EDD_SITE_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  rateLimiting: {
    enabled: true,
    maxRequestsPerSecond: 2,
    retryOnLimit: true,
    maxRetries: 3
  }
});
```

### Cache Invalidation Issues

**Problem**: Stale data returned from API

**Solution**: Force cache refresh or disable caching:

```javascript
// Force fresh data
const products = await vault.products.list({
  per_page: 50,
  _bypass_cache: true
});

// Or clear cache manually
await vault.cache.clear('products:*');
```

### SSL Certificate Errors

**Problem**: `UNABLE_TO_VERIFY_LEAF_SIGNATURE` on self-signed certs

**Solution**: Configure TLS options (development only):

```javascript
const vault = new CommerceVault({
  siteUrl: process.env.EDD_SITE_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  tls: {
    rejectUnauthorized: false  // Only for development!
  }
});
```

### Data Formatting Issues

**Problem**: Dates or currencies displaying incorrectly

**Solution**: Configure locale settings:

```javascript
const vault = new CommerceVault({
  siteUrl: process.env.EDD_SITE_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  locale: {
    language: 'en-US',
    currency: 'USD',
    dateFormat: 'YYYY-MM-DD',
    timezone: 'America/New_York'
  }
});
```

## Best Practices

1. **Always use environment variables** for credentials — never hardcode API keys
2. **Enable caching** for read-heavy operations (product catalogs, customer lists)
3. **Implement webhook handlers** for real-time order processing instead of polling
4. **Use delta sync** when synchronizing large datasets to minimize API quota usage
5. **Log all transactions** with cryptographic fingerprints for audit compliance
6. **Set up monitoring** for API latency and error rates using the built-in metrics endpoint
