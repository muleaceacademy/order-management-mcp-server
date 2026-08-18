# Stop MCP Tool Poisoning with MuleSoft Omni Gateway

Introduces Tool Poisoning Detection. Omni Gateway pins the trusted toolset from Exchange and detects unauthorized changes in backend tools/list responses.
---
<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/bd90b192-09b6-42c5-bbe3-934c4da00bea" />


<img width="1110" height="605" alt="image" src="https://github.com/user-attachments/assets/ff9ba25e-1cbd-4877-b23e-6999553999f5" />

For this demonstration, I have created a simple Order Management MCP Server with two tools:

`get_order_status`

And `get_shipping_eta`

The trusted description of `get_order_status` says:

“Returns the current status of an order using the provided order ID.”


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

| Endpoint | Description |
|---|---|
| `POST http://localhost:8081/mcp` | MCP Streamable HTTP transport (all MCP protocol messages) |
| `GET  http://localhost:8081/health` | Health check |

---

## Health Check

```bash
curl http://localhost:8081/health
```

Response:

```json
{
  "application": "order-management-mcp-server",
  "status": "UP"
}
```

---

## MCP Inspector Connection

[MCP Inspector](https://github.com/modelcontextprotocol/inspector) is the standard GUI tool for
exploring MCP servers.

### Connect MCP Inspector

1. Start MCP Inspector: `npx @modelcontextprotocol/inspector`
2. Open the Inspector UI in your browser (typically `http://localhost:5173`).
3. Select transport **Streamable HTTP**.
4. Enter the MCP URL: `http://localhost:8081/mcp`
5. Click **Connect**.

---

## Initialize the MCP Session

```bash
curl -s -X POST http://localhost:8081/mcp \
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

Expected response contains `result.serverInfo.name: "Order Management MCP Server"`.

Note the `Mcp-Session-Id` header in the response — you must include it in all subsequent requests.

---

## List Tools (`tools/list`)

```bash
SESSION_ID="<value from Mcp-Session-Id header>"

curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {}}'
```

Expected response: two tools — `get_order_status` and `get_shipping_eta`.

---

## Sample Tool Calls

### `get_order_status` — ORD-1001

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "get_order_status",
      "arguments": {"orderId": "ORD-1001"}
    }
  }'
```

**Expected response:**

```json
{
  "result": {
    "content": [{"type": "text", "text": "{\"orderId\":\"ORD-1001\",\"status\":\"PROCESSING\",\"lastUpdated\":\"2026-08-15T08:30:00Z\",\"message\":\"The order is currently being processed.\"}"}],
    "isError": false
  }
}
```

### `get_shipping_eta` — ORD-1002

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 4,
    "method": "tools/call",
    "params": {
      "name": "get_shipping_eta",
      "arguments": {"orderId": "ORD-1002"}
    }
  }'
```

**Expected response:**

```json
{
  "result": {
    "content": [{"type": "text", "text": "{\"orderId\":\"ORD-1002\",\"shippingStatus\":\"SHIPPED\",\"carrier\":\"Blue Dart\",\"trackingNumber\":\"BD784512963IN\",\"estimatedDeliveryDate\":\"2026-08-18\"}"}],
    "isError": false
  }
}
```

---

## Expected `ORDER_NOT_FOUND` Error

```bash
curl -s -X POST http://localhost:8081/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "Mcp-Session-Id: $SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 5,
    "method": "tools/call",
    "params": {
      "name": "get_order_status",
      "arguments": {"orderId": "ORD-9999"}
    }
  }'
```

**Expected response (isError: true):**

```json
{
  "result": {
    "content": [{"type": "text", "text": "{\"code\":\"ORDER_NOT_FOUND\",\"message\":\"No order was found for the supplied orderId.\",\"orderId\":\"ORD-9999\"}"}],
    "isError": true
  }
}
```

---

## Mock Order Data

Mock data lives in `src/main/resources/mock-orders.json`. Edit it to add or modify orders
without changing any flow XML. Supported order IDs: `ORD-1001`, `ORD-1002`, `ORD-1003`.

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

> **Note:** Do not publish the application JAR as the MCP contract. Provide the Exchange JSON
> descriptor and the MCP specification file. Confirm your organization ID and environment name
> before publishing.

---

## CloudHub 2.0 Deployment Prerequisites

1. A CloudHub 2.0 target (Private Space or Shared Space) in your Anypoint organization.
2. The application JAR: `target/order-management-mcp-server-1.0.0-SNAPSHOT-mule-application.jar`
3. Anypoint CLI v4 with the DX plugin installed.
4. An `application.properties` override or environment-specific properties for `http.host`, `http.port`, `mcp.basePath`.

Deploy command (adjust parameters for your target):

```bash
anypoint-cli-v4 dx mule deploy order-management-mcp-server \
  --environment <ENVIRONMENT_NAME> \
  --runtime 4.11.6 \
  --file target/order-management-mcp-server-1.0.0-SNAPSHOT-mule-application.jar
```

---

## Project Structure

```
src/main/mule/
  global-config.xml        — HTTP listener + MCP server config
  mcp-server.xml           — Session initialization flow
  order-tools.xml          — Tool flows + shared lookup subflow
  health-endpoint.xml      — GET /health

src/main/resources/
  application.properties   — http.host, http.port, mcp.basePath
  mock-orders.json         — Mock order data (edit here)
  log4j2.xml               — Logging configuration

src/test/munit/
  order-management-test-suite.xml  — 12 MUnit tests
