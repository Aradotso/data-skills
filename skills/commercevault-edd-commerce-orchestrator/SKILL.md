---
name: commercevault-edd-commerce-orchestrator
description: MCP server for Easy Digital Downloads API integration, sales analytics, and digital product management
triggers:
  - "connect to Easy Digital Downloads API"
  - "fetch EDD sales data and analytics"
  - "integrate with WordPress EDD store"
  - "query digital product orders and customers"
  - "set up CommerceVault MCP server"
  - "analyze EDD revenue and transactions"
  - "manage digital commerce licenses and entitlements"
  - "synchronize EDD product catalog"
---

# CommerceVault EDD Commerce Orchestrator

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CommerceVault is an MCP (Model Context Protocol) server that provides a unified interface to Easy Digital Downloads (EDD) WordPress e-commerce data. It enables AI assistants and applications to query sales analytics, manage products, track customer orders, and handle digital product licensing through a standardized API layer. The system acts as middleware between your application and EDD's REST API, providing enhanced security, caching, observability, and data transformation.

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- Easy Digital Downloads WordPress site with REST API enabled
- EDD API keys (Consumer Key and Secret)
- MCP-compatible client (Claude Desktop, Cline, etc.)

### Setup

1. **Clone and install dependencies:**

```bash
git clone https://github.com/dhapat3927/mcp-edd-analytics-vantage.git
cd mcp-edd-analytics-vantage
npm install
```

2. **Configure environment variables:**

Create `.env` file in project root:

```bash
EDD_API_URL=https://yourstore.com/wp-json/edd/v2
EDD_CONSUMER_KEY=ck_your_consumer_key_here
EDD_CONSUMER_SECRET=cs_your_consumer_secret_here
CACHE_ENABLED=true
CACHE_TTL=3600
LOG_LEVEL=info
```

3. **Add to MCP client configuration:**

For Claude Desktop, edit `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "commercevault-edd": {
      "command": "node",
      "args": ["/path/to/mcp-edd-analytics-vantage/src/index.js"],
      "env": {
        "EDD_API_URL": "https://yourstore.com/wp-json/edd/v2",
        "EDD_CONSUMER_KEY": "ck_your_key",
        "EDD_CONSUMER_SECRET": "cs_your_secret"
      }
    }
  }
}
```

4. **Start the server:**

```bash
npm start
```

## Core API & Commands

### MCP Tools Available

CommerceVault exposes the following MCP tools for AI assistants:

#### 1. Product Management

**get_products** - Retrieve product catalog

```javascript
// MCP tool call
{
  "tool": "get_products",
  "arguments": {
    "page": 1,
    "per_page": 20,
    "status": "publish",
    "category": "software",
    "search": "wordpress"
  }
}
```

**get_product_details** - Get specific product information

```javascript
{
  "tool": "get_product_details",
  "arguments": {
    "product_id": 123
  }
}
```

#### 2. Order & Sales Analytics

**get_orders** - Fetch order history

```javascript
{
  "tool": "get_orders",
  "arguments": {
    "status": "completed",
    "date_from": "2026-01-01",
    "date_to": "2026-06-30",
    "customer_email": "customer@example.com"
  }
}
```

**get_sales_report** - Generate revenue analytics

```javascript
{
  "tool": "get_sales_report",
  "arguments": {
    "period": "month",
    "start_date": "2026-01-01",
    "end_date": "2026-06-30",
    "group_by": "product"
  }
}
```

#### 3. Customer Management

**get_customers** - Query customer database

```javascript
{
  "tool": "get_customers",
  "arguments": {
    "search": "john@example.com",
    "order_by": "purchase_value",
    "per_page": 50
  }
}
```

**get_customer_lifetime_value** - Calculate LTV metrics

```javascript
{
  "tool": "get_customer_lifetime_value",
  "arguments": {
    "customer_id": 456,
    "include_refunds": false
  }
}
```

#### 4. License Management

**get_licenses** - Retrieve license keys

