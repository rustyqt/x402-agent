# x402 Seller Development Guide

Engineering notes on building, deploying, and publishing an x402 service seller.

## Architecture

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────┐
│   Buyer /   │────▶│  Your Service    │────▶│ Facilitator  │
│   Agent     │     │  (FastAPI/MCP)   │     │ (verify +    │
│             │◀────│  + x402 middleware│◀────│  settle)     │
└─────────────┘     └──────────────────┘     └─────────────┘
                            │                       │
                            │  Bazaar extension      │  On-chain
                            │  metadata echoed       │  settlement
                            ▼                       ▼
                    ┌──────────────────┐     ┌─────────────┐
                    │ Discovery Catalog │     │ Blockchain   │
                    │ /discovery/       │     │ (Base/Solana)│
                    │  resources        │     └─────────────┘
                    └──────────────────┘
```

## Step 1: Choose Your Service Type

### HTTP API (FastAPI)

Best for: REST endpoints, browser-accessible services, standard API monetization.

```python
from fastapi import FastAPI
from x402.http import FacilitatorConfig, HTTPFacilitatorClient, PaymentOption
from x402.http.middleware.fastapi import PaymentMiddlewareASGI
from x402.http.types import RouteConfig
from x402.mechanisms.evm.exact import ExactEvmServerScheme
from x402.server import x402ResourceServer

app = FastAPI()
facilitator = HTTPFacilitatorClient(FacilitatorConfig(url="https://x402.org/facilitator"))
server = x402ResourceServer(facilitator)
server.register("eip155:84532", ExactEvmServerScheme())

routes = {
    "GET /my-service": RouteConfig(
        accepts=[
            PaymentOption(scheme="exact", pay_to="0xMyWallet", price="$0.001", network="eip155:84532"),
        ],
        mime_type="application/json",
        description="My service description",
    ),
}
app.add_middleware(PaymentMiddlewareASGI, routes=routes, server=server)

@app.get("/my-service")
async def my_service():
    return {"result": "paid content"}
```

### MCP Tools (FastMCP)

Best for: AI-agent-native interfaces, tool-calling patterns.

```python
from mcp.server.fastmcp import FastMCP
from x402.mcp import create_payment_wrapper_sync, wrap_fastmcp_tool_sync, SyncPaymentWrapperConfig
from x402.schemas import ResourceConfig
from x402.server import x402ResourceServerSync

mcp = FastMCP("my-agent", host="0.0.0.0", port=4022)

resource_server = x402ResourceServerSync(facilitator)
resource_server.register("eip155:84532", ExactEvmServerScheme())
resource_server.initialize()

accepts = resource_server.build_payment_requirements(
    ResourceConfig(scheme="exact", network="eip155:84532", pay_to="0xMyWallet", price="$0.001")
)
paid = create_payment_wrapper_sync(resource_server, SyncPaymentWrapperConfig(accepts=accepts))

@mcp.tool()
def my_tool(query: str, ctx: Context) -> CallToolResult:
    """My tool. Costs $0.001."""
    handler = wrap_fastmcp_tool_sync(
        paid,
        lambda args, _: MCPToolResult(content=[{"type": "text", "text": f"Result: {args['query']}"}]),
        tool_name="my_tool",
    )
    return handler({"query": query}, ctx)

mcp.run(transport="sse")
```

### Flask (Sync)

Best for: Simple sync services, smaller projects.

```python
from flask import Flask, jsonify
from x402.http import FacilitatorConfig, HTTPFacilitatorClientSync, PaymentOption
from x402.http.middleware.flask import payment_middleware
from x402.http.types import RouteConfig
from x402.mechanisms.evm.exact import ExactEvmServerScheme
from x402.server import x402ResourceServerSync

app = Flask(__name__)
facilitator = HTTPFacilitatorClientSync(FacilitatorConfig(url="https://x402.org/facilitator"))
server = x402ResourceServerSync(facilitator)
server.register("eip155:84532", ExactEvmServerScheme())

routes = { "GET /service": RouteConfig(accepts=[...]) }
payment_middleware(app, routes=routes, server=server)

@app.route("/service")
def service():
    return jsonify({"result": "paid content"})
```

## Step 2: Configure Pricing

### Static price (default USDC)

```python
PaymentOption(scheme="exact", pay_to="0x...", price="$0.001", network="eip155:84532")
```

### Explicit token amount

```python
from x402.schemas import AssetAmount

PaymentOption(
    scheme="exact",
    pay_to="0x...",
    price=AssetAmount(
        amount="10000",  # 6 decimals = $0.01 USDC
        asset="0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
        extra={"name": "USDC", "version": "2"},
    ),
    network="eip155:8453",
)
```

### Dynamic pricing

```python
def get_price(context):
    tier = context.adapter.get_query_param("tier")
    return "$0.005" if tier == "premium" else "$0.001"

PaymentOption(scheme="exact", pay_to="0x...", price=get_price, network="eip155:8453")
```

### Multi-network (accept EVM + Solana)

```python
routes = {
    "GET /service": RouteConfig(
        accepts=[
            PaymentOption(scheme="exact", pay_to="0xEvmWallet", price="$0.001", network="eip155:84532"),
            PaymentOption(scheme="exact", pay_to="SolanaWallet", price="$0.001", network="solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1"),
        ],
    ),
}
# Register both schemes
server.register("eip155:84532", ExactEvmServerScheme())
server.register("solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1", ExactSvmServerScheme())
```

## Step 3: Add Observability (Hooks)

```python
async def after_settle(ctx):
    print(f"Payment settled: payer={ctx.result.payer}, tx={ctx.result.transaction}")

