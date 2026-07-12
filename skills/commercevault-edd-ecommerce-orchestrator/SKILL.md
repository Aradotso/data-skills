---
name: commercevault-edd-ecommerce-orchestrator
description: Master the CommerceVault middleware for Easy Digital Downloads API integration, sales analytics, and digital commerce orchestration
triggers:
  - how do I connect to Easy Digital Downloads API
  - set up CommerceVault for EDD integration
  - query sales data from EDD using CommerceVault
  - manage digital product orders with CommerceVault
  - get EDD analytics and revenue reports
  - configure EDD API middleware for commerce
  - troubleshoot CommerceVault connection issues
  - fetch customer data from Easy Digital Downloads
---

# CommerceVault EDD Orchestrator

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CommerceVault (mcp-edd-analytics-vantage) is a precision-engineered middleware layer that provides unified access to Easy Digital Downloads (EDD) e-commerce data. It acts as a digital commerce orchestrator that abstracts, secures, and accelerates communication between applications and EDD REST API backends for sales analytics, product management, customer data, and order lifecycle operations.

## Core Capabilities

- **Product Catalog Synchronization** - Retrieve, filter, and cache product listings with delta sync
- **Order Lifecycle Management** - Track orders from creation through fulfillment to refunds
- **Customer Identity & Segmentation** - Manage customer profiles, LTV, and purchase metrics
- **Analytics & Revenue Streams** - Real-time dashboards for GMV, churn, AOV, and cohort analysis
- **License & Entitlement Management** - Handle license keys and activation codes for digital products
- **Webhook Relay** - Bidirectional event handling between EDD and downstream services

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- Easy Digital Downloads instance with REST API enabled
- Valid EDD API credentials (consumer key and secret)

### Setup

```bash
# Clone the repository
git clone https://github.com/dhapat3927/mcp-edd-analytics-vantage.git
cd mcp-edd-analytics-vantage

# Install dependencies (Node.js example)
npm install

# Or for Python
pip install -r requirements.txt
```

### Environment Configuration

Create a `.env` file in the project root:

```bash
# EDD API Configuration
EDD_SITE_URL=https://yoursite.com
EDD_CONSUMER_KEY=ck_your_consumer_key_here
EDD_CONSUMER_SECRET=cs_your_consumer_secret_here

# CommerceVault Settings
VAULT_API_PORT=3000
VAULT_LOG_LEVEL=info
VAULT_CACHE_ENABLED=true
VAULT_CACHE_TTL=300

# Security
VAULT_HMAC_SECRET=your_secure_hmac_secret_here
VAULT_JWT_SECRET=your_jwt_secret_here

# Optional: Redis for caching
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
```

## Authentication

CommerceVault supports multiple authentication methods:

### API Key Authentication

```javascript
const CommerceVault = require('./src/vault');

const vault = new CommerceVault({
  apiKey: process.env.VAULT_API_KEY,
  baseUrl: 'http://localhost:3000'
});
```

### OAuth/JWT Token

```javascript
const vault = new CommerceVault({
  token: process.env.VAULT_JWT_TOKEN,
  baseUrl: 'http://localhost:3000'
});
```

## Core API Operations

### Product Management

#### Fetch All Products

```javascript
// Retrieve product catalog with pagination
const products = await vault.products.list({
  page: 1,
  perPage: 50,
  status: 'publish',
  orderBy: 'date',
  order: 'desc'
});

console.log(`Retrieved ${products.length} products`);
products.forEach(product => {
  console.log(`${product.id}: ${product.title} - $${product.price}`);
});
```

#### Get Single Product

```javascript
const product = await vault.products.get(123);

console.log({
  id: product.id,
  name: product.title,
  price: product.price,
  downloads: product.sales,
  earnings: product.earnings
});
```

#### Filter Products by Category

```javascript
const categoryProducts = await vault.products.list({
  category: 'plugins',
  status: 'publish',
  minPrice: 10,
  maxPrice: 100
});
```

### Order Operations

#### Retrieve Recent Orders

```javascript
// Get orders from last 7 days
const startDate = new Date();
startDate.setDate(startDate.getDate() - 7);

const orders = await vault.orders.list({
  startDate: startDate.toISOString(),
  endDate: new Date().toISOString(),
  status: 'complete'
});

console.log(`Total orders: ${orders.length}`);
console.log(`Revenue: $${orders.reduce((sum, o) => sum + o.total, 0)}`);
```

#### Get Order Details

```javascript
const order = await vault.orders.get(5678);

console.log({
  orderId: order.id,
  customer: order.customer_email,
  total: order.total,
  status: order.status,
  items: order.items.map(i => ({
    product: i.product_name,
    price: i.price
  }))
});
```

#### Create New Order

