---
name: commercevault-edd-digital-commerce
description: MCP server for Easy Digital Downloads API integration for sales analytics, products, orders, and customer management
triggers:
  - "integrate EDD with analytics platform"
  - "query Easy Digital Downloads sales data"
  - "fetch EDD product catalog programmatically"
  - "analyze digital commerce revenue metrics"
  - "connect to WordPress EDD REST API"
  - "retrieve customer purchase history from EDD"
  - "sync digital product inventory with EDD"
  - "build EDD sales dashboard"
---

# CommerceVault EDD Digital Commerce Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection

## Overview

CommerceVault (mcp-edd-analytics-vantage) is an MCP (Model Context Protocol) server that provides structured access to Easy Digital Downloads (EDD) e-commerce data. It acts as a middleware layer for querying sales analytics, product catalogs, order history, customer data, and license management from WordPress EDD installations.

The server implements a hexagonal architecture with pluggable adapters, cryptographic payload verification, adaptive rate limiting, and built-in analytics aggregation. It's designed for SaaS platforms, subscription businesses, digital storefronts, and analytics applications that need reliable, secure access to EDD commerce data.

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- Access to a WordPress site running Easy Digital Downloads
- EDD REST API credentials (Consumer Key and Secret)

### MCP Server Setup

Install as an MCP server for Claude Desktop or other MCP-compatible clients:

```json
{
  "mcpServers": {
    "commercevault-edd": {
      "command": "node",
      "args": ["/path/to/mcp-edd-analytics-vantage/server.js"],
      "env": {
        "EDD_API_URL": "https://your-site.com/wp-json/edd/v2",
        "EDD_CONSUMER_KEY": "ck_xxxxxxxxxxxx",
        "EDD_CONSUMER_SECRET": "cs_xxxxxxxxxxxx",
        "VAULT_SECRET_KEY": "your-hmac-secret-key"
      }
    }
  }
}
```

### Standalone Installation

```bash
# Clone repository
git clone https://github.com/dhapat3927/mcp-edd-analytics-vantage.git
cd mcp-edd-analytics-vantage

# Install dependencies (Node.js)
npm install

# Or for Python
pip install -r requirements.txt
```

## Configuration

### Environment Variables

```bash
# Required: EDD API endpoint
export EDD_API_URL="https://your-site.com/wp-json/edd/v2"

# Required: Authentication credentials
export EDD_CONSUMER_KEY="ck_your_consumer_key"
export EDD_CONSUMER_SECRET="cs_your_consumer_secret"

# Optional: HMAC signature verification
export VAULT_SECRET_KEY="your-hmac-secret-for-payload-integrity"

# Optional: Rate limiting (requests per minute)
export RATE_LIMIT_RPM="60"

# Optional: Cache TTL (seconds)
export CACHE_TTL="300"

# Optional: Data residency compliance
export DATA_REGION="US"
export AUDIT_LOG_PATH="/var/log/commercevault/audit.log"
```

### Configuration File

Create `config/vault.json`:

```json
{
  "endpoints": {
    "edd": {
      "baseUrl": "https://your-site.com/wp-json/edd/v2",
      "timeout": 30000,
      "retryAttempts": 3
    }
  },
  "cache": {
    "enabled": true,
    "backend": "redis",
    "ttl": 300,
    "redis": {
      "host": "localhost",
      "port": 6379
    }
  },
  "rateLimit": {
    "requestsPerMinute": 60,
    "adaptive": true
  },
  "security": {
    "enableHMAC": true,
    "tlsVersion": "1.2",
    "certificatePinning": false
  }
}
```

## Core API Operations

### Products

#### Fetch Product Catalog

```javascript
// Node.js/JavaScript example
const CommerceVault = require('commercevault-edd');

const vault = new CommerceVault({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET
});

// Get all products with pagination
async function fetchProducts() {
  const products = await vault.products.list({
    page: 1,
    per_page: 50,
    status: 'publish',
    orderby: 'date',
    order: 'desc'
  });
  
  return products.data.map(product => ({
    id: product.id,
    name: product.name,
    price: product.price,
    downloads: product.sales_count,
    created: product.date_created
  }));
}

// Get single product with enriched metadata
async function getProductDetails(productId) {
  const product = await vault.products.get(productId);
  
  return {
    ...product.data,
    enriched: {
      averageRating: product.average_rating,
      totalReviews: product.rating_count,
      permalink: product.permalink
    }
  };
}
```

#### Delta Sync for Changed Products

