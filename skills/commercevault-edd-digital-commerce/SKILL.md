---
name: commercevault-edd-digital-commerce
description: Easy Digital Downloads MCP server for sales analytics, product management, and customer data orchestration
triggers:
  - connect to Easy Digital Downloads store
  - query EDD sales data
  - fetch digital product analytics
  - manage EDD customer records
  - retrieve EDD order history
  - integrate with WordPress commerce API
  - analyze EDD revenue metrics
  - sync digital product inventory
---

# CommerceVault EDD Digital Commerce Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

CommerceVault (mcp-edd-analytics-vantage) is a Model Context Protocol (MCP) server that provides AI agents with structured access to Easy Digital Downloads (EDD) e-commerce data. It exposes sales analytics, product catalogs, customer profiles, license management, and order lifecycle operations through a standardized API layer.

This middleware orchestrator sits between your application and WordPress/EDD installations, providing:
- **Product catalog synchronization** with delta updates
- **Order lifecycle management** from creation to fulfillment
- **Customer segmentation** and LTV analytics
- **Revenue reporting** with cohort analysis
- **License key management** for digital products
- **Webhook relay** for real-time event processing

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- WordPress installation with Easy Digital Downloads plugin
- EDD REST API credentials (API key and token)

### Setup

1. **Clone the repository:**

```bash
git clone https://github.com/dhapat3927/mcp-edd-analytics-vantage.git
cd mcp-edd-analytics-vantage
```

2. **Install dependencies:**

```bash
# For Node.js
npm install

# For Python
pip install -r requirements.txt
```

3. **Configure environment variables:**

Create a `.env` file:

```bash
EDD_API_URL=https://your-store.com/wp-json/edd/v2
EDD_API_KEY=your_api_key_here
EDD_API_TOKEN=your_api_token_here
CACHE_BACKEND=redis
REDIS_URL=redis://localhost:6379
RATE_LIMIT_PER_MINUTE=60
ENABLE_HMAC_VERIFICATION=true
HMAC_SECRET_KEY=your_secret_key_here
LOG_LEVEL=info
```

4. **Start the MCP server:**

```bash
# Node.js
npm start

# Python
python server.py
```

## Core API Operations

### Product Catalog Management

**Fetch all products:**

```javascript
const { MCPClient } = require('commercevault-client');

const client = new MCPClient({
  apiUrl: process.env.EDD_API_URL,
  apiKey: process.env.EDD_API_KEY,
  apiToken: process.env.EDD_API_TOKEN
});

async function getProducts() {
  const products = await client.products.list({
    page: 1,
    per_page: 50,
    status: 'publish',
    orderby: 'date',
    order: 'desc'
  });
  
  return products.data;
}
```

**Get product by ID with pricing tiers:**

```javascript
async function getProductDetails(productId) {
  const product = await client.products.get(productId, {
    include_variations: true,
    include_pricing: true
  });
  
  console.log(`Product: ${product.name}`);
  console.log(`Price: ${product.price}`);
  console.log(`Variations: ${product.variations.length}`);
  
  return product;
}
```

**Delta sync for changed products:**

```javascript
async function syncProductChanges(lastSyncTimestamp) {
  const changedProducts = await client.products.list({
    modified_after: lastSyncTimestamp,
    per_page: 100
  });
  
  for (const product of changedProducts.data) {
    await updateLocalCache(product);
  }
  
  return changedProducts.data.length;
}
```

### Order Management

**Fetch orders with filters:**

```javascript
async function getRecentOrders() {
  const orders = await client.orders.list({
    status: ['completed', 'processing'],
    date_created_after: '2026-01-01',
    per_page: 100,
    include_customer: true
  });
  
  return orders.data.map(order => ({
    id: order.id,
    total: order.total,
    customer: order.customer_email,
    status: order.status,
    items: order.line_items.length
  }));
}
```

**Create new order programmatically:**

```javascript
async function createOrder(orderData) {
  const newOrder = await client.orders.create({
    customer_email: orderData.email,
    customer_first_name: orderData.firstName,
    customer_last_name: orderData.lastName,
    line_items: orderData.items.map(item => ({
      product_id: item.productId,
      quantity: item.quantity,
      price: item.price
    })),
    payment_method: orderData.paymentMethod,
    status: 'pending'
  });
  
  return newOrder;
}
```

