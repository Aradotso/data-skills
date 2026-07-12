---
name: commercevault-edd-digital-commerce
description: Master the CommerceVault middleware for Easy Digital Downloads API integration, sales analytics, order management, and digital product orchestration.
triggers:
  - how do I integrate with Easy Digital Downloads using CommerceVault
  - set up EDD commerce middleware for sales tracking
  - query EDD orders and products through unified API
  - implement digital commerce analytics with CommerceVault
  - connect to WordPress EDD store programmatically
  - manage digital product licenses and entitlements
  - fetch EDD customer data and revenue metrics
  - configure commerce vault for digital downloads
---

# CommerceVault EDD Digital Commerce Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

## Overview

CommerceVault (mcp-edd-analytics-vantage) is a middleware orchestrator that provides a unified, secure interface to Easy Digital Downloads (EDD) WordPress stores. It abstracts the EDD REST API with enhanced features including cryptographic verification, adaptive rate limiting, caching, analytics aggregation, and webhook management. The system uses hexagonal architecture with pluggable adapters, making it extensible to other e-commerce platforms while maintaining consistent interfaces.

**Key Capabilities:**
- Product catalog synchronization with delta updates
- Order lifecycle management (creation, fulfillment, refunds)
- Customer segmentation and lifetime value tracking
- Real-time analytics (GMV, AOV, churn, cohort analysis)
- License key and entitlement management for digital products
- Multilingual support (17 languages)
- HMAC payload verification for data integrity

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+ (depending on client SDK)
- Access to an EDD-enabled WordPress site
- EDD REST API credentials (consumer key and secret)
- Redis (optional, for caching layer)

### NPM Installation (Node.js)

```bash
npm install @commercevault/edd-client
```

### Python Installation

```bash
pip install commercevault-edd
```

### Docker Deployment

```bash
docker pull commercevault/edd-orchestrator:latest

docker run -d \
  -p 8080:8080 \
  -e EDD_API_URL=$EDD_API_URL \
  -e EDD_CONSUMER_KEY=$EDD_CONSUMER_KEY \
  -e EDD_CONSUMER_SECRET=$EDD_CONSUMER_SECRET \
  -e REDIS_URL=$REDIS_URL \
  commercevault/edd-orchestrator:latest
```

### Environment Variables

```bash
# Required
EDD_API_URL=https://your-store.com
EDD_CONSUMER_KEY=ck_xxxxxxxxxxxxx
EDD_CONSUMER_SECRET=cs_xxxxxxxxxxxxx

# Optional
REDIS_URL=redis://localhost:6379
CACHE_TTL=3600
RATE_LIMIT_REQUESTS=100
RATE_LIMIT_WINDOW=60
HMAC_SECRET_KEY=$HMAC_SECRET_KEY
LOG_LEVEL=info
```

## Configuration

### Basic Configuration (Node.js)

```javascript
const { CommerceVaultClient } = require('@commercevault/edd-client');

const client = new CommerceVaultClient({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  hmacSecret: process.env.HMAC_SECRET_KEY,
  enableCache: true,
  cacheTTL: 3600,
  timeout: 30000
});
```

### Advanced Configuration with Middleware

```javascript
const client = new CommerceVaultClient({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  middleware: [
    // Geo-fencing middleware
    {
      name: 'geoFilter',
      handler: async (request, next) => {
        if (request.customer.country === 'RESTRICTED') {
          throw new Error('Region not supported');
        }
        return next(request);
      }
    },
    // Currency normalization
    {
      name: 'currencyNormalizer',
      handler: async (request, next) => {
        const response = await next(request);
        response.data.amount = convertToBaseCurrency(
          response.data.amount,
          response.data.currency
        );
        return response;
      }
    }
  ],
  rateLimiting: {
    maxRequests: 100,
    windowSeconds: 60,
    strategy: 'adaptive'
  }
});
```

## Core API Methods

### Product Operations

#### Fetch All Products

```javascript
// Retrieve product catalog with pagination
const products = await client.products.list({
  page: 1,
  perPage: 50,
  status: 'publish',
  orderBy: 'date',
  order: 'desc'
});

console.log(`Found ${products.total} products`);
products.items.forEach(product => {
  console.log(`${product.id}: ${product.title} - $${product.price}`);
});
```

#### Get Single Product

