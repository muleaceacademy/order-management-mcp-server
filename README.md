# Order Management MCP Server

A production-quality MCP (Model Context Protocol) server built with **MuleSoft**, exposing **9 order management tools** via the Streamable HTTP transport using protocol `2025-06-18`.

This server demonstrates how to build a full-featured AI-integrated commerce backend using the MuleSoft MCP Connector with mock data — covering order search, customer lookup, invoice creation, shipment tracking, payments, refunds, and inventory management.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Tools Reference](#tools-reference)
- [Mock Data](#mock-data)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Build](#build)
- [Run Locally](#run-locally)
- [Endpoints](#endpoints)
- [MCP Inspector Connection](#mcp-inspector-connection)
- [MCP Protocol Walkthrough](#mcp-protocol-walkthrough)
- [Sample Tool Calls](#sample-tool-calls)
- [Error Handling](#error-handling)
- [MUnit Test Suite](#munit-test-suite)
- [Anypoint Exchange Publication](#anypoint-exchange-publication)
- [CloudHub 2.0 Deployment](#cloudhub-20-deployment)

---

## Architecture Overview

```
MCP Client (Claude / Cursor / Inspector)
         │
         │  HTTP POST /mcp  (JSON-RPC 2.0, Streamable HTTP)
         ▼
┌─────────────────────────────────────────────────────┐
│              Order Management MCP Server             │
│                  (Mule 4.11.6)                       │
│                                                      │
│  ┌─────────────┐   ┌───────────────────────────┐    │
│  │ mcp-server  │   │       Tool Flows           │    │
│  │   .xml      │   │                           │    │
│  │  Session    │   │  order-tools.xml           │    │
│  │   Init      │   │  ├─ get_order_status       │    │
│  └─────────────┘   │  └─ get_shipping_eta       │    │
│                    │                           │    │
│  ┌─────────────┐   │  order-search-tools.xml   │    │
│  │  global-    │   │  └─ search_orders          │    │
│  │  config.xml │   │                           │    │
│  │  HTTP +MCP  │   │  customer-tools.xml        │    │
│  │  Config     │   │  └─ customer_lookup        │    │
│  └─────────────┘   │                           │    │
│                    │  fulfillment-tools.xml     │    │
│  ┌─────────────┐   │  ├─ track_shipment         │    │
│  │  health-    │   │  └─ check_inventory        │    │
│  │  endpoint   │   │                           │    │
│  │    .xml     │   │  financial-tools.xml       │    │
│  └─────────────┘   │  ├─ create_invoice         │    │
│                    │  ├─ get_payment_status      │    │
│                    │  └─ process_refund          │    │
│                    └───────────────────────────┘    │
│                               │                      │
│                    ┌──────────▼──────────┐           │
│                    │   Mock Data Layer   │           │
│                    │  (JSON classpath)   │           │
│                    └─────────────────────┘           │
└─────────────────────────────────────────────────────┘
```

---

## Tools Reference

The server exposes **9 MCP tools** total:

### Order Tools

| Tool | Description | Required Input | Optional Input |
|------|-------------|----------------|----------------|
| `get_order_status` | Returns current status, last-updated timestamp, and message for an order | `orderId` | — |
| `get_shipping_eta` | Returns carrier, tracking number, shipping status, and estimated delivery date | `orderId` | — |
| `search_orders` | Searches orders with optional filters; returns a summary list | — | `status`, `customerId`, `dateFrom`, `dateTo` |

### Customer Tools

| Tool | Description | Required Input | Optional Input |
|------|-------------|----------------|----------------|
| `customer_lookup` | Retrieves customer profile including address, tier, and order count | `customerId` OR `email` | — |

### Fulfillment Tools

| Tool | Description | Required Input | Optional Input |
|------|-------------|----------------|----------------|
| `track_shipment` | Returns carrier, shipment status, ETA, and full milestone event history | `trackingNumber` OR `orderId` | — |
| `check_inventory` | Returns stock level, availability status, warehouse location, and reorder threshold | `sku` | — |

### Financial Tools

| Tool | Description | Required Input | Optional Input |
|------|-------------|----------------|----------------|
| `create_invoice` | Retrieves or generates an invoice with line items, subtotal, tax, and total | `orderId` | — |
| `get_payment_status` | Returns payment method, transaction ID, amount, currency, and status | `orderId` OR `paymentId` | — |
| `process_refund` | Initiates a new refund or retrieves an existing refund record | `orderId` | `reason`, `amount` |

---

## Mock Data

All data lives in `src/main/resources/` as plain JSON — edit any file without touching flow XML.

### Orders (`mock-orders.json`) — 5 records

| Order ID | Customer | Status | Total |
|----------|----------|--------|-------|
| ORD-1001 | CUST-101 | PROCESSING | $79.97 |
| ORD-1002 | CUST-102 | SHIPPED (Blue Dart) | $89.99 |
| ORD-1003 | CUST-101 | DELIVERED (DHL) | $45.00 |
| ORD-1004 | CUST-103 | CANCELLED | $29.99 |
| ORD-1005 | CUST-102 | PENDING | $124.98 |

### Customers (`mock-customers.json`) — 3 records

| Customer ID | Name | Email | Tier |
|-------------|------|-------|------|
| CUST-101 | Alice Johnson | alice.johnson@example.com | GOLD |
| CUST-102 | Ravi Kumar | ravi.kumar@example.com | SILVER |
| CUST-103 | Priya Patel | priya.patel@example.com | BRONZE |

### Inventory (`mock-inventory.json`) — 5 SKUs

| SKU | Product | Status | Available |
|-----|---------|--------|-----------|
| SKU-A100 | Wireless Mouse | IN_STOCK | 138 |
| SKU-B200 | USB Hub | IN_STOCK | 70 |
| SKU-C300 | Mechanical Keyboard | LOW_STOCK | 6 |
| SKU-D400 | Monitor Stand | OUT_OF_STOCK | 0 |
| SKU-E500 | Laptop Stand | IN_STOCK | 37 |

### Payments (`mock-payments.json`) — 5 records
PAY-3001 → PAY-3005, linked to each order. Methods: CREDIT_CARD, UPI, NET_BANKING, DEBIT_CARD.

### Invoices (`mock-invoices.json`) — 4 records
INV-2001 → INV-2004, each with line items, subtotal, tax, and total.

### Refunds (`mock-refunds.json`) — 2 records
REF-4001 (ORD-1003, COMPLETED), REF-4002 (ORD-1004, COMPLETED).
For orders without an existing refund, `process_refund` simulates a new `INITIATED` refund.

---

## Prerequisites

| Requirement | Version |
|---|---|
| Java (Temurin or similar) | 17 |
| Maven | 3.8+ |
| Mule Runtime | 4.11.6 (auto-managed by `mule-maven-plugin`) |
| MCP Connector | 1.6.1 |
| MUnit | 3.7.3 |
| Anypoint CLI v4 (Exchange publish only) | latest |

---

## Project Structure

```
order-management-mcp-server/
├── mcp-spec.json                          ← MCP tool manifest (all 9 tools)
├── exchange.json                          ← Anypoint Exchange metadata
├── pom.xml                               ← Maven build descriptor
│
├── src/main/mule/
│   ├── global-config.xml                 ← HTTP Listener + MCP Server config
│   ├── mcp-server.xml                    ← Session initialization flow
│   ├── health-endpoint.xml               ← GET /health
│   ├── order-tools.xml                   ← get_order_status, get_shipping_eta
│   ├── order-search-tools.xml            ← search_orders
│   ├── customer-tools.xml                ← customer_lookup
│   ├── fulfillment-tools.xml             ← track_shipment, check_inventory
│   └── financial-tools.xml               ← create_invoice, get_payment_status, process_refund
│
├── src/main/resources/
│   ├── application.properties            ← http.host, http.port, mcp.basePath
│   ├── log4j2.xml                        ← Logging configuration
│   ├── mock-orders.json                  ← 5 order records (extended schema)
│   ├── mock-customers.json               ← 3 customer records
│   ├── mock-inventory.json               ← 5 product SKUs
│   ├── mock-payments.json                ← 5 payment records
│   ├── mock-invoices.json                ← 4 invoices with line items
│   └── mock-refunds.json                 ← 2 refund records
│
└── src/test/munit/
    └── order-management-test-suite.xml   ← MUnit tests
```

---

## Build

```bash
cd order-management-mcp-server
mvn clean package
```

Output: `target/order-management-mcp-server-1.0.0-SNAPSHOT-mule-application.jar`

---

## Run Locally

```bash
mvn clean package
cd target
java -jar order-management-mcp-server-1.0.0-SNAPSHOT-mule-application.jar
```

The server starts on `http://0.0.0.0:8081`.

---

## Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `http://localhost:8081/mcp` | POST | MCP Streamable HTTP transport (all MCP protocol messages) |
| `http://localhost:8081/health` | GET | Health check — returns `{"status":"UP"}` |

### Health Check

```bash
curl http://localhost:8081/health
```

```json
{
  "application": "order-management-mcp-server",
  "status": "UP"
}
```

---

## MCP Inspector Connection

[MCP Inspector](https://github.com/modelcontextprotocol/inspector) is the standard GUI tool for exploring MCP servers interactively.

1. Start MCP Inspector:
   ```bash
   npx @modelcontextprotocol/inspector
   ```
2. Open the Inspector UI in your browser (typically `http://localhost:5173`).
3. Select transport **Streamable HTTP**.
4. Enter the MCP URL: `http://localhost:8081/mcp`
5. Click **Connect**.
6. Use the **Tools** tab to call any of the 9 tools with JSON arguments.

---

## MCP Protocol Walkthrough

All calls use JSON-RPC 2.0 over `POST http://localhost:8081/mcp`.

### Step 1 — Initialize the session

```bash
curl -si -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-06-18",
      "capabilities": {},
      "clientInfo": {"name": "curl-test", "version": "1.0"}
    }
  }'
```

Copy the `Mcp-Session-Id` header value from the response and export it:

```bash
export SESSION_ID="<value from Mcp-Session-Id header>"
```

### Step 2 — List all tools

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {}}'
```

Returns all **9 tools** with their `inputSchema` and `outputSchema`.

---

## Sample Tool Calls

### `get_order_status`

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_order_status","arguments":{"orderId":"ORD-1001"}}}'
```

```json
{"orderId":"ORD-1001","status":"PROCESSING","lastUpdated":"2026-08-15T08:30:00Z","message":"The order is currently being processed."}
```

---

### `get_shipping_eta`

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"get_shipping_eta","arguments":{"orderId":"ORD-1002"}}}'
```

```json
{"orderId":"ORD-1002","shippingStatus":"SHIPPED","carrier":"Blue Dart","trackingNumber":"BD784512963IN","estimatedDeliveryDate":"2026-08-18"}
```

---

### `search_orders` — filter by status

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"search_orders","arguments":{"status":"SHIPPED"}}}'
```

```json
{"totalResults":1,"orders":[{"orderId":"ORD-1002","customerId":"CUST-102","status":"SHIPPED","orderDate":"2026-08-13","orderTotal":89.99}]}
```

### `search_orders` — filter by customerId

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":6,"method":"tools/call","params":{"name":"search_orders","arguments":{"customerId":"CUST-101"}}}'
```

```json
{"totalResults":2,"orders":[{"orderId":"ORD-1001",...},{"orderId":"ORD-1003",...}]}
```

### `search_orders` — all orders (no filters)

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{"name":"search_orders","arguments":{}}}'
```

---

### `customer_lookup` — by customerId

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":8,"method":"tools/call","params":{"name":"customer_lookup","arguments":{"customerId":"CUST-101"}}}'
```

```json
{"customerId":"CUST-101","name":"Alice Johnson","email":"alice.johnson@example.com","phone":"+91-9876543210","address":{"street":"42 MG Road","city":"Bengaluru","state":"Karnataka","zip":"560001","country":"India"},"tier":"GOLD","totalOrders":2,"createdAt":"2024-03-15"}
```

### `customer_lookup` — by email

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":9,"method":"tools/call","params":{"name":"customer_lookup","arguments":{"email":"ravi.kumar@example.com"}}}'
```

---

### `track_shipment` — by orderId

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":10,"method":"tools/call","params":{"name":"track_shipment","arguments":{"orderId":"ORD-1003"}}}'
```

```json
{
  "orderId": "ORD-1003",
  "trackingNumber": "DHL963258741",
  "carrier": "DHL",
  "shippingStatus": "DELIVERED",
  "estimatedDeliveryDate": "2026-08-12",
  "milestones": [
    {"event":"Order Placed",         "location":"Warehouse",           "timestamp":"2026-08-10T08:00:00Z"},
    {"event":"Picked Up by Carrier", "location":"Dispatch Center",     "timestamp":"2026-08-10T18:00:00Z"},
    {"event":"In Transit",           "location":"Regional Sorting Hub","timestamp":"2026-08-10T22:00:00Z"},
    {"event":"Out for Delivery",     "location":"Local Delivery Hub",  "timestamp":"2026-08-12T07:00:00Z"},
    {"event":"Delivered",            "location":"Customer Address",    "timestamp":"2026-08-12T14:20:00Z"}
  ]
}
```

---

### `check_inventory` — by SKU

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":11,"method":"tools/call","params":{"name":"check_inventory","arguments":{"sku":"SKU-C300"}}}'
```

```json
{"sku":"SKU-C300","name":"Mechanical Keyboard","category":"Computer Peripherals","stockQuantity":8,"reservedQuantity":2,"availableQuantity":6,"warehouseLocation":"WH-CHN-C2","reorderLevel":10,"unitPrice":89.99,"status":"LOW_STOCK"}
```

---

### `create_invoice`

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":12,"method":"tools/call","params":{"name":"create_invoice","arguments":{"orderId":"ORD-1001"}}}'
```

```json
{
  "invoiceId": "INV-2001",
  "orderId": "ORD-1001",
  "customerId": "CUST-101",
  "invoiceDate": "2026-08-14",
  "dueDate": "2026-08-28",
  "status": "PENDING",
  "currency": "USD",
  "subtotal": 79.97,
  "taxRate": 0.09,
  "taxAmount": 7.20,
  "total": 87.17,
  "lineItems": [
    {"sku":"SKU-A100","description":"Wireless Mouse","qty":2,"unitPrice":29.99,"lineTotal":59.98},
    {"sku":"SKU-B200","description":"USB Hub","qty":1,"unitPrice":19.99,"lineTotal":19.99}
  ]
}
```

---

### `get_payment_status` — by orderId

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":13,"method":"tools/call","params":{"name":"get_payment_status","arguments":{"orderId":"ORD-1002"}}}'
```

```json
{"paymentId":"PAY-3002","orderId":"ORD-1002","customerId":"CUST-102","paymentMethod":"UPI","cardLast4":null,"paymentStatus":"CAPTURED","amount":89.99,"currency":"USD","paymentDate":"2026-08-13T11:30:00Z","transactionId":"TXN-DEF789012"}
```

---

### `process_refund` — existing refund

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":14,"method":"tools/call","params":{"name":"process_refund","arguments":{"orderId":"ORD-1003"}}}'
```

```json
{"refundId":"REF-4001","orderId":"ORD-1003","paymentId":"PAY-3003","customerId":"CUST-101","refundAmount":45.0,"currency":"USD","reason":"Customer requested return after delivery","status":"COMPLETED","requestedAt":"2026-08-13T10:00:00Z","processedAt":"2026-08-14T08:30:00Z"}
```

### `process_refund` — simulate new refund (with reason)

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":15,"method":"tools/call","params":{"name":"process_refund","arguments":{"orderId":"ORD-1002","reason":"Item damaged on arrival","amount":50.00}}}'
```

```json
{"refundId":"REF-MOCK-ORD-1002","orderId":"ORD-1002","paymentId":"PAY-3002","customerId":"CUST-102","refundAmount":50.0,"currency":"USD","reason":"Item damaged on arrival","status":"INITIATED","requestedAt":"2026-08-24T...","processedAt":null}
```

---

## Error Handling

Every tool returns a structured error when validation fails or a record is not found.
Errors are returned as MCP tool responses with `isError: true`.

### Error Codes

| Code | Trigger |
|------|---------|
| `ORDER_NOT_FOUND` | `orderId` does not match any order record |
| `CUSTOMER_NOT_FOUND` | `customerId` or `email` does not match any customer |
| `SHIPMENT_NOT_FOUND` | `trackingNumber` or `orderId` does not match any order |
| `PRODUCT_NOT_FOUND` | `sku` does not match any inventory record |
| `INVOICE_NOT_FOUND` | `orderId` has no invoice (e.g. CANCELLED orders) |
| `PAYMENT_NOT_FOUND` | `orderId` or `paymentId` does not match any payment |
| `INVALID_INPUT` | Required parameter is missing or empty |
| `INTERNAL_ERROR` | Unexpected runtime error |

### Example — `ORDER_NOT_FOUND`

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":99,"method":"tools/call","params":{"name":"get_order_status","arguments":{"orderId":"ORD-9999"}}}'
```

```json
{
  "result": {
    "content": [{"type":"text","text":"{\"code\":\"ORDER_NOT_FOUND\",\"message\":\"No order was found for the supplied orderId.\",\"orderId\":\"ORD-9999\"}"}],
    "isError": true
  }
}
```

### Example — `INVALID_INPUT` (missing required parameter)

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":100,"method":"tools/call","params":{"name":"customer_lookup","arguments":{}}}'
```

```json
{
  "result": {
    "content": [{"type":"text","text":"{\"code\":\"INVALID_INPUT\",\"message\":\"Provide either customerId or email to look up a customer.\"}"}],
    "isError": true
  }
}
```

---

## MUnit Test Suite

```bash
mvn clean test
```

The test suite covers the original two tools. To extend tests for the new tools, add test cases to
`src/test/munit/order-management-test-suite.xml` following the existing pattern.

---

## Anypoint Exchange Publication

### Prerequisites

- Anypoint CLI v4 installed: `npm install -g @mulesoft/anypoint-cli-v4`
- Connected-app or username/password credentials configured
- Your **Organization ID** (Business Group ID from Anypoint Platform)

### Publish

1. Run all tests: `mvn clean test`
2. Review the Exchange metadata in `exchange.json`
3. Publish:

```bash
anypoint-cli-v4 exchange asset upload \
  --organization <YOUR_ORG_ID> \
  --name "Order Management MCP Server" \
  --type custom \
  --version 1.0.0 \
  --file exchange.json
```

---

## CloudHub 2.0 Deployment

### Prerequisites

1. A CloudHub 2.0 target (Private Space or Shared Space) in your Anypoint organization.
2. The application JAR: `target/order-management-mcp-server-1.0.0-SNAPSHOT-mule-application.jar`
3. Anypoint CLI v4 with the DX plugin installed.
4. Environment-specific properties for `http.host`, `http.port`, `mcp.basePath`.

### Deploy

```bash
anypoint-cli-v4 dx mule deploy order-management-mcp-server \
  --environment <ENVIRONMENT_NAME> \
  --runtime 4.11.6 \
  --file target/order-management-mcp-server-1.0.0-SNAPSHOT-mule-application.jar
```

---

## References

- [MuleSoft MCP Connector Documentation](https://docs.mulesoft.com/general/)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/specification/2025-06-18)
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
- [Anypoint Exchange](https://www.anypoint.mulesoft.com/exchange/)
