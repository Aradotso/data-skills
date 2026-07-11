---
name: commercevault-edd-digital-commerce
description: Easy Digital Downloads MCP server for sales analytics, product management, and e-commerce data orchestration
triggers:
  - "connect to Easy Digital Downloads API"
  - "fetch EDD sales analytics and reports"
  - "query digital product catalog from WordPress"
  - "retrieve customer purchase data from EDD"
  - "analyze EDD revenue and order metrics"
  - "manage digital downloads products via API"
  - "setup MCP server for Easy Digital Downloads"
  - "integrate EDD commerce data into analytics"
---

# CommerceVault EDD Digital Commerce Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

CommerceVault (mcp-edd-analytics-vantage) is an MCP (Model Context Protocol) server that provides structured access to Easy Digital Downloads (EDD) data through a unified API layer. It enables AI agents and applications to interact with WordPress-based digital commerce platforms, retrieving sales analytics, product catalogs, customer data, and order information through standardized endpoints.

## What This Project Does

This MCP server acts as a middleware orchestrator between applications and Easy Digital Downloads (EDD) e-commerce backends. It provides:

- **Sales & Revenue Analytics**: Real-time access to GMV, AOV, churn rates, and cohort analysis
- **Product Catalog Management**: Sync and query digital product listings with filtering and caching
- **Order Lifecycle Tracking**: Monitor order states from creation through fulfillment to refunds
- **Customer Segmentation**: Access LTV, purchase frequency, and customer profile data
- **License Management**: Handle digital product licenses, activation codes, and entitlements
- **Webhook Relay**: Bidirectional event handling for order updates and system events

The server follows MCP protocol specifications, allowing Claude Desktop and other MCP clients to interact with EDD installations as data sources.

## Installation

### Prerequisites

- Node.js 18+ or Python 3.9+
- Easy Digital Downloads WordPress installation with REST API enabled
- EDD API credentials (Consumer Key and Secret)

### Install as MCP Server

**For Claude Desktop (Node.js):**

```bash
# Clone the repository
git clone https://github.com/dhapat3927/mcp-edd-analytics-vantage.git
cd mcp-edd-analytics-vantage

# Install dependencies
npm install

# Build the server
npm run build
```

**Configure in Claude Desktop** (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "edd-commerce": {
      "command": "node",
      "args": ["/path/to/mcp-edd-analytics-vantage/dist/index.js"],
      "env": {
        "EDD_API_URL": "https://your-site.com",
        "EDD_CONSUMER_KEY": "${EDD_CONSUMER_KEY}",
        "EDD_CONSUMER_SECRET": "${EDD_CONSUMER_SECRET}",
        "CACHE_ENABLED": "true",
        "REDIS_URL": "redis://localhost:6379"
      }
    }
  }
}
```

**Python Implementation:**

```bash
pip install mcp-edd-analytics-vantage

# Set environment variables
export EDD_API_URL="https://your-site.com"
export EDD_CONSUMER_KEY="ck_your_key_here"
export EDD_CONSUMER_SECRET="cs_your_secret_here"
```

## Configuration

### Environment Variables

| Variable | Required | Description | Default |
|----------|----------|-------------|---------|
| `EDD_API_URL` | Yes | WordPress site URL with EDD installed | - |
| `EDD_CONSUMER_KEY` | Yes | EDD REST API consumer key | - |
| `EDD_CONSUMER_SECRET` | Yes | EDD REST API consumer secret | - |
| `CACHE_ENABLED` | No | Enable Redis caching layer | `false` |
| `REDIS_URL` | No | Redis connection string | `redis://localhost:6379` |
| `RATE_LIMIT_MAX` | No | Max requests per window | `100` |
| `RATE_LIMIT_WINDOW` | No | Rate limit window (seconds) | `60` |
| `LOG_LEVEL` | No | Logging verbosity (debug/info/warn/error) | `info` |
| `HMAC_SECRET` | No | Shared secret for payload signing | - |
| `WEBHOOK_PROXY_URL` | No | Webhook relay endpoint | - |

### Obtaining EDD API Credentials

1. Log into WordPress admin panel
2. Navigate to **Downloads → Settings → API**
3. Click **Generate API Keys**
4. Copy Consumer Key and Consumer Secret
5. Grant appropriate permissions (read/write)

## Core MCP Tools