**Update order status:**

```javascript
async function fulfillOrder(orderId) {
  const updated = await client.orders.update(orderId, {
    status: 'completed',
    fulfillment_date: new Date().toISOString(),
    notes: 'Order fulfilled via API'
  });
  
  // Trigger fulfillment webhook
  await client.webhooks.trigger('order.completed', {
    order_id: orderId,
    timestamp: Date.now()
  });
  
  return updated;
}
```

### Customer Analytics

**Get customer lifetime value:**

```javascript
async function getCustomerLTV(customerId) {
  const customer = await client.customers.get(customerId, {
    include_orders: true,
    include_subscriptions: true
  });
  
  const ltv = customer.orders.reduce((sum, order) => {
    return sum + parseFloat(order.total);
  }, 0);
  
  return {
    customerId: customer.id,
    email: customer.email,
    totalOrders: customer.orders.length,
    lifetimeValue: ltv,
    averageOrderValue: ltv / customer.orders.length,
    firstPurchase: customer.orders[0]?.date_created,
    lastPurchase: customer.orders[customer.orders.length - 1]?.date_created
  };
}
```

**Segment customers by purchase frequency:**

```javascript
async function segmentCustomers() {
  const allCustomers = await client.customers.list({
    per_page: 1000,
    include_orders: true
  });
  
  const segments = {
    vip: [],      // 10+ orders
    regular: [],  // 3-9 orders
    new: []       // 1-2 orders
  };
  
  allCustomers.data.forEach(customer => {
    const orderCount = customer.order_count;
    if (orderCount >= 10) segments.vip.push(customer);
    else if (orderCount >= 3) segments.regular.push(customer);
    else segments.new.push(customer);
  });
  
  return segments;
}
```

### Analytics & Reporting

**Generate revenue report:**

```javascript
async function generateRevenueReport(startDate, endDate) {
  const analytics = await client.analytics.revenue({
    date_start: startDate,
    date_end: endDate,
    interval: 'day',
    include_taxes: true,
    include_fees: true
  });
  
  const report = {
    totalRevenue: analytics.total_gross,
    netRevenue: analytics.total_net,
    orderCount: analytics.order_count,
    averageOrderValue: analytics.total_gross / analytics.order_count,
    dailyBreakdown: analytics.intervals.map(day => ({
      date: day.date,
      revenue: day.gross,
      orders: day.count
    }))
  };
  
  return report;
}
```

**Product performance analysis:**

```javascript
async function analyzeProductPerformance(productId, days = 30) {
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - days);
  
  const performance = await client.analytics.products({
    product_id: productId,
    date_start: startDate.toISOString(),
    include_refunds: true
  });
  
  return {
    productId,
    totalSales: performance.total_sales,
    unitsSold: performance.units_sold,
    revenue: performance.revenue,
    refundRate: performance.refund_count / performance.units_sold,
    conversionRate: performance.conversion_rate
  };
}
```

### License Management

**Generate license keys:**

```javascript
async function generateLicense(orderId, productId) {
  const license = await client.licenses.create({
    order_id: orderId,
    product_id: productId,
    activation_limit: 3,
    expiration_date: null, // Lifetime license
    status: 'active'
  });
  
  return {
    key: license.key,
    activationLimit: license.activation_limit,
    activationsUsed: license.activations_used,
    expiresAt: license.expiration_date
  };
}
```

**Validate and activate license:**

```javascript
async function activateLicense(licenseKey, siteUrl) {
  try {
    const activation = await client.licenses.activate({
      license_key: licenseKey,
      site_url: siteUrl,
      product_id: process.env.PRODUCT_ID
    });
    
    return {
      success: true,
      activationId: activation.id,
      remainingActivations: activation.activations_remaining
    };
  } catch (error) {
    if (error.code === 'license_limit_reached') {
      return { success: false, error: 'Activation limit exceeded' };
    }
    throw error;
  }
}
```

## Configuration Patterns

### Rate Limiting Configuration

```javascript
const client = new MCPClient({
  apiUrl: process.env.EDD_API_URL,
  apiKey: process.env.EDD_API_KEY,
  apiToken: process.env.EDD_API_TOKEN,
  rateLimiting: {
    maxRequestsPerMinute: 60,
    adaptiveThrottling: true,
    queueOverflowRequests: true
  }
});
```