```javascript
{
  "tool": "get_licenses",
  "arguments": {
    "status": "active",
    "product_id": 789,
    "customer_id": 456
  }
}
```

**validate_license** - Check license activation status

```javascript
{
  "tool": "validate_license",
  "arguments": {
    "license_key": "abc123-def456-ghi789",
    "product_id": 789
  }
}
```

## Configuration

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `EDD_API_URL` | Base URL for EDD REST API | - | Yes |
| `EDD_CONSUMER_KEY` | EDD API consumer key | - | Yes |
| `EDD_CONSUMER_SECRET` | EDD API secret key | - | Yes |
| `CACHE_ENABLED` | Enable Redis caching | `false` | No |
| `CACHE_TTL` | Cache time-to-live (seconds) | `3600` | No |
| `REDIS_URL` | Redis connection string | `redis://localhost:6379` | No |
| `LOG_LEVEL` | Logging verbosity | `info` | No |
| `RATE_LIMIT_MAX` | Max requests per window | `100` | No |
| `RATE_LIMIT_WINDOW` | Rate limit window (ms) | `60000` | No |
| `WEBHOOK_SECRET` | Secret for webhook verification | - | No |
| `ENABLE_AUDIT_LOG` | Enable audit trail | `true` | No |

### Advanced Configuration

Create `config.json` for fine-grained control:

```json
{
  "adapters": {
    "edd": {
      "timeout": 30000,
      "retries": 3,
      "backoff": "exponential"
    }
  },
  "middleware": {
    "fraud_scoring": {
      "enabled": true,
      "threshold": 0.75
    },
    "currency_conversion": {
      "base_currency": "USD",
      "provider": "openexchangerates"
    }
  },
  "analytics": {
    "retention_days": 1095,
    "aggregation_intervals": ["hourly", "daily", "monthly"]
  }
}
```

## Code Examples

### Example 1: Sales Dashboard Integration

```javascript
const CommerceVault = require('./src/commercevault');

async function generateSalesDashboard() {
  const vault = new CommerceVault({
    apiUrl: process.env.EDD_API_URL,
    consumerKey: process.env.EDD_CONSUMER_KEY,
    consumerSecret: process.env.EDD_CONSUMER_SECRET
  });

  // Get last 30 days revenue
  const report = await vault.getSalesReport({
    period: 'day',
    start_date: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000),
    end_date: new Date(),
    metrics: ['revenue', 'orders', 'avg_order_value']
  });

  // Top products
  const topProducts = await vault.getProducts({
    order_by: 'sales',
    order: 'desc',
    per_page: 10
  });

  // Customer segmentation
  const highValueCustomers = await vault.getCustomers({
    min_purchase_value: 1000,
    order_by: 'lifetime_value',
    per_page: 50
  });

  return {
    summary: report.summary,
    daily_breakdown: report.data,
    top_products: topProducts.map(p => ({
      id: p.id,
      name: p.name,
      sales: p.sales_count,
      revenue: p.total_revenue
    })),
    vip_customers: highValueCustomers.length
  };
}
```

### Example 2: Automated License Provisioning

```javascript
async function provisionLicense(orderId) {
  const vault = new CommerceVault();
  
  // Fetch order details
  const order = await vault.getOrderDetails(orderId);
  
  if (order.status !== 'completed') {
    throw new Error('Order must be completed before provisioning');
  }

  // Generate licenses for each product
  const licenses = [];
  for (const item of order.items) {
    if (item.product.type === 'software') {
      const license = await vault.createLicense({
        product_id: item.product_id,
        customer_id: order.customer_id,
        activation_limit: item.product.activation_limit || 1,
        expiration: item.product.license_expiration
      });
      
      licenses.push({
        product: item.product.name,
        key: license.key,
        activations_left: license.activation_limit
      });
    }
  }

  // Send notification
  await vault.sendLicenseEmail({
    customer_id: order.customer_id,
    licenses: licenses
  });

  return licenses;
}
```

