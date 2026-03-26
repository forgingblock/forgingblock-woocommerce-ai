# ForgingBlock WooCommerce Checkout (Built for Humans + AI Agents)

A modern WooCommerce payment gateway for **ForgingBlock API** focused on conversions, cleaner payment UX, and AI-agent compatibility.

## Why this plugin exists

This plugin is optimized for both:

- **Human checkout** (direct customer payment in WooCommerce)
- **AI-agent checkout** (programmatic resolve/verify flows)

It uses ForgingBlock Orders API and now redirects customers to **`invoice_url`** (single-use invoice) instead of `checkout_url`.

## Important plugin behavior (must know)

- During checkout, this plugin redirects to **`invoice_url`**.

## Features

- `POST /api/v1/orders` order creation (Bearer authentication)
- `GET /api/v1/orders/{orderId}` status sync on thank-you page (Bearer authentication)
- Callback endpoint support: `?wc-api=wc_gateway_forgingblock`
- Status mapping support:
  - `new`
  - `partially_paid`
  - `completed`
  - `expired`
## AI agent routes

- `GET /wp-json/forgingblock/v1/agent/resolve?id=<checkoutId>`
- `GET /wp-json/forgingblock/v1/agent/resolve?url=<checkoutOrInvoiceUrl>`
- `GET /wp-json/forgingblock/v1/agent/verify/<invoiceId>`
- `POST /wp-json/forgingblock/v1/agent/checkout`
- `POST /wp-json/forgingblock/v1/agent/create-order`
- `GET /wp-json/forgingblock/v1/agent/products`
- `GET /wp-json/forgingblock/v1/agent/products/{id}`

---

### Route purposes

#### resolve

```http
GET /wp-json/forgingblock/v1/agent/resolve
````

Normalize checkout or invoice into agent-friendly format.

Supports:

* `id=<checkoutId>`
* `url=<checkoutOrInvoiceUrl>`

Returns:

* invoice_id
* invoice_url
* payment routes
* recommended_tx (if available)

---

#### products

```http
GET /wp-json/forgingblock/v1/agent/products?q=search&page=1
GET /wp-json/forgingblock/v1/agent/products/{id}
```

Used for:

* catalog browsing
* agent-side filtering
* product selection

---

#### create-order

```http
POST /wp-json/forgingblock/v1/agent/create-order
```

```json
{
  "product_id": 123,
  "quantity": 1
}
```

Creates a WooCommerce order and returns:

* order_id
* order_key
* total
* currency

---

#### checkout

```http
POST /wp-json/forgingblock/v1/agent/checkout
```

```json
{
  "order_id": 987,
  "order_key": "wc_order_xxx"
}
```

Converts Woo order → ForgingBlock invoice.

Returns:

* invoice_url
* remote_order_id
* status
* totals

**Access model:**

* allowed for logged-in users
* OR guests with valid `order_id + order_key`
* otherwise returns 401

---

#### verify

```http
GET /wp-json/forgingblock/v1/agent/verify/{invoiceId}
```

Returns payment state:

```
new → partially_paid → completed → expired
```

---

### Access model (important)

```
products / resolve / verify → public (read-only)

create-order → public

checkout → requires order_key OR logged-in user
```
---

## Install guide

### Install from a [ZIP file](https://github.com/forgingblock/forgingblock-woocommerce-ai/releases/download/v3.0.3/forgingblock-woocommerce-plugin.zip) (recommended)

1. In WordPress Admin, go to **Plugins → Add New Plugin → Upload Plugin**.
2. Choose the plugin ZIP.
3. Click **Install Now** and then **Activate**.
4. Go to **WooCommerce → Settings → Payments → ForgingBlock CommerceFlow**.
5. Turn it on and paste your ForgingBlock API key (**[Dashboard](https://dash.forgingblock.io) → Account Settings → Integrations → API Token**).
6. Save.
---

## Human checkout flow

1. Shopper chooses ForgingBlock Pay at checkout.
2. WooCommerce creates order via ForgingBlock.
3. Redirect to **invoice_url**.
4. Shopper pays.
5. Store updates via callback or polling.

---

## Callback URL

```
https://YOUR_STORE_DOMAIN/?wc-api=wc_gateway_forgingblock
```

---

# 🧠 AI Agent Integration (Recommended Flow)

Use:

[https://github.com/forgingblock/forgingblock-agentkit](https://github.com/forgingblock/forgingblock-agentkit)

---

## Agent Architecture

```
AI Agent (AgentKit)
        │
        ▼