```javascript
const product = await client.products.get(12345);

console.log(`Product: ${product.title}`);
console.log(`SKU: ${product.sku}`);
console.log(`Price: $${product.price}`);
console.log(`Downloads: ${product.sales_count}`);
console.log(`Licensing: ${product.licensing_enabled}`);
```

#### Delta Sync Products

```javascript
// Only fetch products modified since last sync
const lastSyncTime = '2026-07-01T00:00:00Z';
const changedProducts = await client.products.listDelta({
  modifiedSince: lastSyncTime,
  includeDeleted: true
});

console.log(`${changedProducts.added.length} new products`);
console.log(`${changedProducts.updated.length} updated products`);
console.log(`${changedProducts.deleted.length} deleted products`);
```

### Order Operations

#### Retrieve Orders

```javascript
// Fetch recent orders with filtering
const orders = await client.orders.list({
  status: ['completed', 'processing'],
  dateFrom: '2026-07-01',
  dateTo: '2026-07-12',
  perPage: 100
});

let totalRevenue = 0;
orders.items.forEach(order => {
  totalRevenue += parseFloat(order.total);
  console.log(`Order #${order.number}: $${order.total} - ${order.status}`);
});

console.log(`Total Revenue: $${totalRevenue.toFixed(2)}`);
```

#### Get Single Order with Line Items

```javascript
const order = await client.orders.get(98765, {
  includeLineItems: true,
  includeCustomer: true,
  includeLicenses: true
});

console.log(`Order: #${order.number}`);
console.log(`Customer: ${order.customer.email}`);
console.log(`Total: $${order.total}`);

order.lineItems.forEach(item => {
  console.log(`  - ${item.product_name} x${item.quantity}: $${item.subtotal}`);
  if (item.licenses) {
    console.log(`    License Key: ${item.licenses[0].key}`);
  }
});
```

#### Create Order Programmatically

```javascript
const newOrder = await client.orders.create({
  customer: {
    email: 'customer@example.com',
    firstName: 'Jane',
    lastName: 'Doe'
  },
  lineItems: [
    {
      productId: 123,
      quantity: 1,
      priceId: 0 // Default price
    }
  ],
  paymentMethod: 'stripe',
  status: 'pending',
  metadata: {
    source: 'api',
    campaign: 'summer-sale'
  }
});

console.log(`Created order #${newOrder.number}`);
console.log(`Payment URL: ${newOrder.paymentUrl}`);
```

#### Process Refund

```javascript
const refund = await client.orders.refund(98765, {
  amount: 29.99,
  reason: 'Customer requested refund',
  revokeLicenses: true,
  notifyCustomer: true
});

console.log(`Refund ID: ${refund.id}`);
console.log(`Amount: $${refund.amount}`);
console.log(`Status: ${refund.status}`);
```

### Customer Operations

#### Fetch Customer Data

```javascript
const customer = await client.customers.get(4567, {
  includeOrders: true,
  includeStats: true
});

console.log(`Customer: ${customer.email}`);
console.log(`Total Orders: ${customer.stats.orderCount}`);
console.log(`Lifetime Value: $${customer.stats.lifetimeValue}`);
console.log(`Average Order Value: $${customer.stats.averageOrderValue}`);
console.log(`Last Purchase: ${customer.stats.lastPurchaseDate}`);
```

#### Customer Segmentation

```javascript
// Find high-value customers
const segments = await client.customers.segment({
  criteria: {
    lifetimeValueMin: 500,
    orderCountMin: 5,
    lastPurchaseDays: 90
  },
  limit: 100
});

console.log(`Found ${segments.customers.length} VIP customers`);

// Export to CRM
await client.customers.exportSegment(segments.id, {
  format: 'csv',
  destination: 's3://bucket/segments/vip-customers.csv'
});
```

### Analytics & Reporting

#### Revenue Analytics

```javascript
const analytics = await client.analytics.revenue({
  dateFrom: '2026-06-01',
  dateTo: '2026-06-30',
  groupBy: 'day',
  includeTax: true,
  includeRefunds: true
});

console.log(`Total Revenue: $${analytics.totalRevenue}`);
console.log(`Gross Merchandise Value: $${analytics.gmv}`);
console.log(`Average Order Value: $${analytics.aov}`);
console.log(`Order Count: ${analytics.orderCount}`);