The server exposes these tools via MCP protocol:

### `edd_get_products`

Retrieve product catalog with filtering and pagination.

```typescript
// Tool call via MCP
{
  "name": "edd_get_products",
  "arguments": {
    "category": "software",
    "status": "publish",
    "limit": 50,
    "page": 1,
    "sort": "sales_desc"
  }
}
```

**Response Structure:**

```json
{
  "products": [
    {
      "id": 1234,
      "title": "Premium WordPress Plugin",
      "price": 49.99,
      "currency": "USD",
      "sales": 342,
      "earnings": 17115.58,
      "status": "publish",
      "download_url": "https://...",
      "licenses": ["regular", "extended"]
    }
  ],
  "pagination": {
    "total": 156,
    "pages": 4,
    "current_page": 1
  }
}
```

### `edd_get_sales_report`

Generate sales analytics for specified time range.

```typescript
{
  "name": "edd_get_sales_report",
  "arguments": {
    "start_date": "2026-06-01",
    "end_date": "2026-07-01",
    "granularity": "daily",
    "metrics": ["revenue", "orders", "avg_order_value"],
    "group_by": "product"
  }
}
```

**Response Structure:**

```json
{
  "summary": {
    "total_revenue": 45678.90,
    "total_orders": 523,
    "avg_order_value": 87.35,
    "refund_rate": 0.023
  },
  "time_series": [
    {
      "date": "2026-06-01",
      "revenue": 1234.56,
      "orders": 18,
      "avg_order_value": 68.59
    }
  ],
  "by_product": [
    {
      "product_id": 1234,
      "product_name": "Premium Plugin",
      "revenue": 5678.90,
      "units_sold": 67
    }
  ]
}
```

### `edd_get_orders`

Query orders with advanced filtering.

```typescript
{
  "name": "edd_get_orders",
  "arguments": {
    "status": ["completed", "processing"],
    "customer_email": "user@example.com",
    "min_total": 50.00,
    "date_from": "2026-06-01",
    "include_refunds": false,
    "limit": 100
  }
}
```

### `edd_get_customer`

Retrieve customer profile with purchase history.

```typescript
{
  "name": "edd_get_customer",
  "arguments": {
    "customer_id": 5678,
    "include_orders": true,
    "include_ltv": true
  }
}
```

**Response Structure:**

```json
{
  "customer": {
    "id": 5678,
    "email": "customer@example.com",
    "name": "Jane Smith",
    "purchase_count": 12,
    "lifetime_value": 567.89,
    "first_purchase": "2024-03-15T10:30:00Z",
    "last_purchase": "2026-06-28T14:22:00Z"
  },
  "orders": [
    {
      "id": 9012,
      "total": 49.99,
      "date": "2026-06-28T14:22:00Z",
      "status": "completed"
    }
  ]
}
```

### `edd_validate_license`

Check license key validity and activation status.

```typescript
{
  "name": "edd_validate_license",
  "arguments": {
    "license_key": "XXXX-XXXX-XXXX-XXXX",
    "product_id": 1234,
    "url": "https://customer-site.com"
  }
}
```

### `edd_create_discount`

Generate promotional discount codes programmatically.

```typescript
{
  "name": "edd_create_discount",
  "arguments": {
    "code": "SUMMER2026",
    "amount": 25,
    "type": "percent",
    "products": [1234, 5678],
    "max_uses": 100,
    "expires": "2026-08-31T23:59:59Z"
  }
}
```

## Code Examples

### Node.js/TypeScript Client

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js';