Next.js Agent API (/api/agent)
        │
        ├───────────────┬──────────────────────────────┐
        ▼               ▼                              ▼
Woo Plugin        ForgingBlock API               Wallet Provider
(agent routes)         │                              │
        │              ▼                              ▼
        │        invoice_url                   ERC20 transfer
        │              │                              │
        ▼              ▼                              ▼
Woo Order → Checkout → Payment → Verify → Completed
```

---

## End-to-End Agent Flow

### 1. Create or obtain Woo order

```
order_id
order_key
```

Via:

* Woo REST API
* Woo MCP tools
* existing checkout

---

### 2. Convert Woo order → blockchain invoice

```
POST /wp-json/forgingblock/v1/agent/checkout
```

```json
{
  "order_id": 987,
  "order_key": "wc_order_xxx"
}
```

---

### 3. Receive invoice

Response includes:

* `invoice_url`
* `remote_order_id`
* `redirect_url`

---

### 4. Create payment (AgentKit)

```
invoice_url → create_payment → recommended_tx
```

---

### 5. Execute transaction

```
ERC20ActionProvider_transfer
```

---

### 6. Verify payment

```
GET /wp-json/forgingblock/v1/agent/verify/{invoiceId}
```

---

## Deterministic Agent Routing

```
search → Woo APIs
checkout → /agent/checkout
payment → create_payment
execute → wallet
verify → /agent/verify
```

---

## 🔐 Security model

This plugin uses a **layered security model** designed for both human users and AI agents.

---

### Access layers

```

Public (no auth)
│
▼
Read-only endpoints
(products, resolve, verify)
│
▼
Capability-based access
(order_id + order_key)
│
▼
Checkout (invoice creation)
│
▼
Server-side API (ForgingBlock)
│
▼
Blockchain payment

````

---

### Endpoint access

| Endpoint | Access |
|--------|--------|
| `/agent/products` | Public |
| `/agent/products/{id}` | Public |
| `/agent/resolve` | Public |
| `/agent/verify/{invoiceId}` | Public |
| `/agent/create-order` | Public |
| `/agent/checkout` | Requires `order_id + order_key` OR logged-in user |

---

### Capability-based checkout

Checkout does **not require full authentication**.

Instead it uses:

```text
order_id + order_key
````

This acts as a **capability token**:

* grants access to a specific order
* cannot be used to access other orders
* safe to expose to AI agents

---

### Why this design

#### 1. Safe for agents

Agents can:

* create order
* checkout
* pay

without needing:

```text
❌ WordPress login
❌ API keys
❌ cookies
```

---

#### 2. No merchant key exposure

* ForgingBlock API key is **never exposed**
* all sensitive calls happen server-side

---

#### 3. Read-only endpoints are public

Safe because they:

* do not mutate state
* only return payment / catalog data

---

### Trust boundaries

```
Agent
  │
  ├── public endpoints (safe)
  │
  ├── checkout (requires order_key)
  │
  ▼
Woo Plugin (trusted boundary)
  │
  ▼
ForgingBlock API (secret key)
  │
  ▼
Blockchain
```

---

### Summary

```text
Public read → Capability write → Server execution → Blockchain
```

```
---

## Optional MCP Integration

* [https://github.com/WordPress/mcp-adapter](https://github.com/WordPress/mcp-adapter)
* [https://developer.woocommerce.com/docs/features/mcp/](https://developer.woocommerce.com/docs/features/mcp/)

---

## Minimal Agent Example

```ts
const checkout = await fetch('/agent/checkout', {...})

const payment = await create_payment(checkout.invoice_url)

await ERC20ActionProvider_transfer(payment)

await fetch(`/agent/verify/${payment.invoiceId}`)
```

---

## Summary

```
Woo order → agent/checkout → invoice_url
→ AgentKit → create_payment
→ wallet → transfer
→ agent/verify → completed
```

