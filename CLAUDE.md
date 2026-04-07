# x402-agent

## Project Overview

This project builds an AI agent that uses the x402 payment protocol to programmatically pay for and access paid APIs. The x402 submodule at `x402/` contains the full protocol SDK.

## x402 Protocol

x402 activates the HTTP `402 Payment Required` status code for programmatic, on-chain payments. Zero fees, ~1s settlement.

### Core Flow

1. Client requests a resource
2. Server returns `402` with `PAYMENT-REQUIRED` header (base64 JSON)
3. Client signs a payment payload with their crypto wallet
4. Client retries with `PAYMENT-SIGNATURE` header
5. Server verifies + settles (directly or via facilitator)
6. Server returns `200` with `PAYMENT-RESPONSE` header

### Roles

- **Client (Buyer)** - Signs payment payloads, accesses resources
- **Server (Seller)** - Defines payment requirements, verifies/settles
- **Facilitator** - Optional service handling on-chain verification + settlement

### Networks (CAIP-2 format)

- EVM: `eip155:<chainId>` (Base `8453`, Base Sepolia `84532`, Polygon `137`, Avalanche `43114`)
- Solana: `solana:<genesisHash>` (mainnet `5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`, devnet `EtWTRABZaYq6iMfeYKouRu166VU2xqa1`)
- Tokens: any ERC-20 (via EIP-3009 or Permit2), any SPL/Token-2022

### Testnet facilitator

`https://x402.org/facilitator` — supports Base Sepolia + Solana Devnet, no setup required.

## Python SDK (v2)

Installed in editable mode from `x402/python/x402/`. Python 3.10+, uses pydantic.

### Key imports

```python
# Client (buyer)
from x402 import x402Client, x402ClientSync
from x402.http import x402HTTPClient, x402HTTPClientSync
from x402.http.clients import x402HttpxClient, x402_requests
from x402.mechanisms.evm import EthAccountSigner
from x402.mechanisms.evm.exact.register import register_exact_evm_client
from x402.mechanisms.svm import KeypairSigner
from x402.mechanisms.svm.exact.register import register_exact_svm_client

# Server (seller)
from x402.server import x402ResourceServer, x402ResourceServerSync
from x402.http import FacilitatorConfig, HTTPFacilitatorClient, HTTPFacilitatorClientSync, PaymentOption
from x402.http.types import RouteConfig
from x402.http.middleware.fastapi import PaymentMiddlewareASGI
from x402.http.middleware.flask import payment_middleware
from x402.mechanisms.evm.exact import ExactEvmServerScheme
from x402.mechanisms.svm.exact import ExactSvmServerScheme

# Extensions
from x402.extensions.payment_identifier import generate_payment_id, extract_payment_identifier
from x402.extensions.eip2612_gas_sponsoring import declare_eip2612_gas_sponsoring_extension
from x402.extensions.erc20_approval_gas_sponsoring import declare_erc20_approval_gas_sponsoring_extension
```

### Buyer pattern (async)

```python
from eth_account import Account
from x402 import x402Client
from x402.http.clients import x402HttpxClient
from x402.mechanisms.evm import EthAccountSigner
from x402.mechanisms.evm.exact.register import register_exact_evm_client

client = x402Client()
account = Account.from_key(os.getenv("EVM_PRIVATE_KEY"))
register_exact_evm_client(client, EthAccountSigner(account))

async with x402HttpxClient(client) as http:
    response = await http.get("https://api.example.com/paid-endpoint")
```

### Seller pattern (FastAPI)

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
    "GET /endpoint": RouteConfig(
        accepts=[PaymentOption(scheme="exact", pay_to="0x...", price="$0.001", network="eip155:84532")],
        mime_type="application/json",
        description="...",
    ),
}
app.add_middleware(PaymentMiddlewareASGI, routes=routes, server=server)
```

### Lifecycle hooks

Server: `on_before_verify`, `on_after_verify`, `on_verify_failure`, `on_before_settle`, `on_after_settle`, `on_settle_failure`
Client: `on_before_payment_creation`, `on_after_payment_creation`, `on_payment_creation_failure`

### Extensions (Python support)

- **Bazaar** - Service discovery via facilitator's `/discovery/resources`
- **Payment Identifier** - Idempotency for safe retries
- **EIP-2612 Gas Sponsoring** - Gasless Permit2 approval
- **ERC-20 Approval Gas Sponsoring** - Universal gasless approval fallback

## Repository Structure

- `x402/` - Git submodule: https://github.com/x402-foundation/x402
- `x402/python/x402/` - Python SDK source (editable install)
- `x402/docs/` - Protocol documentation
- `x402/specs/` - Protocol specifications
- `x402/examples/` - Example implementations
- `web3.py/` - Git submodule: https://github.com/ethereum/web3.py
- `docs/` - Engineering notes and research

## Commands

```bash
# Update submodule
git submodule update --remote --recursive