```python
# Python example for incremental sync
from commercevault import EDDClient
import os
from datetime import datetime, timedelta

client = EDDClient(
    api_url=os.environ['EDD_API_URL'],
    consumer_key=os.environ['EDD_CONSUMER_KEY'],
    consumer_secret=os.environ['EDD_CONSUMER_SECRET']
)

# Fetch products modified in last 24 hours
def sync_updated_products(last_sync_time):
    delta = datetime.utcnow() - timedelta(hours=24)
    
    products = client.products.list(
        modified_after=delta.isoformat(),
        fields=['id', 'name', 'price', 'date_modified']
    )
    
    # Process only changed records
    for product in products['data']:
        if product['date_modified'] > last_sync_time:
            # Sync to your database
            update_local_product(product)
    
    return products['meta']['total']
```

### Orders

#### Retrieve Order History

```javascript
// Fetch orders with customer and payment details
async function getOrders(params = {}) {
  const orders = await vault.orders.list({
    status: params.status || 'complete',
    customer: params.customerId,
    date_created_after: params.startDate,
    date_created_before: params.endDate,
    per_page: params.limit || 100
  });
  
  return orders.data.map(order => ({
    orderId: order.id,
    total: order.total,
    currency: order.currency,
    status: order.status,
    customer: {
      id: order.customer_id,
      email: order.billing.email,
      name: `${order.billing.first_name} ${order.billing.last_name}`
    },
    items: order.line_items.map(item => ({
      productId: item.product_id,
      quantity: item.quantity,
      subtotal: item.subtotal
    })),
    createdAt: order.date_created,
    paymentMethod: order.payment_method
  }));
}
```

#### Order Lifecycle Management

```python
# Create, update, and refund orders
def process_order_lifecycle(order_id):
    # Update order status
    client.orders.update(order_id, {
        'status': 'processing',
        'customer_note': 'Order is being prepared for delivery'
    })
    
    # Add order note
    client.orders.add_note(order_id, {
        'note': 'License keys generated and sent to customer',
        'customer_note': False
    })
    
    # Process refund
    refund = client.orders.refund(order_id, {
        'amount': '49.99',
        'reason': 'Customer requested refund',
        'refund_items': [
            {'id': 123, 'quantity': 1}
        ]
    })
    
    return refund
```

### Customers

#### Customer Identity & Segmentation

```javascript
// Retrieve customer data with lifetime value metrics
async function getCustomerProfile(customerId) {
  const customer = await vault.customers.get(customerId);
  
  // Calculate LTV metrics
  const orders = await vault.orders.list({ customer: customerId });
  const ltv = orders.data.reduce((sum, order) => sum + parseFloat(order.total), 0);
  
  return {
    id: customer.data.id,
    email: customer.data.email,
    firstName: customer.data.first_name,
    lastName: customer.data.last_name,
    dateCreated: customer.data.date_created,
    metrics: {
      totalOrders: orders.meta.total,
      lifetimeValue: ltv,
      averageOrderValue: ltv / orders.meta.total,
      lastPurchase: orders.data[0]?.date_created
    }
  };
}

// Segment customers by purchase behavior
async function segmentCustomers() {
  const customers = await vault.customers.list({ per_page: 1000 });
  
  const segments = {
    whales: [], // LTV > $1000
    active: [],  // Purchase in last 30 days
    churned: []  // No purchase in 180+ days
  };
  
  for (const customer of customers.data) {
    const profile = await getCustomerProfile(customer.id);
    
    if (profile.metrics.lifetimeValue > 1000) {
      segments.whales.push(profile);
    }
    
    const daysSinceLastPurchase = 
      (Date.now() - new Date(profile.metrics.lastPurchase)) / (1000 * 60 * 60 * 24);
    
    if (daysSinceLastPurchase < 30) {
      segments.active.push(profile);
    } else if (daysSinceLastPurchase > 180) {
      segments.churned.push(profile);
    }
  }
  
  return segments;
}
```

### Analytics & Revenue Metrics

#### Sales Analytics Dashboard

```python
# Generate revenue analytics report
from datetime import datetime, timedelta
import statistics

def generate_sales_report(days=30):
    end_date = datetime.utcnow()
    start_date = end_date - timedelta(days=days)
    
    # Fetch orders in date range
    orders = client.orders.list(
        status='complete',
        date_created_after=start_date.isoformat(),
        date_created_before=end_date.isoformat(),
        per_page=1000
    )
    
    order_totals = [float(order['total']) for order in orders['data']]
    
    analytics = {
        'period': {
            'start': start_date.isoformat(),
            'end': end_date.isoformat(),
            'days': days
        },
        'revenue': {
            'total': sum(order_totals),
            'average_order_value': statistics.mean(order_totals) if order_totals else 0,
            'median_order_value': statistics.median(order_totals) if order_totals else 0,
            'order_count': len(order_totals)
        },
        'top_products': get_top_products(orders['data']),
        'geographic_breakdown': get_geographic_revenue(orders['data']),
        'payment_methods': get_payment_method_breakdown(orders['data'])
    }
    
    return analytics

def get_top_products(orders, limit=10):
    product_sales = {}
    
    for order in orders:
        for item in order['line_items']:
            product_id = item['product_id']
            if product_id not in product_sales:
                product_sales[product_id] = {
                    'name': item['name'],
                    'quantity': 0,
                    'revenue': 0.0
                }
            product_sales[product_id]['quantity'] += item['quantity']
            product_sales[product_id]['revenue'] += float(item['total'])
    
    # Sort by revenue
    sorted_products = sorted(
        product_sales.items(),
        key=lambda x: x[1]['revenue'],
        reverse=True
    )
    
    return sorted_products[:limit]
```