### Example 3: Cohort Analysis

```javascript
async function analyzeCohorts() {
  const vault = new CommerceVault();
  
  // Define cohorts by signup month
  const cohorts = {};
  const customers = await vault.getCustomers({
    per_page: -1, // Get all
    date_from: '2026-01-01'
  });

  customers.forEach(customer => {
    const cohortMonth = customer.created_at.substring(0, 7); // YYYY-MM
    if (!cohorts[cohortMonth]) {
      cohorts[cohortMonth] = {
        count: 0,
        total_revenue: 0,
        repeat_rate: 0,
        customers: []
      };
    }
    cohorts[cohortMonth].count++;
    cohorts[cohortMonth].total_revenue += customer.lifetime_value;
    cohorts[cohortMonth].customers.push(customer.id);
  });

  // Calculate repeat purchase rate per cohort
  for (const [month, data] of Object.entries(cohorts)) {
    const orders = await vault.getOrders({
      customer_ids: data.customers,
      date_from: month + '-01'
    });
    
    const repeatCustomers = new Set(
      orders.filter(o => o.order_number > 1).map(o => o.customer_id)
    );
    
    data.repeat_rate = repeatCustomers.size / data.count;
    data.avg_lifetime_value = data.total_revenue / data.count;
  }

  return cohorts;
}
```

### Example 4: Real-time Webhook Handler

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();
app.use(express.json());

app.post('/webhooks/edd', async (req, res) => {
  // Verify webhook signature
  const signature = req.headers['x-edd-signature'];
  const hash = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(JSON.stringify(req.body))
    .digest('hex');

  if (signature !== hash) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  const vault = new CommerceVault();
  const event = req.body;

  switch (event.type) {
    case 'order.completed':
      // Auto-provision licenses
      await provisionLicense(event.data.order_id);
      
      // Update analytics
      await vault.recordEvent({
        type: 'sale',
        product_id: event.data.product_id,
        amount: event.data.total,
        customer_id: event.data.customer_id
      });
      break;

    case 'subscription.cancelled':
      // Revoke licenses
      await vault.revokeLicense({
        subscription_id: event.data.subscription_id
      });
      break;

    case 'refund.created':
      // Update revenue analytics
      await vault.recordRefund({
        order_id: event.data.order_id,
        amount: event.data.amount
      });
      break;
  }

  res.json({ received: true });
});

app.listen(3000);
```

## Common Patterns

### Pattern 1: Cached Product Catalog

```javascript
async function getCachedProducts(category = null) {
  const cacheKey = `products:${category || 'all'}`;
  const vault = new CommerceVault({ cache: true });
  
  let products = await vault.cache.get(cacheKey);
  
  if (!products) {
    products = await vault.getProducts({
      category: category,
      per_page: -1,
      status: 'publish'
    });
    
    await vault.cache.set(cacheKey, products, 3600); // 1 hour
  }
  
  return products;
}
```

### Pattern 2: Batch Order Processing

```javascript
async function processBulkOrders(orderIds) {
  const vault = new CommerceVault();
  const results = [];
  
  // Process in batches of 10
  const batchSize = 10;
  for (let i = 0; i < orderIds.length; i += batchSize) {
    const batch = orderIds.slice(i, i + batchSize);
    
    const promises = batch.map(orderId => 
      vault.getOrderDetails(orderId)
        .then(order => ({ success: true, order }))
        .catch(error => ({ success: false, orderId, error: error.message }))
    );
    
    const batchResults = await Promise.all(promises);
    results.push(...batchResults);
    
    // Rate limiting pause
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  return results;
}
```

### Pattern 3: Customer Segmentation Query

```javascript
async function getCustomerSegments() {
  const vault = new CommerceVault();
  
  return {
    whales: await vault.getCustomers({
      min_lifetime_value: 5000,
      order_by: 'lifetime_value'
    }),
    
    at_risk: await vault.getCustomers({
      last_purchase_before: new Date(Date.now() - 180 * 24 * 60 * 60 * 1000),
      min_purchase_count: 2
    }),
    
    new_customers: await vault.getCustomers({
      created_after: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000),
      purchase_count: 1
    }),
    
    repeat_buyers: await vault.getCustomers({
      min_purchase_count: 3,
      purchase_frequency: 'monthly'
    })
  };
}
```

## Troubleshooting

### Issue: Authentication Failures

**Symptom:** `401 Unauthorized` errors

**Solution:**
```javascript
// Verify credentials are correct
const vault = new CommerceVault({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  debug: true // Enable debug logging
});