### Caching Strategy

```javascript
const client = new MCPClient({
  apiUrl: process.env.EDD_API_URL,
  apiKey: process.env.EDD_API_KEY,
  apiToken: process.env.EDD_API_TOKEN,
  cache: {
    enabled: true,
    backend: 'redis',
    redisUrl: process.env.REDIS_URL,
    ttl: {
      products: 3600,      // 1 hour
      customers: 1800,     // 30 minutes
      orders: 300,         // 5 minutes
      analytics: 600       // 10 minutes
    },
    invalidateOnWebhook: true
  }
});
```

### Webhook Configuration

```javascript
// Register webhook endpoints
await client.webhooks.register({
  event: 'order.completed',
  target_url: 'https://your-app.com/webhooks/order-completed',
  secret: process.env.WEBHOOK_SECRET
});

// Handle incoming webhooks
app.post('/webhooks/order-completed', async (req, res) => {
  const signature = req.headers['x-edd-signature'];
  const payload = req.body;
  
  // Verify HMAC signature
  const isValid = client.webhooks.verifySignature(
    payload,
    signature,
    process.env.WEBHOOK_SECRET
  );
  
  if (!isValid) {
    return res.status(401).send('Invalid signature');
  }
  
  // Process the order completion
  await processOrderCompletion(payload.order_id);
  
  res.status(200).send('OK');
});
```

## Python Usage Examples

### Basic Client Setup

```python
from commercevault import MCPClient
import os

client = MCPClient(
    api_url=os.getenv('EDD_API_URL'),
    api_key=os.getenv('EDD_API_KEY'),
    api_token=os.getenv('EDD_API_TOKEN')
)
```

### Async Operations

```python
import asyncio
from commercevault import AsyncMCPClient

async def fetch_sales_data():
    async with AsyncMCPClient(
        api_url=os.getenv('EDD_API_URL'),
        api_key=os.getenv('EDD_API_KEY'),
        api_token=os.getenv('EDD_API_TOKEN')
    ) as client:
        orders = await client.orders.list(
            status=['completed'],
            per_page=100
        )
        
        total_revenue = sum(float(order['total']) for order in orders['data'])
        return total_revenue

revenue = asyncio.run(fetch_sales_data())
print(f"Total revenue: ${revenue:.2f}")
```

### Batch Operations

```python
def bulk_update_product_prices(price_updates):
    """
    price_updates: dict of {product_id: new_price}
    """
    results = []
    
    for product_id, new_price in price_updates.items():
        try:
            updated = client.products.update(product_id, {
                'price': new_price,
                'updated_by': 'bulk_price_update'
            })
            results.append({'id': product_id, 'success': True})
        except Exception as e:
            results.append({'id': product_id, 'success': False, 'error': str(e)})
    
    return results
```

## Common Patterns

### Pagination Handler

```javascript
async function fetchAllPages(endpoint, params = {}) {
  let allData = [];
  let page = 1;
  let hasMore = true;
  
  while (hasMore) {
    const response = await client[endpoint].list({
      ...params,
      page,
      per_page: 100
    });
    
    allData = allData.concat(response.data);
    hasMore = response.pagination.has_more;
    page++;
    
    // Rate limiting pause
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  return allData;
}

// Usage
const allProducts = await fetchAllPages('products', { status: 'publish' });
```

### Error Handling with Retry

```javascript
async function retryableRequest(requestFn, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await requestFn();
    } catch (error) {
      if (error.status === 429) {
        // Rate limit hit, exponential backoff
        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      
      if (attempt === maxRetries) {
        throw error;
      }
      
      // Retry on network errors
      if (error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT') {
        await new Promise(resolve => setTimeout(resolve, 2000));
        continue;
      }
      
      throw error;
    }
  }
}

// Usage
const orders = await retryableRequest(() => 
  client.orders.list({ status: 'completed' })
);
```

### Event-Driven Architecture