// Daily breakdown
analytics.timeSeries.forEach(day => {
  console.log(`${day.date}: $${day.revenue} (${day.orders} orders)`);
});
```

#### Product Performance

```javascript
const productStats = await client.analytics.productPerformance({
  dateFrom: '2026-06-01',
  dateTo: '2026-06-30',
  limit: 10,
  sortBy: 'revenue'
});

console.log('Top 10 Products by Revenue:');
productStats.products.forEach((product, index) => {
  console.log(`${index + 1}. ${product.name}`);
  console.log(`   Revenue: $${product.revenue}`);
  console.log(`   Units Sold: ${product.unitsSold}`);
  console.log(`   Conversion Rate: ${product.conversionRate}%`);
});
```

#### Cohort Analysis

```javascript
const cohorts = await client.analytics.cohortAnalysis({
  cohortType: 'month',
  dateFrom: '2026-01-01',
  dateTo: '2026-06-30',
  metric: 'revenue'
});

cohorts.cohorts.forEach(cohort => {
  console.log(`Cohort: ${cohort.period}`);
  console.log(`Customers: ${cohort.customerCount}`);
  console.log(`Month 0: $${cohort.periods[0]}`);
  console.log(`Month 1: $${cohort.periods[1]}`);
  console.log(`Month 3: $${cohort.periods[3]}`);
  console.log(`Retention Rate: ${cohort.retentionRate}%`);
});
```

### License Management

#### Generate License Keys

```javascript
const licenses = await client.licenses.generate({
  productId: 123,
  quantity: 50,
  expirationDays: 365,
  activationLimit: 3,
  metadata: {
    batch: 'bulk-order-2026',
    distributor: 'partner-xyz'
  }
});

console.log(`Generated ${licenses.keys.length} licenses`);
licenses.keys.forEach(license => {
  console.log(`Key: ${license.key}`);
  console.log(`Expires: ${license.expirationDate}`);
});
```

#### Validate License

```javascript
const validation = await client.licenses.validate({
  key: 'XXXX-XXXX-XXXX-XXXX',
  productId: 123,
  siteUrl: 'https://customer-site.com'
});

if (validation.valid) {
  console.log('License is valid');
  console.log(`Activations: ${validation.activationCount}/${validation.activationLimit}`);
  console.log(`Expires: ${validation.expirationDate}`);
} else {
  console.log(`License invalid: ${validation.reason}`);
}
```

#### Revoke License

```javascript
await client.licenses.revoke('XXXX-XXXX-XXXX-XXXX', {
  reason: 'Refund issued',
  notifyCustomer: true
});

console.log('License revoked successfully');
```

## Python SDK Usage

### Basic Client Setup

```python
from commercevault import CommerceVaultClient
import os

client = CommerceVaultClient(
    api_url=os.environ['EDD_API_URL'],
    consumer_key=os.environ['EDD_CONSUMER_KEY'],
    consumer_secret=os.environ['EDD_CONSUMER_SECRET'],
    hmac_secret=os.environ.get('HMAC_SECRET_KEY'),
    enable_cache=True
)
```

### Async Operations

```python
import asyncio
from commercevault import AsyncCommerceVaultClient

async def fetch_dashboard_data():
    async with AsyncCommerceVaultClient(
        api_url=os.environ['EDD_API_URL'],
        consumer_key=os.environ['EDD_CONSUMER_KEY'],
        consumer_secret=os.environ['EDD_CONSUMER_SECRET']
    ) as client:
        # Parallel requests
        orders, products, analytics = await asyncio.gather(
            client.orders.list(status='completed', limit=100),
            client.products.list(limit=50),
            client.analytics.revenue(date_from='2026-07-01')
        )
        
        return {
            'order_count': len(orders.items),
            'product_count': len(products.items),
            'revenue': analytics.total_revenue
        }

dashboard = asyncio.run(fetch_dashboard_data())
print(f"Orders: {dashboard['order_count']}")
print(f"Products: {dashboard['product_count']}")
print(f"Revenue: ${dashboard['revenue']}")
```

### Data Export Pipeline

```python
import pandas as pd

# Export orders to DataFrame for analysis
orders = client.orders.list(
    date_from='2026-01-01',
    date_to='2026-06-30',
    per_page=1000
)

df = pd.DataFrame([{
    'order_id': order.id,
    'date': order.date_created,
    'total': float(order.total),
    'status': order.status,
    'customer_email': order.customer.email,
    'product_count': len(order.line_items)
} for order in orders.items])

