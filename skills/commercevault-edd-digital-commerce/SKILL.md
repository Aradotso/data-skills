---
name: commercevault-edd-digital-commerce
description: MCP server for Easy Digital Downloads API integration with sales analytics, product management, and revenue reporting
triggers:
  - "connect to Easy Digital Downloads API"
  - "fetch EDD sales data and analytics"
  - "integrate with WordPress commerce backend"
  - "query digital product sales metrics"
  - "manage EDD customer and order data"
  - "retrieve WordPress ecommerce analytics"
  - "build EDD sales dashboard"
  - "sync digital downloads inventory"
---

# CommerceVault EDD Analytics Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CommerceVault (mcp-edd-analytics-vantage) is an MCP (Model Context Protocol) server that provides a unified interface for interacting with Easy Digital Downloads (EDD) WordPress e-commerce installations. It abstracts REST API complexity into structured tools for retrieving sales data, managing products, analyzing customer behavior, and generating revenue reports.

## What It Does

- **Sales & Revenue Analytics**: Query gross merchandise value (GMV), average order value (AOV), revenue by period
- **Product Catalog Management**: List, filter, and sync digital product catalogs with pricing and download details
- **Order Lifecycle**: Retrieve orders, track fulfillment status, process refunds
- **Customer Segmentation**: Fetch customer profiles, lifetime value (LTV), purchase history
- **License Management**: Handle software license keys and entitlements for digital products
- **Real-time Reporting**: Generate dashboards with product performance and cohort analysis

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- Access to an Easy Digital Downloads WordPress installation
- EDD REST API credentials (API key or OAuth token)

### As MCP Server

Add to your MCP client configuration (e.g., Claude Desktop):

```json
{
  "mcpServers": {
    "edd-analytics": {
      "command": "npx",
      "args": ["-y", "mcp-edd-analytics-vantage"],
      "env": {
        "EDD_API_URL": "https://your-site.com/wp-json/edd/v2",
        "EDD_API_KEY": "ck_xxxxxxxxxxxxx",
        "EDD_API_SECRET": "cs_xxxxxxxxxxxxx"
      }
    }
  }
}
```

### As Standalone Service

```bash
# Clone repository
git clone https://github.com/dhapat3927/mcp-edd-analytics-vantage.git
cd mcp-edd-analytics-vantage

# Install dependencies
npm install
# or
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your EDD credentials
```

## Configuration

### Environment Variables

```bash
# Required: EDD API endpoint
EDD_API_URL=https://yourstore.com/wp-json/edd/v2

# Required: Authentication (choose one method)
EDD_API_KEY=ck_your_consumer_key
EDD_API_SECRET=cs_your_consumer_secret

# OR use OAuth
EDD_OAUTH_TOKEN=your_oauth_token

# Optional: Performance tuning
EDD_CACHE_TTL=300                    # Cache expiration in seconds
EDD_RATE_LIMIT_PER_MIN=60           # Max requests per minute
EDD_REQUEST_TIMEOUT=30000            # Timeout in milliseconds

# Optional: Data residency
EDD_DATA_REGION=EU                   # EU, US, APAC for GDPR compliance
EDD_AUDIT_LOG_ENABLED=true          # Enable immutable audit logging
```

### Adapter Configuration

For multi-platform support, configure adapters in `config.json`:

```json
{
  "adapters": {
    "edd": {
      "endpoint": "${EDD_API_URL}",
      "auth": {
        "type": "basic",
        "key": "${EDD_API_KEY}",
        "secret": "${EDD_API_SECRET}"
      },
      "features": ["products", "orders", "customers", "licenses"]
    }
  },
  "cache": {
    "enabled": true,
    "backend": "redis",
    "ttl": 300,
    "invalidateOnWebhook": true
  },
  "security": {
    "hmacVerification": true,
    "hmacSecret": "${HMAC_SECRET}"
  }
}
```

## Core API Tools

### Product Operations

#### List Products

```javascript
// Fetch all digital products
const products = await commerceVault.listProducts({
  status: 'publish',
  type: 'download',
  per_page: 50,
  orderby: 'date',
  order: 'desc'
});

// Filter by category
const softwareProducts = await commerceVault.listProducts({
  category: 'software',
  status: 'publish'
});

// With pricing and license info
const detailedProducts = await commerceVault.listProducts({
  include_pricing: true,
  include_licensing: true
});
```

#### Get Product Details

```javascript
const product = await commerceVault.getProduct({
  id: 1234,
  include: ['variations', 'downloads', 'reviews']
});

console.log(product.name);
console.log(product.price);
console.log(product.licensing_model); // 'per_site', 'unlimited', etc.
```

### Sales Analytics

#### Revenue Reports

```javascript
// Daily revenue for last 30 days
const revenue = await commerceVault.getRevenue({
  period: 'daily',
  start_date: '2026-06-01',
  end_date: '2026-06-30',
  currency: 'USD'
});

revenue.forEach(day => {
  console.log(`${day.date}: $${day.gross_revenue}`);
});

// Monthly GMV breakdown
const monthlyGMV = await commerceVault.getRevenue({
  period: 'monthly',
  start_date: '2026-01-01',
  end_date: '2026-12-31',
  metrics: ['gmv', 'net_revenue', 'refunds']
});
```