```javascript
const newOrder = await vault.orders.create({
  customer: {
    email: 'customer@example.com',
    firstName: 'John',
    lastName: 'Doe'
  },
  items: [
    {
      productId: 123,
      quantity: 1,
      price: 49.99
    }
  ],
  paymentMethod: 'stripe',
  status: 'pending'
});
```

### Customer Management

#### Fetch Customer Data

```javascript
const customer = await vault.customers.get('customer@example.com');

console.log({
  email: customer.email,
  name: `${customer.firstName} ${customer.lastName}`,
  totalSpent: customer.lifetime_value,
  orderCount: customer.purchase_count,
  lastPurchase: customer.last_purchase_date
});
```

#### Customer Segmentation

```javascript
// Get high-value customers (LTV > $500)
const vipCustomers = await vault.customers.segment({
  minLifetimeValue: 500,
  minPurchases: 3,
  activeWithinDays: 90
});

console.log(`VIP Customers: ${vipCustomers.length}`);
```

### Analytics & Reporting

#### Sales Analytics

```javascript
const analytics = await vault.analytics.getSales({
  startDate: '2026-01-01',
  endDate: '2026-12-31',
  groupBy: 'month'
});

analytics.periods.forEach(period => {
  console.log(`${period.label}: $${period.revenue} (${period.orders} orders)`);
});
```

#### Product Performance Report

```javascript
const topProducts = await vault.analytics.getTopProducts({
  limit: 10,
  metric: 'revenue',
  period: 'last_30_days'
});

topProducts.forEach((product, index) => {
  console.log(`${index + 1}. ${product.name} - $${product.revenue}`);
});
```

#### Revenue Cohort Analysis

```javascript
const cohorts = await vault.analytics.getCohorts({
  cohortType: 'monthly',
  startMonth: '2026-01',
  metric: 'retention'
});

console.log('Customer Retention by Cohort:');
cohorts.forEach(cohort => {
  console.log(`${cohort.period}: ${cohort.retention}%`);
});
```

### License Management

#### Generate License Key

```javascript
const license = await vault.licenses.create({
  productId: 123,
  customerId: 456,
  activationLimit: 3,
  expirationDate: '2027-12-31'
});

console.log(`License Key: ${license.key}`);
console.log(`Activations Remaining: ${license.activationsRemaining}`);
```

#### Validate License

```javascript
const validation = await vault.licenses.validate({
  key: 'XXXX-XXXX-XXXX-XXXX',
  productId: 123,
  siteUrl: 'https://customer-site.com'
});

if (validation.valid) {
  console.log('License is valid and active');
} else {
  console.error(`License invalid: ${validation.reason}`);
}
```

## Common Patterns

### Batch Product Sync with Delta Updates

```javascript
async function syncProducts() {
  const lastSync = await vault.cache.get('last_product_sync');
  
  const products = await vault.products.list({
    modifiedSince: lastSync || new Date(0),
    perPage: 100
  });
  
  for (const product of products) {
    await saveToLocalDatabase(product);
  }
  
  await vault.cache.set('last_product_sync', new Date().toISOString());
  console.log(`Synced ${products.length} products`);
}
```

### Real-time Order Processing with Webhooks

```javascript
const express = require('express');
const app = express();

app.post('/webhooks/edd/order-completed', async (req, res) => {
  const signature = req.headers['x-edd-signature'];
  
  // Verify webhook signature
  if (!vault.webhooks.verify(req.body, signature)) {
    return res.status(401).send('Invalid signature');
  }
  
  const order = req.body;
  
  // Process order
  await fulfillDigitalProduct(order);
  await sendConfirmationEmail(order.customer_email);
  
  res.status(200).send('OK');
});
```

### Revenue Dashboard with Caching

```javascript
async function getDashboardMetrics() {
  const cacheKey = 'dashboard_metrics';
  const cached = await vault.cache.get(cacheKey);
  
  if (cached) return cached;
  
  const [revenue, orders, customers] = await Promise.all([
    vault.analytics.getTotalRevenue({ period: 'this_month' }),
    vault.analytics.getOrderCount({ period: 'this_month' }),
    vault.customers.count({ activeWithinDays: 30 })
  ]);
  
  const metrics = {
    revenue: revenue.total,
    orders: orders.count,
    avgOrderValue: revenue.total / orders.count,
    activeCustomers: customers
  };
  
  // Cache for 5 minutes
  await vault.cache.set(cacheKey, metrics, 300);
  
  return metrics;
}
```

### Error Handling with Retry Logic

```javascript
async function fetchWithRetry(operation, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (error.status === 429) {
        // Rate limit - wait and retry
        const waitTime = Math.pow(2, attempt) * 1000;
        console.log(`Rate limited. Retrying in ${waitTime}ms...`);
        await new Promise(resolve => setTimeout(resolve, waitTime));
      } else if (attempt === maxRetries) {
        throw error;
      }
    }
  }
}

// Usage
const products = await fetchWithRetry(() => vault.products.list());
```

