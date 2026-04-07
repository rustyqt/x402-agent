# x402 Ecosystem Research

Research conducted 2026-04-07. Source: `x402/docs/`, `x402/specs/`, `x402/examples/`, ecosystem partner data.

## Where x402 Is Used Today

### Scale

- 198 ecosystem partners across 6 categories
- 30+ live facilitators processing real payments
- 8 blockchain networks with production support

### Service Categories

| Category | Examples | Payment Model |
|----------|----------|---------------|
| Financial Data | Messari (crypto intelligence), Zerion (asset tracking), Dapplucker (analytics) | Pay-per-query |
| AI & Inference | Heurist Mesh (decentralized inference), Nuwa AI, Einstein AI, SLAM AI | Pay-per-call |
| Web Scraping | Firecrawl (web-to-LLM data), Browserbase (cloud browser sessions) | Pay-per-page |
| Infrastructure | Quicknode (RPC), Pinata (IPFS), Vercel (edge compute) | Pay-per-request |
| Social Data | Neynar (Farcaster), Robtex (DNS intelligence) | Pay-per-query |
| Commerce | Bitrefill (gift cards), Stripe integration, 1pay, XPay | Per-transaction |

### Payment Schemes

| Scheme | Status | Use Case |
|--------|--------|----------|
| **exact** | Production | Fixed price per request. APIs, paywalls, tool calls. |
| **upto** | Specified (EVM only) | Authorize max, pay actual usage. LLM tokens, bandwidth metering. |
| **stream** | Planned | Multi-settlement from single auth. Streaming content, bounties. |

### Supported Networks

| Network | CAIP-2 ID | Tokens | Status |
|---------|-----------|--------|--------|
| Base | `eip155:8453` | Any ERC-20 | Mainnet |
| Base Sepolia | `eip155:84532` | Any ERC-20 | Testnet |
| Solana | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` | Any SPL | Mainnet |
| Solana Devnet | `solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1` | Any SPL | Testnet |
| Polygon | `eip155:137` | Any ERC-20 | Mainnet |
| Avalanche | `eip155:43114` | Any ERC-20 | Mainnet |
| Aptos | `aptos:1` | Fungible assets | Mainnet (TS only) |
| Stellar | `stellar:pubnet` | Soroban tokens | Mainnet (TS only) |

### Live Facilitators

| Facilitator | URL | Networks | Notes |
|-------------|-----|----------|-------|
| x402.org | `https://x402.org/facilitator` | Base Sepolia, Solana Devnet | Free testnet, no setup |
| Coinbase CDP | `https://api.cdp.coinbase.com/platform/v2/x402` | Base, Polygon, Solana | Fee-free USDC, KYT/OFAC |
| PayAI | `https://facilitator.payai.network` | Various | Bazaar discovery |
| 0x402.ai | Various | Various | Facilitator cloud infra |
| 20+ community | Various | Various | See x402.org/ecosystem |

---

## How Agents Discover Services

### Discovery Landscape

Bazaar is the only built-in discovery mechanism in the x402 protocol, but it's early-stage. The docs describe it as "more like Yahoo search" than the eventual vision of "Google for agentic endpoints."

**Key limitations:**
- Bazaar is an extension, not a core requirement — facilitators opt-in
- Each facilitator runs its own catalog — no unified global index
- Only Coinbase CDP and PayAI are confirmed to expose `/discovery/resources`
- Filtering is basic: by type (`http`/`mcp`), with pagination, no full-text search

**All discovery channels today:**

| Channel | Reach | Maturity |
|---------|-------|----------|
| **Bazaar** (`/discovery/resources`) | Agents querying specific facilitators | Early, fragmented across facilitators |
| **MCP** (`tools/list`) | Agents with direct server connections | Mature but requires knowing the server URL |
| **x402.org/ecosystem** | Human developers browsing | Static listing, 198 partners, manual PR |
| **Word of mouth / docs** | Developers reading examples | How most adoption happens today |

There is no meta-aggregator across facilitators yet. An agent must query each facilitator's Bazaar separately, or know the service URL upfront. The spec is open for anyone to build an aggregator.

**Practical implication:** For real discoverability today, getting listed on the ecosystem page and sharing direct MCP server configs matters as much as Bazaar registration.

### Bazaar (HTTP Endpoint Discovery)

The Bazaar is x402's discovery layer — a machine-readable catalog at each facilitator's `/discovery/resources` endpoint.

**API:**
```
GET /discovery/resources?type=http&limit=20&offset=0
```

**Response structure:**
```json
{
  "x402Version": 2,
  "items": [
    {
      "resource": "https://api.example.com/weather",
      "type": "http",
      "x402Version": 2,
      "lastUpdated": "2025-08-09T01:07:04.005Z",
      "accepts": [
        {
          "scheme": "exact",
          "network": "eip155:8453",
          "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
          "maxAmountRequired": "200",
          "maxTimeoutSeconds": 60,
          "payTo": "0xa2477E...",
          "outputSchema": {
            "input": { "type": "http", "method": "GET", "queryParams": { "city": "San Francisco" } },
            "output": null
          }
        }
      ]
    }
  ],
  "pagination": { "limit": 20, "offset": 0, "total": 150 }
}
```