#### Product Performance

```javascript
const topProducts = await commerceVault.getProductPerformance({
  period: '30d',
  sort_by: 'revenue',
  limit: 10,
  include_refund_rate: true
});

topProducts.forEach(p => {
  console.log(`${p.name}: $${p.revenue} (${p.units_sold} units)`);
});
```

### Order Management

#### Retrieve Orders

```javascript
// Recent orders
const orders = await commerceVault.listOrders({
  status: 'completed',
  per_page: 100,
  date_range: {
    after: '2026-06-01T00:00:00',
    before: '2026-06-30T23:59:59'
  }
});

// Filter by customer
const customerOrders = await commerceVault.listOrders({
  customer_id: 567,
  status: ['completed', 'refunded']
});
```

#### Get Order Details

```javascript
const order = await commerceVault.getOrder({
  id: 8901,
  include: ['customer', 'line_items', 'licenses']
});

console.log(`Order #${order.number}`);
console.log(`Total: ${order.total} ${order.currency}`);
console.log(`Customer: ${order.customer.email}`);
order.line_items.forEach(item => {
  console.log(`  - ${item.product_name}: ${item.quantity}`);
});
```

#### Process Refund

```javascript
const refund = await commerceVault.createRefund({
  order_id: 8901,
  amount: 49.99,
  reason: 'Customer request',
  revoke_licenses: true // Deactivate associated licenses
});

console.log(`Refund ${refund.id} processed`);
```

### Customer Insights

#### Customer Profile

```javascript
const customer = await commerceVault.getCustomer({
  id: 567,
  include: ['purchase_history', 'ltv', 'licenses']
});

console.log(`${customer.name} (${customer.email})`);
console.log(`Lifetime Value: $${customer.lifetime_value}`);
console.log(`Total Orders: ${customer.total_orders}`);
console.log(`Active Licenses: ${customer.active_licenses.length}`);
```

#### Cohort Analysis

```javascript
const cohorts = await commerceVault.getCohortAnalysis({
  cohort_by: 'signup_month',
  start_date: '2026-01-01',
  end_date: '2026-06-30',
  metrics: ['retention_rate', 'avg_order_value', 'churn_rate']
});

cohorts.forEach(cohort => {
  console.log(`${cohort.period}: Retention ${cohort.retention_rate}%`);
});
```

### License & Entitlement

#### List Licenses

```javascript
const licenses = await commerceVault.listLicenses({
  status: 'active',
  product_id: 1234,
  per_page: 100
});

licenses.forEach(license => {
  console.log(`${license.key}: ${license.activations}/${license.activation_limit}`);
});
```

#### Validate License

```javascript
const validation = await commerceVault.validateLicense({
  license_key: 'XXXX-XXXX-XXXX-XXXX',
  product_id: 1234,
  site_url: 'https://customer-site.com'
});

if (validation.valid) {
  console.log('License valid');
  console.log(`Expires: ${validation.expires_at}`);
} else {
  console.log(`Invalid: ${validation.reason}`);
}
```

## Common Patterns

### Building a Sales Dashboard

```javascript
async function buildSalesDashboard(period = '30d') {
  // Parallel fetch for performance
  const [revenue, topProducts, orderStats] = await Promise.all([
    commerceVault.getRevenue({ period: 'daily', last: period }),
    commerceVault.getProductPerformance({ period, limit: 5 }),
    commerceVault.getOrderStatistics({ period })
  ]);

  return {
    totalRevenue: revenue.reduce((sum, day) => sum + day.gross_revenue, 0),
    averageOrderValue: orderStats.average_order_value,
    conversionRate: orderStats.conversion_rate,
    topSellingProducts: topProducts.map(p => ({
      name: p.name,
      revenue: p.revenue,
      units: p.units_sold
    })),
    dailyTrend: revenue.map(day => ({
      date: day.date,
      revenue: day.gross_revenue
    }))
  };
}
```

### Syncing Product Catalog

```javascript
async function syncProductCatalog() {
  let page = 1;
  let allProducts = [];
  let hasMore = true;

  while (hasMore) {
    const response = await commerceVault.listProducts({
      per_page: 100,
      page,
      status: 'publish',
      _fields: 'id,name,price,sku,stock_quantity' // Optimize payload
    });

    allProducts = allProducts.concat(response.products);
    hasMore = response.total_pages > page;
    page++;
  }

  // Store in local database or cache
  await db.products.bulkUpsert(allProducts);
  console.log(`Synced ${allProducts.length} products`);
}
```

### Customer LTV Calculation

```javascript
async function calculateCustomerLTV(customerId) {
  const orders = await commerceVault.listOrders({
    customer_id: customerId,
    status: 'completed',
    per_page: -1 // Get all orders
  });

  const totalSpent = orders.reduce((sum, order) => {
    return sum + parseFloat(order.total);
  }, 0);

  const avgOrderValue = totalSpent / orders.length;
  const purchaseFrequency = orders.length / 
    ((Date.now() - new Date(orders[0].date_created)) / (1000 * 60 * 60 * 24 * 365));

  return {
    total_spent: totalSpent,
    order_count: orders.length,
    avg_order_value: avgOrderValue,
    purchase_frequency: purchaseFrequency,
    estimated_ltv: avgOrderValue * purchaseFrequency * 3 // 3-year projection
  };
}
```

### Webhook Handler

```javascript
// Express.js webhook endpoint
app.post('/webhooks/edd', async (req, res) => {
  const event = req.body;
  const signature = req.headers['x-edd-signature'];

  // Verify HMAC signature
  if (!commerceVault.verifyWebhookSignature(event, signature)) {
    return res.status(401).send('Invalid signature');
  }

  // Handle events
  switch (event.event) {
    case 'order.completed':
      await handleOrderCompleted(event.data.order);
      break;
    case 'license.activated':
      await handleLicenseActivation(event.data.license);
      break;
    case 'subscription.renewed':
      await handleSubscriptionRenewal(event.data.subscription);
      break;
  }

  res.status(200).send('OK');
});
```

## Python Usage

```python
from commercevault import CommerceVaultClient
import os