async def settle_failure(ctx):
    print(f"Settlement failed: {ctx.error}")

server.on_after_settle(after_settle)
server.on_settle_failure(settle_failure)
```

All available hooks:
- `on_before_verify` / `on_after_verify` / `on_verify_failure`
- `on_before_settle` / `on_after_settle` / `on_settle_failure`

## Step 4: Make It Discoverable (Bazaar)

```python
from x402.extensions.bazaar import OutputConfig, bazaar_resource_server_extension, declare_discovery_extension

server.register_extension(bazaar_resource_server_extension)

routes = {
    "GET /weather": RouteConfig(
        accepts=[...],
        description="Get weather data for any city",
        extensions={
            **declare_discovery_extension(
                input={"city": "San Francisco"},
                input_schema={
                    "properties": {"city": {"type": "string", "description": "City name"}},
                    "required": ["city"],
                },
                output=OutputConfig(
                    example={"weather": "sunny", "temperature": 70},
                    schema={
                        "properties": {
                            "weather": {"type": "string"},
                            "temperature": {"type": "number"},
                        },
                    },
                ),
            ),
        },
    ),
}
```

For dynamic routes:
```python
"GET /weather/:city": RouteConfig(
    accepts=[...],
    extensions=declare_discovery_extension(
        path_params_schema={
            "properties": {"city": {"type": "string", "description": "City name slug"}},
            "required": ["city"],
        },
        output=OutputConfig(example={"city": "san-francisco", "weather": "foggy"}),
    ),
)
```

## Step 5: Add Idempotency (Optional)

```python
from x402.extensions.payment_identifier import (
    PAYMENT_IDENTIFIER, declare_payment_identifier_extension, extract_payment_identifier,
)

routes = {
    "GET /service": RouteConfig(
        accepts=[...],
        extensions={ PAYMENT_IDENTIFIER: declare_payment_identifier_extension(required=False) },
    ),
}

idempotency_cache = {}

async def after_settle(ctx):
    payment_id = extract_payment_identifier(ctx.payment_payload)
    if payment_id:
        idempotency_cache[payment_id] = {"timestamp": time.time(), "response": {...}}

server.on_after_settle(after_settle)
```

## Step 6: Test on Testnet

```bash
# Start server
uvicorn main:app --host 0.0.0.0 --port 4021

# Verify 402 response
curl -i http://localhost:4021/my-service
# Should return HTTP 402 with PAYMENT-REQUIRED header

# Test with x402 buyer client (separate script)
python buyer_test.py
```

## Step 7: Deploy

### Hosting options

| Platform | Command | Notes |
|----------|---------|-------|
| Local | `uvicorn main:app --port 4021` | Development |
| Docker | `docker build -t my-service . && docker run -p 4021:4021 my-service` | Any cloud |
| Railway | `railway up` | Auto-deploy from git |
| Render | Push to git, configure in dashboard | Free tier available |
| VPS | `ssh + systemd service` | Full control |

### Environment variables

```
EVM_ADDRESS=0xYourWallet
SVM_ADDRESS=YourSolanaWallet          # optional
FACILITATOR_URL=https://x402.org/facilitator
PORT=4021
```

## Step 8: Go to Mainnet

Three changes:

```python
# 1. Facilitator
facilitator = HTTPFacilitatorClient(
    FacilitatorConfig(url="https://api.cdp.coinbase.com/platform/v2/x402")
)

# 2. Network
network = "eip155:8453"  # was eip155:84532

# 3. Wallet
pay_to = "0xYourMainnetWallet"  # real address
```

## Step 9: Get Listed on Ecosystem Page (Optional)

Submit a PR to `x402-foundation/x402` adding:

```
typescript/site/app/ecosystem/partners-data/your-service/metadata.json
```

```json
{
  "name": "Your Service",
  "description": "What your service does",
  "category": "Services/Endpoints",
  "websiteUrl": "https://your-service.com",
  "logoUrl": "/ecosystem/your-service/logo.png"
}
```

## Reference: Example Files in x402 Repo

| Pattern | Path |
|---------|------|
| FastAPI server | `x402/examples/python/servers/fastapi/main.py` |
| Flask server | `x402/examples/python/servers/flask/main.py` |
| MCP server (simple) | `x402/examples/python/servers/mcp/simple.py` |
| MCP server (advanced) | `x402/examples/python/servers/mcp/advanced.py` |
| Custom middleware | `x402/examples/python/servers/custom/main.py` |
| Dynamic pricing | `x402/examples/python/servers/advanced/dynamic_price.py` |
| Dynamic pay-to | `x402/examples/python/servers/advanced/dynamic_pay_to.py` |
| Bazaar discovery | `x402/examples/python/servers/advanced/bazaar.py` |
| Dynamic routes | `x402/examples/python/servers/bazaar/main.py` |
| Payment identifier | `x402/examples/python/servers/payment-identifier/main.py` |
| Browser paywall | `x402/examples/python/servers/advanced/paywall.py` |
| Custom tokens | `x402/examples/python/servers/advanced/custom_token.py` |
| Multi-network | `x402/examples/python/servers/advanced/all_networks.py` |
| Hooks | `x402/examples/python/servers/advanced/hooks.py` |
| Basic facilitator | `x402/examples/python/facilitator/basic/main.py` |