**Filtering dimensions:**
- `accepts[].asset` — token address (e.g., USDC)
- `accepts[].network` — blockchain (`eip155:8453`)
- `accepts[].maxAmountRequired` — price in atomic units
- `accepts[].maxTimeoutSeconds` — settlement latency
- `type` — `"http"` or `"mcp"`

**Python SDK:**
```python
from x402.http import FacilitatorConfig, HTTPFacilitatorClient

facilitator = HTTPFacilitatorClient(FacilitatorConfig(url="https://x402.org/facilitator"))
response = await facilitator.list_resources(type="http")

affordable = [
    item for item in response.items
    if any(int(req.max_amount_required) < 100000 for req in item.accepts)
]
```

### MCP Tool Discovery

MCP tools are discovered by connecting to an MCP server and calling `tools/list`. Payment flows through `_meta`:

1. Agent calls tool without payment
2. Server returns 402 in `_meta["x402/payment"]`
3. Agent signs payment, retries with `_meta["x402/payment"]` containing `PaymentPayload`
4. Server returns result with `_meta["x402/payment-response"]`

**MCP tool in Bazaar catalog** (unique key = `resource` + `input.tool`):
```json
{
  "type": "mcp",
  "outputSchema": {
    "input": {
      "type": "mcp",
      "tool": "financial_analysis",
      "inputSchema": {
        "type": "object",
        "properties": { "ticker": { "type": "string" } },
        "required": ["ticker"]
      },
      "transport": "streamable-http"
    }
  }
}
```

### How Services Register for Discovery

Services become discoverable automatically by including the `bazaar` extension in their route config. On each payment settlement, the facilitator catalogs the service and returns status in `EXTENSION-RESPONSES` header (`"success"`, `"processing"`, or `"rejected"`).

No submission forms, no approval process — include the extension and process payments.

---

## How Agents Provide Their Own Services

### Python Server Patterns

All examples at `x402/examples/python/servers/`:

| Pattern | File | Use Case |
|---------|------|----------|
| Basic FastAPI | `fastapi/main.py` | Standard paid API |
| Basic Flask | `flask/main.py` | Sync paid API |
| MCP tools | `mcp/simple.py` | AI-agent tool monetization |
| MCP + hooks | `mcp/advanced.py` | Production MCP with logging |
| Dynamic pricing | `advanced/dynamic_price.py` | Tier-based pricing |
| Dynamic pay-to | `advanced/dynamic_pay_to.py` | Route payments by region |
| Bazaar discovery | `advanced/bazaar.py` | Make endpoint discoverable |
| Dynamic routes | `bazaar/main.py` | `/weather/:city` style |
| Idempotency | `payment-identifier/main.py` | Safe retries |
| Browser paywall | `advanced/paywall.py` | Human-facing payment UI |
| Custom middleware | `custom/main.py` | Full manual control |
| Custom tokens | `advanced/custom_token.py` | Non-USDC tokens |

### Development Flow

```
1. Write service logic (no x402 awareness needed)
2. Wrap with payment middleware (3-5 lines)
3. Add Bazaar extension for discovery
4. Test on testnet with x402.org/facilitator
5. Switch 3 values for mainnet (facilitator URL, network ID, wallet)
```

### Service Type Decision

| Type | When to use | Discovery |
|------|-------------|-----------|
| HTTP API (FastAPI) | Standard REST endpoints | Bazaar `/discovery/resources` |
| MCP Tools (FastMCP) | AI-agent-native interface | MCP `tools/list` + Bazaar |
| Both | Maximum reach | Both mechanisms |

### Publishing Layers

**Layer 1 — Hosting** (where code runs): Any HTTP host works. VPS, Vercel, Railway, Render, self-hosted.

**Layer 2 — Facilitator** (who settles payments):
- Dev: `https://x402.org/facilitator` (Base Sepolia + Solana Devnet)
- Prod: `https://api.cdp.coinbase.com/platform/v2/x402` (Base + Polygon + Solana)
- Switching is a one-line URL change.

**Layer 3 — Discovery** (how agents find you):
- **Bazaar** (automatic): Add `bazaar` extension, facilitator catalogs on settlement
- **Ecosystem listing** (manual): PR to `x402-foundation/x402` repo with metadata
- **MCP registry** (for AI agents): Publish MCP server config for Claude Desktop, Cursor, etc.

### Mainnet Migration (3 changes)

```python
url = "https://api.cdp.coinbase.com/platform/v2/x402"  # was x402.org/facilitator
network = "eip155:8453"                                  # was eip155:84532
pay_to = "0xYourMainnetWallet"                           # real mainnet address
```

### Dual-Role Agent

An agent can simultaneously:
- **Provide** services via FastAPI/MCP with payment middleware (receives payments)
- **Consume** other x402 services via `x402HttpxClient` (makes payments)
- **Discover** services via Bazaar `/discovery/resources`
- **Register** its own services for discovery via the bazaar extension

---

## Future Directions

From x402 ROADMAP and PROJECT-IDEAS:

- Unstoppable Agent — pays per API call for inference/tooling
- Wealth Manager Bot — per-data-fetch + per-trade fees (streamed)
- Dynamic Endpoint Shopper — discovers Bazaar, pays per call
- Bounty Hunter Agent — entry fee + streaming pay-as-you-work
- Consultant Agent — finds experts, books calls, auto-pays
- Pay-As-You-Learn Tutor — 1 cent per explanation
- Grants program — up to $3k for projects live on mainnet