# Verify SDK
python -c "import x402; print(x402.__version__)"
```

## x402 Ecosystem

### Scale

198 ecosystem partners, 30+ live facilitators, 8 blockchain networks with production support.

### Where x402 Is Used

| Category | Examples | Payment Model |
|----------|----------|---------------|
| Financial Data | Messari, Zerion, Dapplucker | Pay-per-query |
| AI & Inference | Heurist Mesh, Nuwa AI, Einstein AI, SLAM AI | Pay-per-call |
| Web Scraping | Firecrawl, Browserbase | Pay-per-page |
| Infrastructure | Quicknode (RPC), Pinata (IPFS), Vercel | Pay-per-request |
| Social Data | Neynar (Farcaster), Robtex (DNS) | Pay-per-query |
| Commerce | Bitrefill, Stripe integration, 1pay, XPay | Per-transaction |

### Payment Schemes

- **exact** (production) — Fixed price per request. APIs, paywalls, tool calls.
- **upto** (EVM only) — Authorize max, pay actual usage. LLM tokens, bandwidth metering.
- **stream** (planned) — Multi-settlement from single auth. Streaming content, bounties.

### Facilitators

| Facilitator | Networks | Notes |
|-------------|----------|-------|
| `https://x402.org/facilitator` | Base Sepolia, Solana Devnet | Free testnet, no setup |
| `https://api.cdp.coinbase.com/platform/v2/x402` | Base, Polygon, Solana | Coinbase CDP, fee-free, KYT/OFAC |
| `https://facilitator.payai.network` | Various | PayAI, Bazaar discovery |
| Self-hosted | Any EVM | Requires wallet + RPC |

## Service Discovery (Bazaar)

The Bazaar is x402's discovery layer — a machine-readable catalog of paid services.

### Discovering HTTP endpoints

```python
from x402.http import FacilitatorConfig, HTTPFacilitatorClient

facilitator = HTTPFacilitatorClient(FacilitatorConfig(url="https://x402.org/facilitator"))
response = await facilitator.list_resources(type="http")

# Filter by price
affordable = [
    item for item in response.items
    if any(int(req.max_amount_required) < 100000 for req in item.accepts)  # < $0.10
]
```

### Discovery response structure

```
GET /discovery/resources?type=http&limit=20&offset=0
```

Each item contains:
- `resource` — URL of the service
- `type` — `"http"` or `"mcp"`
- `accepts[]` — payment options with `scheme`, `network`, `asset`, `maxAmountRequired`, `payTo`
- `accepts[].outputSchema.input` — method, queryParams, body schema
- `pagination` — `limit`, `offset`, `total`

### Filtering dimensions

- `accepts[].asset` — token address (e.g., USDC `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`)
- `accepts[].network` — blockchain (`eip155:8453`, `solana:5eykt4...`)
- `accepts[].maxAmountRequired` — price in atomic units
- `accepts[].maxTimeoutSeconds` — settlement latency
- `type` — `"http"` or `"mcp"`

### MCP tool discovery

MCP tools are discovered by connecting to an MCP server and calling `tools/list`. Payment flows through `_meta`:
1. Agent calls tool without payment
2. Server returns 402 in `_meta["x402/payment"]`
3. Agent signs payment, retries with `_meta["x402/payment"]` containing `PaymentPayload`
4. Server returns result with `_meta["x402/payment-response"]`

### Registering a service for discovery

Add the `bazaar` extension to route config:

```python
routes = {
    "GET /my-endpoint": RouteConfig(
        accepts=[...],
        description="My service description",
        extensions={
            "bazaar": {
                "discoverable": True,
                "inputSchema": {
                    "queryParams": {
                        "param": {"type": "string", "description": "...", "required": True}
                    }
                },
                "outputSchema": {
                    "type": "object",
                    "properties": {"result": {"type": "string"}}
                }
            }
        },
    ),
}
```

The facilitator catalogs the service on each payment settlement.

## Agent Service Provision

### Providing a paid HTTP API (FastAPI)

See [Seller pattern](#seller-pattern-fastapi) above. The middleware handles 402 responses, verification, and settlement automatically.

### Providing paid MCP tools

```python
from mcp.server.fastmcp import FastMCP
from x402.mcp import create_payment_wrapper_sync, SyncPaymentWrapperConfig
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
    handler = paid(lambda args, _: MCPToolResult(
        content=[{"type": "text", "text": f"Result: {query}"}]
    ))
    return handler({"query": query}, ctx)
```

### Dynamic pricing

```python
def get_price(context):
    tier = context.adapter.get_query_param("tier")
    return "$0.005" if tier == "premium" else "$0.001"

PaymentOption(scheme="exact", pay_to="0x...", price=get_price, network="eip155:8453")
```

### Dual-role agent (consumer + provider)

An agent can simultaneously:
- **Provide** services via FastAPI/MCP with payment middleware (receives payments)
- **Consume** other x402 services via `x402HttpxClient` (makes payments)
- **Discover** services via Bazaar `/discovery/resources`
- **Register** its own services for discovery via the bazaar extension

## Engineering Notes

In-depth research and development guides in `docs/`:

- [`docs/x402-ecosystem-research.md`](docs/x402-ecosystem-research.md) — Where x402 is used (198 partners, networks, facilitators), discovery landscape and maturity (Bazaar limitations, all channels), how agents discover and provide services
- [`docs/x402-seller-development-guide.md`](docs/x402-seller-development-guide.md) — Step-by-step guide to building a seller: service type choice, pricing patterns, hooks, Bazaar registration, idempotency, testnet-to-mainnet migration, deployment options, reference table of all Python examples
- [`docs/x402-product-research.md`](docs/x402-product-research.md) — Viable products for a seller agent: tiered categories (data APIs, content processing, risk gates, AI inference, agent infra), pricing models, ecosystem gaps, real-world pricing examples, agent-as-product patterns

## GitHub

- Repo: https://github.com/rustyqt/x402-agent
- Submodule upstream: https://github.com/x402-foundation/x402