async function analyzeEDDSales() {
  const transport = new StdioClientTransport({
    command: 'node',
    args: ['dist/index.js'],
    env: {
      EDD_API_URL: process.env.EDD_API_URL,
      EDD_CONSUMER_KEY: process.env.EDD_CONSUMER_KEY,
      EDD_CONSUMER_SECRET: process.env.EDD_CONSUMER_SECRET,
    },
  });

  const client = new Client({
    name: 'edd-analytics-client',
    version: '1.0.0',
  }, {
    capabilities: {},
  });

  await client.connect(transport);

  // Get sales report for last 30 days
  const salesReport = await client.callTool({
    name: 'edd_get_sales_report',
    arguments: {
      start_date: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString().split('T')[0],
      end_date: new Date().toISOString().split('T')[0],
      granularity: 'daily',
      metrics: ['revenue', 'orders', 'avg_order_value'],
    },
  });

  console.log('30-Day Sales Summary:', salesReport.content);

  // Get top-selling products
  const products = await client.callTool({
    name: 'edd_get_products',
    arguments: {
      sort: 'sales_desc',
      limit: 10,
      status: 'publish',
    },
  });

  console.log('Top 10 Products:', products.content);

  await client.close();
}
```

### Python Client

```python
import os
import asyncio
from datetime import datetime, timedelta
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def analyze_edd_revenue():
    server_params = StdioServerParameters(
        command="node",
        args=["dist/index.js"],
        env={
            "EDD_API_URL": os.getenv("EDD_API_URL"),
            "EDD_CONSUMER_KEY": os.getenv("EDD_CONSUMER_KEY"),
            "EDD_CONSUMER_SECRET": os.getenv("EDD_CONSUMER_SECRET"),
        }
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # Get monthly revenue breakdown
            end_date = datetime.now()
            start_date = end_date - timedelta(days=90)
            
            result = await session.call_tool(
                "edd_get_sales_report",
                arguments={
                    "start_date": start_date.strftime("%Y-%m-%d"),
                    "end_date": end_date.strftime("%Y-%m-%d"),
                    "granularity": "monthly",
                    "metrics": ["revenue", "orders", "refund_rate"],
                    "group_by": "product"
                }
            )
            
            print(f"Quarterly Revenue: ${result.content['summary']['total_revenue']:.2f}")
            print(f"Total Orders: {result.content['summary']['total_orders']}")
            
            # Customer LTV analysis
            customers = await session.call_tool(
                "edd_get_customers",
                arguments={
                    "min_ltv": 500.00,
                    "sort": "ltv_desc",
                    "limit": 50
                }
            )
            
            for customer in customers.content['customers']:
                print(f"{customer['email']}: LTV ${customer['lifetime_value']:.2f}")

asyncio.run(analyze_edd_revenue())
```

### Webhook Event Handler

```typescript
import express from 'express';
import crypto from 'crypto';

const app = express();
app.use(express.json());

// Webhook receiver for EDD events
app.post('/webhooks/edd', (req, res) => {
  const signature = req.headers['x-edd-signature'];
  const payload = JSON.stringify(req.body);
  
  // Verify HMAC signature
  const expectedSignature = crypto
    .createHmac('sha256', process.env.HMAC_SECRET!)
    .update(payload)
    .digest('hex');
  
  if (signature !== expectedSignature) {
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  const { event, data } = req.body;
  
  switch (event) {
    case 'order.completed':
      console.log('Order completed:', data.order_id);
      // Trigger fulfillment, send license keys, etc.
      break;
    
    case 'subscription.renewed':
      console.log('Subscription renewed:', data.subscription_id);
      break;
    
    case 'refund.processed':
      console.log('Refund issued:', data.order_id);
      // Revoke licenses, update inventory
      break;
  }
  
  res.status(200).json({ received: true });
});

app.listen(3000);
```

## Common Patterns

### Revenue Dashboard Generation

```typescript
async function generateDashboard(client: Client, dateRange: { start: string; end: string }) {
  const [sales, products, customers] = await Promise.all([
    client.callTool({
      name: 'edd_get_sales_report',
      arguments: {
        start_date: dateRange.start,
        end_date: dateRange.end,
        granularity: 'daily',
        metrics: ['revenue', 'orders', 'avg_order_value'],
      },
    }),
    client.callTool({
      name: 'edd_get_products',
      arguments: { sort: 'earnings_desc', limit: 10 },
    }),
    client.callTool({
      name: 'edd_get_customers',
      arguments: { sort: 'ltv_desc', limit: 25 },
    }),
  ]);

  return {
    kpis: sales.content.summary,
    topProducts: products.content.products,
    vipCustomers: customers.content.customers,
    trend: sales.content.time_series,
  };
}
```

### Customer Cohort Analysis

```typescript
async function analyzeCohorts(client: Client, cohortMonth: string) {
  const result = await client.callTool({
    name: 'edd_get_cohort_analysis',
    arguments: {
      cohort_month: cohortMonth,
      metric: 'retention_rate',
      periods: 12,
    },
  });

  // Result shows month-over-month retention
  // cohort_month: "2026-01", periods: 3
  // Month 1: 100% (baseline)
  // Month 2: 65% (retained)
  // Month 3: 48% (retained)
  return result.content;
}
```

### Automated License Provisioning

```typescript
async function provisionLicense(client: Client, orderId: number) {
  const order = await client.callTool({
    name: 'edd_get_order',
    arguments: { order_id: orderId },
  });

  for (const item of order.content.items) {
    if (item.is_licensed) {
      const license = await client.callTool({
        name: 'edd_generate_license',
        arguments: {
          product_id: item.product_id,
          customer_email: order.content.customer.email,
          activations_limit: item.license_tier === 'extended' ? 25 : 1,
        },
      });

      // Send license key via email or API
      console.log(`License generated: ${license.content.license_key}`);
    }
  }
}
```

## Troubleshooting

### Authentication Failures

**Issue:** `401 Unauthorized` errors when calling tools.

**Solution:**

```bash
# Verify API credentials are correct
curl -u "ck_YOUR_KEY:cs_YOUR_SECRET" \
  https://your-site.com/wp-json/edd/v2/products

# Check key permissions in WordPress admin
# Ensure read/write permissions are enabled
```

### Rate Limiting

**Issue:** `429 Too Many Requests` errors.

**Solution:**

```typescript
// Enable adaptive rate limiting in config
env: {
  RATE_LIMIT_MAX: "60",
  RATE_LIMIT_WINDOW: "60",
  CACHE_ENABLED: "true", // Reduces redundant API calls
}
```

### Cache Invalidation

**Issue:** Stale data returned from cached endpoints.

**Solution:**

```typescript
// Force cache refresh
const products = await client.callTool({
  name: 'edd_get_products',
  arguments: {
    cache_bypass: true, // Ignores cache
    limit: 100,
  },
});

// Or clear specific cache keys via Redis
// REDIS: DEL edd:products:*
```

### Webhook Signature Verification Failures

**Issue:** Webhook events rejected due to signature mismatch.

**Solution:**

```typescript
// Ensure HMAC_SECRET matches both server and receiver
// Check raw payload (before JSON parsing)
const rawBody = req.body; // Use express.raw() middleware
const signature = crypto
  .createHmac('sha256', process.env.HMAC_SECRET!)
  .update(rawBody)
  .digest('hex');
```

### Connection Timeouts

**Issue:** MCP client fails to connect to server.

**Solution:**

```json
{
  "mcpServers": {
    "edd-commerce": {
      "command": "node",
      "args": ["dist/index.js"],
      "timeout": 30000, // Increase timeout to 30s
      "env": { ... }
    }
  }
}
```

### Data Type Mismatches

**Issue:** Tool arguments rejected due to type errors.

**Solution:**

```typescript
// Ensure numeric values are numbers, not strings
{
  "name": "edd_get_products",
  "arguments": {
    "limit": 50,        // ✅ Number
    "page": 1,          // ✅ Number
    "min_price": "9.99" // ❌ Should be number: 9.99
  }
}
```

## Advanced Configuration

### Multi-Tenant Setup

```json
{
  "mcpServers": {
    "edd-store-a": {
      "command": "node",
      "args": ["dist/index.js"],
      "env": {
        "EDD_API_URL": "https://store-a.com",
        "EDD_CONSUMER_KEY": "${STORE_A_KEY}",
        "EDD_CONSUMER_SECRET": "${STORE_A_SECRET}",
        "TENANT_ID": "store_a"
      }
    },
    "edd-store-b": {
      "command": "node",
      "args": ["dist/index.js"],
      "env": {
        "EDD_API_URL": "https://store-b.com",
        "EDD_CONSUMER_KEY": "${STORE_B_KEY}",
        "EDD_CONSUMER_SECRET": "${STORE_B_SECRET}",
        "TENANT_ID": "store_b"
      }
    }
  }
}
```

### Audit Logging

```typescript
// Enable immutable audit trail
env: {
  AUDIT_LOG_ENABLED: "true",
  AUDIT_LOG_STORAGE: "postgres", // or "qldb" or "kafka"
  AUDIT_LOG_DSN: "postgresql://user:pass@localhost/audit"
}
```

This MCP server enables AI agents to perform sophisticated e-commerce data analysis, automate digital product fulfillment, and integrate EDD stores into broader analytics and automation workflows.