# Analyze monthly trends
monthly_revenue = df.groupby(df['date'].dt.to_period('M'))['total'].sum()
print(monthly_revenue)

# Export to CSV
df.to_csv('orders_export.csv', index=False)
```

## Webhook Management

### Register Webhook

```javascript
const webhook = await client.webhooks.register({
  url: 'https://your-app.com/webhooks/edd',
  events: [
    'order.completed',
    'order.refunded',
    'customer.created',
    'license.activated'
  ],
  secret: process.env.WEBHOOK_SECRET,
  active: true
});

console.log(`Webhook ID: ${webhook.id}`);
console.log(`Signature Header: X-CV-Signature`);
```

### Verify Webhook Signature

```javascript
const express = require('express');
const crypto = require('crypto');

app.post('/webhooks/edd', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-cv-signature'];
  const payload = req.body;
  
  const expectedSignature = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(payload)
    .digest('hex');
  
  if (signature !== expectedSignature) {
    return res.status(401).send('Invalid signature');
  }
  
  const event = JSON.parse(payload);
  
  switch (event.type) {
    case 'order.completed':
      console.log(`Order ${event.data.id} completed`);
      // Process fulfillment
      break;
    case 'order.refunded':
      console.log(`Order ${event.data.id} refunded`);
      // Handle refund logic
      break;
  }
  
  res.status(200).send('OK');
});
```

## Common Patterns

### Automated Inventory Sync

```javascript
// Sync products every hour
const cron = require('node-cron');

cron.schedule('0 * * * *', async () => {
  console.log('Starting inventory sync...');
  
  const lastSync = await redis.get('last_product_sync');
  const products = await client.products.listDelta({
    modifiedSince: lastSync || new Date(0).toISOString()
  });
  
  // Update local database
  for (const product of products.updated) {
    await db.products.upsert({
      id: product.id,
      title: product.title,
      price: product.price,
      inventory: product.inventory,
      updated_at: product.modified
    });
  }
  
  // Remove deleted products
  for (const deletedId of products.deleted) {
    await db.products.delete(deletedId);
  }
  
  await redis.set('last_product_sync', new Date().toISOString());
  console.log(`Synced ${products.updated.length} products`);
});
```

### Customer LTV Calculation

```javascript
async function calculateCustomerLTV(customerId) {
  const customer = await client.customers.get(customerId, {
    includeOrders: true
  });
  
  const orders = customer.orders.filter(o => o.status === 'completed');
  const totalRevenue = orders.reduce((sum, o) => sum + parseFloat(o.total), 0);
  const avgOrderValue = totalRevenue / orders.length;
  const purchaseFrequency = orders.length / customer.accountAgeDays * 365;
  
  // Predict future value based on historical behavior
  const expectedLifespanYears = 3;
  const predictedLTV = avgOrderValue * purchaseFrequency * expectedLifespanYears;
  
  return {
    historicalValue: totalRevenue,
    averageOrderValue: avgOrderValue,
    purchaseFrequency: purchaseFrequency,
    predictedLTV: predictedLTV,
    segment: predictedLTV > 1000 ? 'VIP' : predictedLTV > 500 ? 'Gold' : 'Standard'
  };
}
```

### Fraud Detection Middleware

```javascript
const fraudScorer = {
  name: 'fraudDetection',
  handler: async (request, next) => {
    if (request.type !== 'order.create') {
      return next(request);
    }
    
    let riskScore = 0;
    
    // High-value order
    if (request.data.total > 500) riskScore += 20;
    
    // Multiple high-value items
    if (request.data.lineItems.length > 5) riskScore += 15;
    
    // New customer
    const customer = await client.customers.get(request.data.customerId);
    if (customer.orderCount === 0) riskScore += 25;
    
    // Mismatched billing/shipping
    if (request.data.billingCountry !== request.data.shippingCountry) {
      riskScore += 30;
    }
    
    if (riskScore > 60) {
      request.data.status = 'on-hold';
      request.data.metadata.fraudScore = riskScore;
      console.warn(`High fraud risk detected: ${riskScore}`);
    }
    
    return next(request);
  }
};