#### Cohort Analysis

```javascript
// Analyze customer cohorts by signup month
async function cohortAnalysis(months = 12) {
  const cohorts = {};
  const today = new Date();
  
  for (let i = 0; i < months; i++) {
    const cohortMonth = new Date(today.getFullYear(), today.getMonth() - i, 1);
    const nextMonth = new Date(cohortMonth.getFullYear(), cohortMonth.getMonth() + 1, 1);
    
    // Get customers who signed up in this month
    const customers = await vault.customers.list({
      date_created_after: cohortMonth.toISOString(),
      date_created_before: nextMonth.toISOString(),
      per_page: 1000
    });
    
    // Calculate retention and revenue for cohort
    const cohortMetrics = {
      signupMonth: cohortMonth.toISOString().substring(0, 7),
      customerCount: customers.data.length,
      firstMonthRevenue: 0,
      retainedCustomers: 0
    };
    
    for (const customer of customers.data) {
      const orders = await vault.orders.list({ customer: customer.id });
      
      // First month orders
      const firstMonthOrders = orders.data.filter(order => {
        const orderDate = new Date(order.date_created);
        return orderDate >= cohortMonth && orderDate < nextMonth;
      });
      
      cohortMetrics.firstMonthRevenue += firstMonthOrders.reduce(
        (sum, order) => sum + parseFloat(order.total), 0
      );
      
      // Check if customer made a purchase in subsequent months
      const laterOrders = orders.data.filter(order => {
        return new Date(order.date_created) >= nextMonth;
      });
      
      if (laterOrders.length > 0) {
        cohortMetrics.retainedCustomers++;
      }
    }
    
    cohortMetrics.retentionRate = 
      (cohortMetrics.retainedCustomers / cohortMetrics.customerCount) * 100;
    
    cohorts[cohortMetrics.signupMonth] = cohortMetrics;
  }
  
  return cohorts;
}
```

### License & Entitlement Management

```javascript
// Manage software licenses for digital products
async function manageLicense(customerId, productId) {
  // Generate license key
  const license = await vault.licenses.create({
    customer_id: customerId,
    product_id: productId,
    activation_limit: 3,
    expiration: '2027-12-31',
    status: 'active'
  });
  
  // Activate license
  const activation = await vault.licenses.activate(license.data.key, {
    site_url: 'https://customer-site.com',
    instance_id: 'uuid-here'
  });
  
  // Check license status
  const status = await vault.licenses.check(license.data.key);
  
  return {
    key: license.data.key,
    activationsRemaining: status.data.activations_left,
    expires: license.data.expiration,
    isValid: status.data.valid
  };
}

// Deactivate license
async function deactivateLicense(licenseKey, instanceId) {
  return await vault.licenses.deactivate(licenseKey, {
    instance_id: instanceId
  });
}
```

## Common Patterns

### Webhook Event Handling

```python
# Process EDD webhooks with payload verification
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)

@app.route('/webhooks/edd', methods=['POST'])
def handle_edd_webhook():
    # Verify HMAC signature
    signature = request.headers.get('X-EDD-Signature')
    payload = request.get_data()
    
    expected_signature = hmac.new(
        os.environ['VAULT_SECRET_KEY'].encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(signature, expected_signature):
        return {'error': 'Invalid signature'}, 401
    
    event = request.json
    
    # Route event to appropriate handler
    handlers = {
        'order.completed': handle_order_completed,
        'subscription.created': handle_subscription_created,
        'customer.updated': handle_customer_updated,
        'refund.processed': handle_refund_processed
    }
    
    handler = handlers.get(event['type'])
    if handler:
        handler(event['data'])
    
    return {'status': 'received'}, 200

def handle_order_completed(order_data):
    # Send fulfillment email, generate license keys, etc.
    print(f"Order {order_data['id']} completed: ${order_data['total']}")
```

### Batch Operations with Rate Limiting