# Initialize client
client = CommerceVaultClient(
    api_url=os.getenv('EDD_API_URL'),
    api_key=os.getenv('EDD_API_KEY'),
    api_secret=os.getenv('EDD_API_SECRET')
)

# Fetch products
products = client.products.list(
    status='publish',
    per_page=50
)

for product in products:
    print(f"{product.name}: ${product.price}")

# Revenue report
revenue = client.analytics.get_revenue(
    period='monthly',
    start_date='2026-01-01',
    end_date='2026-12-31'
)

total = sum(month['gross_revenue'] for month in revenue)
print(f"Annual Revenue: ${total}")

# Customer insights
customer = client.customers.get(567)
print(f"LTV: ${customer.lifetime_value}")
```

## Troubleshooting

### Authentication Failures

```javascript
// Test credentials
try {
  await commerceVault.testConnection();
  console.log('Connection successful');
} catch (error) {
  console.error('Auth failed:', error.message);
  // Check: EDD_API_KEY and EDD_API_SECRET are correct
  // Verify: REST API is enabled in EDD settings
  // Confirm: Key has required permissions
}
```

### Rate Limiting

```javascript
// Handle 429 errors with retry
const result = await commerceVault.listProducts({
  per_page: 100
}).catch(async (error) => {
  if (error.status === 429) {
    const retryAfter = error.headers['retry-after'] || 60;
    console.log(`Rate limited, retrying after ${retryAfter}s`);
    await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
    return commerceVault.listProducts({ per_page: 100 });
  }
  throw error;
});
```

### Cache Invalidation

```javascript
// Force fresh data
const products = await commerceVault.listProducts({
  bypass_cache: true
});

// Manual cache clear
await commerceVault.cache.invalidate('products:*');
```

### Debug Mode

```bash
# Enable verbose logging
export DEBUG=commercevault:*
export LOG_LEVEL=debug

# Run with trace
node --trace-warnings index.js
```

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `ECONNREFUSED` | EDD API unreachable | Check `EDD_API_URL`, verify site is accessible |
| `401 Unauthorized` | Invalid credentials | Regenerate API keys in EDD settings |
| `403 Forbidden` | Insufficient permissions | Grant "Read/Write" permissions to API key |
| `404 Not Found` | Resource doesn't exist | Verify IDs, check if resource was deleted |
| `429 Too Many Requests` | Rate limit exceeded | Implement exponential backoff, reduce request frequency |

## Advanced Features

### Custom Middleware

```javascript
// Add fraud detection
commerceVault.use('order:before', async (order) => {
  const riskScore = await fraudService.evaluate(order);
  if (riskScore > 80) {
    throw new Error('High risk order flagged');
  }
  return order;
});

// Currency conversion
commerceVault.use('revenue:after', async (revenue) => {
  if (revenue.currency !== 'USD') {
    revenue.usd_equivalent = await currencyAPI.convert(
      revenue.amount,
      revenue.currency,
      'USD'
    );
  }
  return revenue;
});
```

### Multi-Store Aggregation

```javascript
const stores = [
  { name: 'Store A', url: process.env.STORE_A_URL, key: process.env.STORE_A_KEY },
  { name: 'Store B', url: process.env.STORE_B_URL, key: process.env.STORE_B_KEY }
];

const aggregatedRevenue = await Promise.all(
  stores.map(async store => {
    const client = new CommerceVaultClient(store);
    const revenue = await client.analytics.get_revenue({ period: '30d' });
    return { store: store.name, revenue };
  })
);

const total = aggregatedRevenue.reduce(
  (sum, store) => sum + store.revenue.reduce((s, day) => s + day.gross_revenue, 0),
  0
);
```