```javascript
const EventEmitter = require('events');

class CommerceVaultEvents extends EventEmitter {
  constructor(client) {
    super();
    this.client = client;
  }
  
  async monitorOrders(pollInterval = 60000) {
    let lastChecked = new Date();
    
    setInterval(async () => {
      const newOrders = await this.client.orders.list({
        date_created_after: lastChecked.toISOString(),
        status: ['pending', 'processing']
      });
      
      newOrders.data.forEach(order => {
        this.emit('order:created', order);
      });
      
      lastChecked = new Date();
    }, pollInterval);
  }
}

// Usage
const events = new CommerceVaultEvents(client);

events.on('order:created', async (order) => {
  console.log(`New order: ${order.id}`);
  await sendOrderNotification(order);
});

events.monitorOrders(30000); // Check every 30 seconds
```

## Troubleshooting

### Authentication Failures

**Issue:** 401 Unauthorized errors

**Solution:**
```javascript
// Verify credentials are correctly set
const testAuth = async () => {
  try {
    await client.auth.verify();
    console.log('Authentication successful');
  } catch (error) {
    console.error('Auth failed:', error.message);
    // Check EDD API key/token in WordPress admin
  }
};
```

### Rate Limit Exceeded

**Issue:** 429 Too Many Requests

**Solution:**
```javascript
// Enable adaptive throttling
const client = new MCPClient({
  apiUrl: process.env.EDD_API_URL,
  apiKey: process.env.EDD_API_KEY,
  apiToken: process.env.EDD_API_TOKEN,
  rateLimiting: {
    adaptiveThrottling: true,
    queueOverflowRequests: true,
    maxQueueSize: 1000
  }
});
```

### Cache Invalidation Issues

**Issue:** Stale data returned from cache

**Solution:**
```javascript
// Force cache bypass for specific requests
const freshData = await client.products.get(productId, {
  cache: false
});

// Or invalidate specific cache keys
await client.cache.invalidate('products', productId);

// Or flush entire cache
await client.cache.flush();
```

### Webhook Signature Verification Failed

**Issue:** Webhook payloads rejected due to signature mismatch

**Solution:**
```javascript
// Ensure raw body is used for signature verification
app.use('/webhooks', express.raw({ type: 'application/json' }));

app.post('/webhooks/order-completed', (req, res) => {
  const signature = req.headers['x-edd-signature'];
  const rawBody = req.body.toString('utf8');
  
  const isValid = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET)
    .update(rawBody)
    .digest('hex') === signature;
  
  if (!isValid) {
    return res.status(401).send('Invalid signature');
  }
  
  const payload = JSON.parse(rawBody);
  // Process webhook...
});
```

### Connection Timeouts

**Issue:** Requests timing out on slow networks

**Solution:**
```javascript
const client = new MCPClient({
  apiUrl: process.env.EDD_API_URL,
  apiKey: process.env.EDD_API_KEY,
  apiToken: process.env.EDD_API_TOKEN,
  timeout: 30000, // 30 seconds
  retries: 3,
  retryDelay: 2000
});
```

## Advanced Usage

### Custom Middleware Pipeline

```javascript
// Add custom middleware for request enrichment
client.use('beforeRequest', async (request) => {
  request.headers['X-Client-Version'] = '1.4.0';
  request.headers['X-Request-ID'] = generateUUID();
  return request;
});

client.use('afterResponse', async (response) => {
  // Log all responses
  await logToAnalytics({
    endpoint: response.config.url,
    status: response.status,
    duration: response.duration
  });
  return response;
});
```

### Multi-Store Management

```javascript
const stores = {
  us: new MCPClient({ 
    apiUrl: process.env.EDD_US_API_URL,
    apiKey: process.env.EDD_US_API_KEY,
    apiToken: process.env.EDD_US_API_TOKEN
  }),
  eu: new MCPClient({ 
    apiUrl: process.env.EDD_EU_API_URL,
    apiKey: process.env.EDD_EU_API_KEY,
    apiToken: process.env.EDD_EU_API_TOKEN
  })
};

async function getGlobalRevenue() {
  const [usRevenue, euRevenue] = await Promise.all([
    stores.us.analytics.revenue({ date_start: '2026-01-01' }),
    stores.eu.analytics.revenue({ date_start: '2026-01-01' })
  ]);
  
  return {
    us: usRevenue.total_gross,
    eu: euRevenue.total_gross,
    total: usRevenue.total_gross + euRevenue.total_gross
  };
}
```