```javascript
// Process large datasets without hitting rate limits
const pLimit = require('p-limit');

async function batchUpdateProducts(updates) {
  // Limit concurrent requests to respect rate limits
  const limit = pLimit(5);
  
  const tasks = updates.map(update => 
    limit(() => vault.products.update(update.id, update.data))
  );
  
  // Execute with backoff on rate limit errors
  const results = await Promise.allSettled(tasks);
  
  const succeeded = results.filter(r => r.status === 'fulfilled').length;
  const failed = results.filter(r => r.status === 'rejected');
  
  // Retry failed updates
  if (failed.length > 0) {
    console.log(`Retrying ${failed.length} failed updates after delay`);
    await new Promise(resolve => setTimeout(resolve, 60000)); // 1 minute delay
    
    const retryTasks = failed.map((result, index) => 
      limit(() => vault.products.update(updates[index].id, updates[index].data))
    );
    
    await Promise.allSettled(retryTasks);
  }
  
  return { succeeded, failed: failed.length };
}
```

### Caching Strategy

```javascript
// Implement smart caching with automatic invalidation
const Redis = require('redis');
const redisClient = Redis.createClient({
  url: process.env.REDIS_URL
});

await redisClient.connect();

async function getCachedProduct(productId) {
  const cacheKey = `product:${productId}`;
  
  // Try cache first
  const cached = await redisClient.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Fetch from API
  const product = await vault.products.get(productId);
  
  // Cache with TTL
  await redisClient.setEx(
    cacheKey,
    300, // 5 minutes
    JSON.stringify(product.data)
  );
  
  return product.data;
}

// Invalidate cache on webhook events
function invalidateProductCache(productId) {
  redisClient.del(`product:${productId}`);
}
```

## Troubleshooting

### Authentication Errors

```javascript
// Verify credentials and endpoint
async function testConnection() {
  try {
    const response = await vault.products.list({ per_page: 1 });
    console.log('✓ Connection successful');
    return true;
  } catch (error) {
    if (error.response?.status === 401) {
      console.error('✗ Invalid consumer key or secret');
      console.error('Check EDD_CONSUMER_KEY and EDD_CONSUMER_SECRET');
    } else if (error.code === 'ENOTFOUND') {
      console.error('✗ Cannot reach API endpoint');
      console.error('Check EDD_API_URL:', process.env.EDD_API_URL);
    } else {
      console.error('✗ Connection error:', error.message);
    }
    return false;
  }
}
```

### Rate Limit Handling

```python
# Implement exponential backoff for rate limits
import time
import requests

def fetch_with_backoff(url, max_retries=5):
    retries = 0
    backoff = 1
    
    while retries < max_retries:
        try:
            response = client._request('GET', url)
            return response
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:
                retry_after = int(e.response.headers.get('Retry-After', backoff))
                print(f"Rate limited. Waiting {retry_after}s before retry {retries+1}/{max_retries}")
                time.sleep(retry_after)
                retries += 1
                backoff *= 2
            else:
                raise
    
    raise Exception(f"Max retries ({max_retries}) exceeded")
```

### Debugging API Responses

```javascript
// Enable verbose logging
const vault = new CommerceVault({
  apiUrl: process.env.EDD_API_URL,
  consumerKey: process.env.EDD_CONSUMER_KEY,
  consumerSecret: process.env.EDD_CONSUMER_SECRET,
  debug: true // Enable request/response logging
});

// Inspect raw response
vault.on('response', (response) => {
  console.log('Request:', response.config.url);
  console.log('Status:', response.status);
  console.log('Headers:', response.headers);
  console.log('Data:', JSON.stringify(response.data, null, 2));
});

// Handle errors with context
vault.on('error', (error) => {
  console.error('API Error:', {
    message: error.message,
    endpoint: error.config?.url,
    status: error.response?.status,
    data: error.response?.data
  });
});
```

### Data Validation

```python
# Validate response schema before processing
from jsonschema import validate, ValidationError

product_schema = {
    "type": "object",
    "required": ["id", "name", "price"],
    "properties": {
        "id": {"type": "integer"},
        "name": {"type": "string"},
        "price": {"type": "string"},
        "status": {"enum": ["publish", "draft", "pending"]}
    }
}

def safe_fetch_product(product_id):
    product = client.products.get(product_id)
    
    try:
        validate(instance=product['data'], schema=product_schema)
        return product['data']
    except ValidationError as e:
        print(f"Invalid product data for ID {product_id}: {e.message}")
        return None
```

## MCP Tool Integration

When using CommerceVault as an MCP server, the following tools are exposed:

- `edd_list_products` - List products with filtering
- `edd_get_product` - Get single product details
- `edd_list_orders` - Query order history
- `edd_get_customer` - Retrieve customer profile
- `edd_get_analytics` - Fetch revenue metrics
- `edd_manage_license` - License key operations

Invoke via MCP protocol:

```json
{
  "method": "tools/call",
  "params": {
    "name": "edd_get_analytics",
    "arguments": {
      "start_date": "2026-01-01",
      "end_date": "2026-12-31",
      "metrics": ["revenue", "orders", "customers"]
    }
  }
}
```
