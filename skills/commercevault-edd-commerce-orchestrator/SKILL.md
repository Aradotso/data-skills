---
name: commercevault-edd-commerce-orchestrator
description: EDD MCP Server for Easy Digital Downloads API integration with sales analytics, product management, and order processing
triggers:
  - "integrate Easy Digital Downloads with my application"
  - "query EDD sales data and analytics"
  - "manage digital products through EDD API"
  - "set up CommerceVault middleware for e-commerce"
  - "connect to Easy Digital Downloads REST API"
  - "fetch order and customer data from EDD"
  - "implement digital commerce orchestration layer"
  - "sync product catalog with EDD"
---

# CommerceVault EDD Commerce Orchestrator

> Skill by [ara.so](https://ara.so) — Data Skills collection

CommerceVault is a middleware orchestration layer for Easy Digital Downloads (EDD) and other e-commerce platforms. It provides a unified API interface for sales analytics, product management, order lifecycle tracking, and customer data synchronization. Built on MCP (Model Context Protocol) architecture, it abstracts REST API complexity into clean, testable endpoints with built-in authentication, rate limiting, caching, and audit trails.

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- Easy Digital Downloads WordPress installation with REST API enabled
- API credentials (Consumer Key and Consumer Secret from EDD settings)

### Install via npm

```bash
npm install @commercevault/edd-connector
```

### Install via pip

```bash
pip install commercevault-edd
```

### Environment Configuration

Create a `.env` file:

```env
EDD_API_URL=https://your-site.com/edd-api/v2
EDD_CONSUMER_KEY=ck_your_consumer_key_here
EDD_CONSUMER_SECRET=cs_your_consumer_secret_here
EDD_API_TIMEOUT=30000
CACHE_TTL=300
LOG_LEVEL=info
```

## Core API Operations

### Product Management

#### Fetch All Products

```javascript
const { EDDClient } = require('@commercevault/edd-connector');

const client = new EDDClient({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET
});

// Get all products with pagination
async function getAllProducts() {
  const products = await client.products.list({
    page: 1,
    per_page: 50,
    status: 'publish'
  });
  
  return products;
}

// Get single product by ID
async function getProduct(productId) {
  const product = await client.products.get(productId);
  
  console.log(`Product: ${product.info.title}`);
  console.log(`Price: ${product.pricing.amount}`);
  console.log(`Downloads: ${product.stats.sales}`);
  
  return product;
}
```

#### Create/Update Products

```javascript
async function createProduct(productData) {
  const newProduct = await client.products.create({
    title: productData.title,
    content: productData.description,
    status: 'publish',
    pricing: {
      amount: productData.price,
      currency: 'USD'
    },
    files: productData.downloadFiles
  });
  
  return newProduct;
}

async function updateProductPrice(productId, newPrice) {
  const updated = await client.products.update(productId, {
    pricing: {
      amount: newPrice
    }
  });
  
  return updated;
}
```

### Order & Sales Analytics

#### Retrieve Orders

```javascript
async function getRecentOrders(days = 7) {
  const since = new Date();
  since.setDate(since.getDate() - days);
  
  const orders = await client.orders.list({
    date_query: {
      after: since.toISOString(),
      inclusive: true
    },
    order: 'desc',
    orderby: 'date'
  });
  
  return orders;
}

async function getOrderById(orderId) {
  const order = await client.orders.get(orderId);
  
  return {
    id: order.ID,
    status: order.status,
    total: order.total,
    customer: order.user_info.email,
    products: order.cart_details,
    date: order.date
  };
}
```

#### Sales Analytics

```javascript
async function getSalesStats(startDate, endDate) {
  const stats = await client.analytics.sales({
    start_date: startDate,
    end_date: endDate
  });
  
  return {
    totalRevenue: stats.total_revenue,
    totalOrders: stats.total_orders,
    averageOrderValue: stats.average_order_value,
    topProducts: stats.top_products
  };
}

async function getRevenuByProduct(productId, timeRange = 'month') {
  const revenue = await client.analytics.productRevenue(productId, {
    range: timeRange
  });
  
  return revenue;
}
```

### Customer Management

#### Fetch Customer Data

```javascript
async function getCustomerProfile(customerId) {
  const customer = await client.customers.get(customerId);
  
  return {
    id: customer.id,
    email: customer.email,
    name: customer.name,
    purchaseCount: customer.purchase_count,
    lifetimeValue: customer.purchase_value,
    dateCreated: customer.date_created
  };
}

async function getCustomerOrders(customerId) {
  const orders = await client.customers.orders(customerId, {
    per_page: 100
  });
  
  return orders;
}
```

#### Customer Segmentation

```javascript
async function getHighValueCustomers(minValue = 1000) {
  const customers = await client.customers.list({
    orderby: 'purchase_value',
    order: 'desc'
  });
  
  return customers.filter(c => c.purchase_value >= minValue);
}
```

### License & Entitlement Management

```javascript
async function generateLicenseKey(orderId, productId) {
  const license = await client.licenses.create({
    order_id: orderId,
    product_id: productId,
    activation_limit: 3,
    expiration: '1 year'
  });
  
  return license.key;
}

async function validateLicense(licenseKey) {
  const validation = await client.licenses.validate(licenseKey);
  
  return {
    valid: validation.success,
    activationsLeft: validation.activations_left,
    expiresAt: validation.expires
  };
}

async function deactivateLicense(licenseKey, siteUrl) {
  await client.licenses.deactivate({
    license: licenseKey,
    url: siteUrl
  });
}
```

## Python Implementation

```python
from commercevault import EDDClient
import os
from datetime import datetime, timedelta

# Initialize client
client = EDDClient(
    api_url=os.getenv('EDD_API_URL'),
    consumer_key=os.getenv('EDD_CONSUMER_KEY'),
    consumer_secret=os.getenv('EDD_CONSUMER_SECRET')
)

# Get products
def fetch_products_with_filters():
    products = client.products.list(
        status='publish',
        per_page=50,
        category='software'
    )
    
    for product in products:
        print(f"{product['info']['title']}: ${product['pricing']['amount']}")
    
    return products

# Analytics query
def get_monthly_revenue():
    today = datetime.now()
    start_of_month = today.replace(day=1)
    
    stats = client.analytics.sales(
        start_date=start_of_month.isoformat(),
        end_date=today.isoformat()
    )
    
    return stats['total_revenue']

# Customer lifetime value
def calculate_ltv_by_cohort(signup_month):
    customers = client.customers.list(
        date_query={'month': signup_month}
    )
    
    total_value = sum(c['purchase_value'] for c in customers)
    avg_ltv = total_value / len(customers) if customers else 0
    
    return avg_ltv

# Order processing with webhooks
def process_order_webhook(order_data):
    order_id = order_data['id']
    order = client.orders.get(order_id)
    
    # Generate licenses for digital products
    for item in order['cart_details']:
        if item['is_digital']:
            license = client.licenses.create({
                'order_id': order_id,
                'product_id': item['id'],
                'activation_limit': 1
            })
            print(f"License generated: {license['key']}")
    
    return order
```

## Configuration & Middleware

### Rate Limiting

```javascript
const client = new EDDClient({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  rateLimit: {
    maxRequests: 100,
    perWindow: 60000, // 1 minute
    strategy: 'adaptive'
  }
});
```

### Caching Layer

```javascript
const client = new EDDClient({
  // ... auth config
  cache: {
    enabled: true,
    ttl: 300, // 5 minutes
    redis: {
      host: process.env.REDIS_HOST || 'localhost',
      port: process.env.REDIS_PORT || 6379
    }
  }
});

// Bypass cache for specific request
const freshData = await client.products.get(123, { noCache: true });
```

### Error Handling & Retry Logic

```javascript
const client = new EDDClient({
  // ... auth config
  retry: {
    attempts: 3,
    backoff: 'exponential',
    retryableErrors: [429, 500, 502, 503, 504]
  }
});

// Manual error handling
try {
  const orders = await client.orders.list();
} catch (error) {
  if (error.code === 'RATE_LIMIT_EXCEEDED') {
    console.error('Rate limit hit, retry after:', error.retryAfter);
  } else if (error.code === 'AUTH_FAILED') {
    console.error('Check API credentials');
  } else {
    console.error('Unexpected error:', error.message);
  }
}
```

## Common Patterns

### Daily Sales Report

```javascript
async function generateDailySalesReport() {
  const today = new Date();
  const yesterday = new Date(today);
  yesterday.setDate(yesterday.getDate() - 1);
  
  const orders = await client.orders.list({
    date_query: {
      after: yesterday.toISOString(),
      before: today.toISOString()
    }
  });
  
  const stats = {
    totalOrders: orders.length,
    totalRevenue: orders.reduce((sum, o) => sum + parseFloat(o.total), 0),
    productBreakdown: {}
  };
  
  orders.forEach(order => {
    order.cart_details.forEach(item => {
      if (!stats.productBreakdown[item.name]) {
        stats.productBreakdown[item.name] = { count: 0, revenue: 0 };
      }
      stats.productBreakdown[item.name].count += item.quantity;
      stats.productBreakdown[item.name].revenue += parseFloat(item.price);
    });
  });
  
  return stats;
}
```

### Inventory Sync

```javascript
async function syncInventoryToCRM(crmClient) {
  const products = await client.products.list({ per_page: 100 });
  
  for (const product of products) {
    await crmClient.updateProduct({
      externalId: product.info.id,
      name: product.info.title,
      price: product.pricing.amount,
      stock: product.stats.sales,
      lastModified: product.info.modified
    });
  }
  
  console.log(`Synced ${products.length} products to CRM`);
}
```

### Webhook Event Processing

```javascript
const express = require('express');
const app = express();

app.post('/webhooks/edd/order-completed', async (req, res) => {
  const signature = req.headers['x-edd-signature'];
  
  // Verify webhook signature
  if (!client.webhooks.verify(req.body, signature)) {
    return res.status(401).send('Invalid signature');
  }
  
  const order = req.body;
  
  // Process order
  await processOrder(order.id);
  
  res.status(200).send('OK');
});

async function processOrder(orderId) {
  const order = await client.orders.get(orderId);
  
  // Send fulfillment email
  // Update inventory
  // Trigger analytics event
}
```

## Troubleshooting

### Authentication Errors

**Problem**: `401 Unauthorized` errors

**Solution**: Verify API credentials in WordPress admin under Downloads → Settings → API Keys. Ensure the key has appropriate permissions.

```javascript
// Test authentication
async function testAuth() {
  try {
    const test = await client.auth.test();
    console.log('Auth successful:', test);
  } catch (error) {
    console.error('Auth failed:', error.message);
  }
}
```

### Rate Limiting

**Problem**: `429 Too Many Requests`

**Solution**: Enable adaptive rate limiting or implement exponential backoff.

```javascript
const client = new EDDClient({
  // ... auth
  rateLimit: {
    strategy: 'adaptive',
    respectRetryAfter: true
  }
});
```

### Stale Cache Data

**Problem**: Getting outdated product/order information

**Solution**: Reduce TTL or invalidate cache on specific events.

```javascript
// Invalidate cache for specific product
await client.cache.invalidate('products', productId);

// Clear all cache
await client.cache.flush();
```

### Missing Orders

**Problem**: Orders not appearing in API results

**Solution**: Check date range, status filters, and pagination.

```javascript
// Get ALL orders including drafts/pending
const allOrders = await client.orders.list({
  status: 'any',
  per_page: 100,
  page: 1
});
```

### Performance Issues

**Problem**: Slow API responses for large datasets

**Solution**: Use pagination, field filtering, and caching.

```javascript
// Only fetch needed fields
const products = await client.products.list({
  per_page: 50,
  fields: ['id', 'title', 'pricing', 'sales']
});

// Use cursor-based pagination for large datasets
let cursor = null;
do {
  const batch = await client.orders.listCursor({ cursor, limit: 100 });
  await processBatch(batch.data);
  cursor = batch.next_cursor;
} while (cursor);
```

## Advanced Usage

### Custom Adapter for Other Platforms

```javascript
const { BaseAdapter } = require('@commercevault/core');

class WooCommerceAdapter extends BaseAdapter {
  async listProducts(params) {
    // Transform params to WooCommerce format
    const response = await this.request('/products', params);
    // Transform response to UCL standard
    return this.transformToUCL(response);
  }
}

const client = new CommerceVault({
  adapter: new WooCommerceAdapter(/* config */)
});
```

### Batch Operations

```javascript
// Bulk update product prices
async function bulkUpdatePrices(priceUpdates) {
  const operations = priceUpdates.map(update => 
    client.products.update(update.id, { 
      pricing: { amount: update.newPrice } 
    })
  );
  
  const results = await Promise.allSettled(operations);
  
  const succeeded = results.filter(r => r.status === 'fulfilled').length;
  console.log(`Updated ${succeeded}/${priceUpdates.length} products`);
}
```

This skill provides comprehensive coverage of CommerceVault's EDD integration capabilities for building robust digital commerce automation and analytics solutions.