## Configuration

### Adapter Configuration

CommerceVault uses a hexagonal architecture with pluggable adapters:

```javascript
// config/adapters.js
module.exports = {
  edd: {
    enabled: true,
    baseUrl: process.env.EDD_SITE_URL,
    auth: {
      consumerKey: process.env.EDD_CONSUMER_KEY,
      consumerSecret: process.env.EDD_CONSUMER_SECRET
    },
    rateLimit: {
      maxRequests: 100,
      perSeconds: 60
    }
  }
};
```

### Middleware Pipeline

```javascript
// config/middleware.js
module.exports = {
  pipeline: [
    'authentication',
    'logging',
    'rateLimiting',
    'caching',
    'transformation',
    'validation'
  ],
  
  custom: {
    geoFencing: {
      enabled: true,
      allowedRegions: ['US', 'CA', 'EU']
    },
    fraudScoring: {
      enabled: true,
      threshold: 0.7
    }
  }
};
```

### Cache Configuration

```javascript
// config/cache.js
module.exports = {
  enabled: true,
  driver: 'redis', // or 'memory'
  ttl: {
    products: 600,      // 10 minutes
    customers: 300,     // 5 minutes
    analytics: 900      // 15 minutes
  },
  invalidation: {
    onUpdate: true,
    onWebhook: true
  }
};
```

## CLI Commands

If CommerceVault includes a CLI tool:

```bash
# Initialize new project
commercevault init

# Test API connection
commercevault test-connection

# Sync products
commercevault sync products --full

# Generate analytics report
commercevault report sales --start 2026-01-01 --end 2026-12-31 --format csv

# Webhook management
commercevault webhooks register --event order.completed --url https://your-app.com/webhooks

# Cache operations
commercevault cache clear --all
commercevault cache stats
```

## Troubleshooting

### Connection Issues

```javascript
// Test basic connectivity
try {
  const status = await vault.health.check();
  console.log('API Status:', status);
} catch (error) {
  console.error('Connection failed:', error.message);
  // Check: API credentials, network access, SSL certificates
}
```

### Rate Limiting

```javascript
// Monitor rate limit headers
const response = await vault.products.list();
console.log('Rate Limit Remaining:', response.headers['x-rate-limit-remaining']);
console.log('Rate Limit Reset:', response.headers['x-rate-limit-reset']);
```

### Authentication Errors

```javascript
// Verify credentials
if (error.status === 401) {
  console.error('Authentication failed. Check:');
  console.error('- EDD_CONSUMER_KEY is correct');
  console.error('- EDD_CONSUMER_SECRET is correct');
  console.error('- REST API is enabled in EDD settings');
  console.error('- API user has proper permissions');
}
```

### Data Synchronization Issues

```javascript
// Enable detailed logging
const vault = new CommerceVault({
  logLevel: 'debug',
  logRequests: true,
  logResponses: true
});

// Check sync status
const syncStatus = await vault.sync.getStatus();
console.log('Last sync:', syncStatus.lastSync);
console.log('Errors:', syncStatus.errors);
```

### Cache Invalidation

```javascript
// Manual cache clear for specific keys
await vault.cache.delete('products:*');
await vault.cache.delete('customer:*');

// Or clear all
await vault.cache.flush();
```

### Webhook Debugging

```javascript
// Log webhook payloads for debugging
app.post('/webhooks/edd/*', (req, res, next) => {
  console.log('Webhook received:', {
    path: req.path,
    headers: req.headers,
    body: req.body
  });
  next();
});
```

## Security Best Practices

### Signature Verification

```javascript
// Always verify HMAC signatures
function verifyPayload(payload, signature) {
  const hmac = crypto.createHmac('sha256', process.env.VAULT_HMAC_SECRET);
  const computed = hmac.update(JSON.stringify(payload)).digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(computed)
  );
}
```

### Secure Credential Storage

Never hardcode credentials. Always use environment variables:

```javascript
// ✅ Good
const vault = new CommerceVault({
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET
});

// ❌ Bad
const vault = new CommerceVault({
  consumerKey: 'ck_12345',
  consumerSecret: 'cs_67890'
});
```

### API Key Rotation

```javascript
// Implement key rotation schedule
async function rotateApiKeys() {
  const newKeys = await vault.auth.generateNewKeys();
  
  // Update environment
  await updateEnvironmentVariable('EDD_CONSUMER_KEY', newKeys.consumerKey);
  await updateEnvironmentVariable('EDD_CONSUMER_SECRET', newKeys.consumerSecret);
  
  // Revoke old keys after grace period
  setTimeout(async () => {
    await vault.auth.revokeOldKeys();
  }, 24 * 60 * 60 * 1000); // 24 hours
}
```