client.addMiddleware(fraudScorer);
```

## Error Handling

### Retry Logic with Exponential Backoff

```javascript
async function retryableRequest(fn, maxRetries = 3) {
  let lastError;
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      if (error.code === 'RATE_LIMIT_EXCEEDED') {
        const waitTime = Math.pow(2, attempt) * 1000;
        console.log(`Rate limited, waiting ${waitTime}ms before retry...`);
        await new Promise(resolve => setTimeout(resolve, waitTime));
        continue;
      }
      
      // Don't retry client errors
      if (error.statusCode >= 400 && error.statusCode < 500) {
        throw error;
      }
      
      if (attempt < maxRetries - 1) {
        await new Promise(resolve => setTimeout(resolve, 1000 * (attempt + 1)));
      }
    }
  }
  
  throw lastError;
}

// Usage
const orders = await retryableRequest(() => 
  client.orders.list({ perPage: 100 })
);
```

### Comprehensive Error Handling

```javascript
try {
  const order = await client.orders.get(99999);
} catch (error) {
  if (error.code === 'NOT_FOUND') {
    console.error('Order does not exist');
  } else if (error.code === 'UNAUTHORIZED') {
    console.error('Invalid API credentials');
  } else if (error.code === 'RATE_LIMIT_EXCEEDED') {
    console.error('Rate limit exceeded, retry after:', error.retryAfter);
  } else if (error.code === 'HMAC_VERIFICATION_FAILED') {
    console.error('Data integrity check failed - possible tampering');
  } else {
    console.error('Unknown error:', error.message);
  }
}
```

## Troubleshooting

### Connection Issues

**Problem:** `ECONNREFUSED` or timeout errors

**Solution:**
```javascript
// Verify API URL is accessible
const client = new CommerceVaultClient({
  apiUrl: process.env.EDD_API_URL,
  timeout: 60000, // Increase timeout to 60s
  retryConfig: {
    maxRetries: 3,
    retryDelay: 2000
  }
});

// Test connection
try {
  const health = await client.health.check();
  console.log('API Status:', health.status);
} catch (error) {
  console.error('Cannot connect to API:', error.message);
}
```

### Authentication Failures

**Problem:** `401 Unauthorized` responses

**Solution:**
```bash
# Verify credentials are correct
curl -u "$EDD_CONSUMER_KEY:$EDD_CONSUMER_SECRET" \
  "$EDD_API_URL/wp-json/edd/v2/products?per_page=1"

# Check WordPress REST API is enabled
curl "$EDD_API_URL/wp-json/"

# Regenerate API keys in WordPress admin if needed
# WP Admin > Downloads > Settings > API > Generate New Keys
```

### Cache Invalidation

**Problem:** Stale data being returned

**Solution:**
```javascript
// Force bypass cache for specific request
const freshOrders = await client.orders.list({ 
  perPage: 50,
  _cache: false 
});

// Clear entire cache
await client.cache.clear();

// Clear specific cache pattern
await client.cache.clearPattern('orders:*');

// Set shorter TTL for frequently changing data
const products = await client.products.list({
  _cacheTTL: 300 // 5 minutes
});
```

### HMAC Verification Failures

**Problem:** `HMAC_VERIFICATION_FAILED` errors

**Solution:**
```javascript
// Ensure HMAC secret matches on both sides
const client = new CommerceVaultClient({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  hmacSecret: process.env.HMAC_SECRET_KEY, // Must match server config
  verifyHmac: true
});

// Temporarily disable HMAC for debugging (NOT for production)
client.config.verifyHmac = false;

// Re-enable after confirming secrets match
client.config.verifyHmac = true;
```

### Rate Limiting

**Problem:** `429 Too Many Requests` errors

**Solution:**
```javascript
// Enable adaptive rate limiting
const client = new CommerceVaultClient({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  rateLimiting: {
    strategy: 'adaptive',
    maxRequests: 100,
    windowSeconds: 60,
    respectServerLimits: true // Obey X-RateLimit headers
  }
});

// Check rate limit status
const limits = await client.rateLimit.status();
console.log(`Remaining: ${limits.remaining}/${limits.limit}`);
console.log(`Resets at: ${limits.resetTime}`);
```

---

**Official Resources:**
- GitHub: https://github.com/dhapat3927/mcp-edd-analytics-vantage
- Documentation: https://dhapat3927.github.io/mcp-edd-analytics-vantage/
- Issue Tracker: https://github.com/dhapat3927/mcp-edd-analytics-vantage/issues