// Test connection
try {
  await vault.testConnection();
  console.log('Connection successful');
} catch (error) {
  console.error('Auth failed:', error.message);
}
```

### Issue: Rate Limit Exceeded

**Symptom:** `429 Too Many Requests` errors

**Solution:**
```javascript
// Enable adaptive rate limiting
const vault = new CommerceVault({
  rateLimit: {
    enabled: true,
    maxRequests: 100,
    windowMs: 60000,
    strategy: 'adaptive'
  }
});

// Use queue for bulk operations
const queue = vault.createQueue();
orderIds.forEach(id => queue.add(() => vault.getOrderDetails(id)));
const results = await queue.execute();
```

### Issue: Stale Cache Data

**Symptom:** Old data returned despite updates

**Solution:**
```javascript
// Invalidate specific cache keys
await vault.cache.invalidate('products:all');
await vault.cache.invalidate(`customer:${customerId}`);

// Or clear all cache
await vault.cache.flush();

// Use event-driven invalidation
vault.on('order.completed', (event) => {
  vault.cache.invalidate(`customer:${event.customer_id}`);
  vault.cache.invalidate('sales_report:*');
});
```

### Issue: Missing Products in Response

**Symptom:** Fewer products returned than expected

**Solution:**
```javascript
// Check pagination and filters
const allProducts = [];
let page = 1;
let hasMore = true;

while (hasMore) {
  const products = await vault.getProducts({
    page: page,
    per_page: 100,
    status: 'any' // Include draft, pending, etc.
  });
  
  allProducts.push(...products);
  hasMore = products.length === 100;
  page++;
}

console.log(`Total products: ${allProducts.length}`);
```

### Issue: Webhook Verification Failures

**Symptom:** Webhooks rejected with signature mismatch

**Solution:**
```javascript
// Ensure raw body is used for signature verification
app.use(express.json({
  verify: (req, res, buf) => {
    req.rawBody = buf.toString();
  }
}));

app.post('/webhooks/edd', (req, res) => {
  const signature = req.headers['x-edd-signature'];
  const hash = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(req.rawBody) // Use raw body, not parsed JSON
    .digest('hex');
    
  // Compare using timing-safe method
  const valid = crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(hash)
  );
  
  if (!valid) {
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  // Process webhook...
});
```

## Advanced Usage

### Custom Middleware Pipeline

```javascript
const vault = new CommerceVault();

// Add fraud detection middleware
vault.use('order', async (order, next) => {
  const score = await calculateFraudScore(order);
  if (score > 0.8) {
    order.flagged = true;
    order.hold_for_review = true;
  }
  return next(order);
});

// Add currency conversion middleware
vault.use('product', async (product, next) => {
  if (product.currency !== 'USD') {
    product.price_usd = await convertCurrency(
      product.price,
      product.currency,
      'USD'
    );
  }
  return next(product);
});
```

### Analytics Aggregation

```javascript
// Schedule periodic aggregation
const cron = require('node-cron');

cron.schedule('0 * * * *', async () => {
  const vault = new CommerceVault();
  
  const hourlyMetrics = await vault.aggregateMetrics({
    period: 'hour',
    metrics: ['revenue', 'orders', 'customers'],
    timestamp: new Date()
  });
  
  await vault.storage.saveMetrics(hourlyMetrics);
});
```
