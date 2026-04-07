# x402 Product Research

Research conducted 2026-04-07. Sources: PROJECT-IDEAS.md, 198 ecosystem partners, x402 specs, server examples.

## Product Categories That Fit x402

### Tier 1: Best Fit (high-frequency, deterministic, sub-$0.25/call)

**Data APIs** — Per-query pricing, instant results, no human involvement
- Token/crypto prices, on-chain analytics, wallet balances
- Real-time sentiment analysis (Reddit, Twitter, news)
- DNS/WHOIS lookups, IP geolocation
- Weather, exchange rates, public datasets
- Typical price: $0.001–$0.05/call

**Content Processing** — Transform input to structured output
- Web scraping → LLM-ready markdown
- Document summarization, entity extraction
- Image description/captioning, OCR
- Code analysis (complexity, security scan)
- Typical price: $0.001–$0.10/call

**Risk & Compliance Gates** — Quick binary or scored assessments
- Wallet screening (sanctions, KYT)
- Smart contract risk scoring
- URL/content safety scanning
- Merchant trust scoring
- Typical price: $0.05–$0.25/call

**AI Inference** — Per-call model access
- Image generation, text-to-speech, speech-to-text
- Embedding generation, semantic search
- Classification, translation
- Fine-tuned domain models (legal, medical, financial)
- Typical price: $0.01–$0.50/call

### Tier 2: Good Fit (medium-frequency, structured output)

**Browser Automation** — Per-session or per-action
- Headless browser sessions for agents
- Automated form filling, screenshots
- Geo-targeted browsing (residential proxies)
- Typical price: $0.01–$0.25/session

**Agent Infrastructure** — Services for other agents
- Health monitoring, execution metrics
- Security scanning of agent inputs/outputs
- Context persistence across model switches
- Agent reputation/certification probes
- Typical price: $0.01–$0.10/call

**Communication Actions** — Per-message pricing
- Send email on behalf of agent
- SMS/voice notifications
- Social media posting (Farcaster, etc.)
- Typical price: $0.01–$0.10/action

### Tier 3: Viable but Niche (lower frequency, higher price)

**Expert Knowledge** — Flat fee per assessment
- Code review ($1–$5/review)
- Legal document generation ($0.50–$5)
- Specialized consulting responses
- Typical price: $0.50–$5.00/call

**Commerce Actions** — Transaction-based
- Price comparison across marketplaces
- Automated purchasing/checkout
- Gift card/prepaid card purchasing
- Typical price: $0.10–$1.00/transaction

---

## Pricing Models Available

| Model | Scheme | How It Works |
|-------|--------|-------------|
| Fixed per-call | `exact` | Every request costs the same (e.g., $0.001) |
| Tiered by quality | `exact` + dynamic pricing | `?tier=premium` → $0.005, standard → $0.001 |
| Auth-conditional | `exact` + SIWX | Authenticated: $0.01, anonymous: $0.10 |
| Usage-based | `upto` (EVM only) | Authorize max, settle actual (per-token, per-byte) |
| Multi-network | `exact` | Same endpoint, different prices per chain |
| Dynamic routing | `exact` + dynamic pay_to | Route payments by region/user to different wallets |

---

## Ecosystem Gaps (Underserved Opportunities)

1. **Agent execution analytics** — No service offers historical cost/success/latency metrics per endpoint
2. **Agent reputation scoring** — No on-chain reliability scores for agents
3. **Cross-agent dispute resolution** — No arbitration service for failed agent transactions
4. **Training data as a service** — No pay-per-example curated dataset access
5. **Real-time competitor intelligence** — No price/action aggregation for agentic trading
6. **API monetization wrapper** — Turn any existing API into an x402 endpoint (Foldset exists but limited)
7. **Legal/contract generation** — No automated NDA/contract service for agent transactions
8. **Supply chain verification** — No product provenance checking at scale

---

## Real Ecosystem Examples (What's Actually Being Sold)

| Service | What It Sells | Price Range |
|---------|--------------|-------------|
| Firecrawl | Web pages → LLM-ready data | ~$0.01/page |
| Browserbase | Headless browser sessions | ~$0.01–0.10/session |
| Messari | Crypto research/data APIs | ~$0.01–0.05/query |
| Augur | Contract risk admission control | ~$0.10/call |
| AI Security Guard | Content safety scanning | ~$0.01–0.05/scan |
| AgentMail | Email send/receive for agents | ~$0.01–0.05/email |
| dTelecom STT | Speech-to-text (99+ languages) | ~$0.01–0.10/minute |
| GenVox | Crypto sentiment from Reddit | ~$0.001–0.01/query |
| Elsa x402 | Portfolio data, swap quotes | ~$0.01/call |
| Freepik | Design assets, AI image gen | ~$0.05–0.50/image |
| Grove API | Tip anyone on internet | ~$0.01+ |

---

## What an AI Agent Specifically Could Provide

Services where the agent IS the product (not just wrapping an API):

1. **LLM-powered analysis** — The agent uses its own reasoning to analyze inputs (code review, document analysis, research synthesis)
2. **Multi-source aggregation** — Agent queries multiple sources, synthesizes a unified answer
3. **Autonomous task execution** — Agent receives a goal, executes multi-step workflows, returns results
4. **Monitoring & alerting** — Agent continuously watches conditions, charges per alert triggered
5. **Negotiation agent** — Agent negotiates with other agents/services on behalf of client
6. **Orchestration layer** — Agent discovers and chains multiple x402 services to fulfill complex requests

---

## From PROJECT-IDEAS.md (Specific Proposals)

- **Unstoppable Agent** — Pays per API call for inference/tooling
- **Wealth Manager Bot** — Per-data-fetch + per-trade fees (streamed)
- **Prediction Market Oracle** — Resolution fees on settlement
- **Rapid KYC/AML** — $0.25 per wallet check
- **Bounty Hunter Agent** — Entry fee + streaming pay-as-you-work
- **Code Review Marketplace** — $5 flat per review
- **Dynamic Endpoint Shopper** — Discovers Bazaar, pays per call, chains results
- **Real-Time Fact Checker** — Journalists pay per source page
- **Pay-As-You-Learn Tutor** — 1 cent per explanation
- **Consultant Agent** — Finds experts, books calls, auto-pays

---

## Crypto-Native Use Cases

x402 payments are already on-chain, so services interacting with blockchain data, DeFi, or crypto infrastructure have a natural fit — the payment rail and the service domain are the same ecosystem.

### Why x402 Is Uniquely Suited

- **Same-chain composability** — Payment and service execution on the same chain. An agent can detect an arb via a $0.001 oracle call, execute a swap, and settle — all on-chain.
- **Wallet as identity** — No accounts, API keys, or sessions. Wallet address = identity across all services.
- **Gasless for user** — Facilitator pays gas. Per-call pricing ($0.001–0.002 typical for oracles/analytics).
- **Non-custodial** — Facilitator cannot move funds, only broadcast signed transactions.
- **On-chain verifiability** — All payments cryptographically signed, settlement immutable.

### Crypto-Native Services Already in Ecosystem

| Service | What It Does | Price |
|---------|-------------|-------|
| DiamondClaws | DeFi yield scoring, protocol risk, gas oracle (17K+ pools, 7 chains) | $0.0005–0.002/req |
| WalletIQ | Wallet intelligence: age, activity, DeFi usage, risk score (5+ chains) | $0.005/lookup |
| Mycelia Signal | Ed25519-signed price oracle (BTC, ETH, SOL, EUR, XAU, VWAP) | $0.001/query |
| Spraay Gateway | 62 endpoints: swaps (Uni V3), bridges (11 chains), tax, KYC | Various |
| PredictOS | Prediction market framework with multi-agent trading bots | Various |
| SLAMai | Smart money intelligence, live on-chain data (Base + Ethereum) | Various |
| Elsa x402 | Portfolio data, token prices, swap quotes, wallet analytics, yields | ~$0.01/call |
| BlackSwan | Real-time risk intelligence for autonomous agents | Various |
| OOBE Protocol | Decentralized agent identity & discovery on Solana | Various |
| AdEx AURA | Portfolio data, DeFi positions, yield strategies, tx payloads | Various |

### Crypto-Native Opportunities (Not Yet Built)

**DeFi Intelligence & Execution**
- Yield opportunity aggregator — query 17K+ pools, construct optimal deployment
- Liquidation bot service — monitor positions, trigger liquidations, get paid from proceeds
- MEV/arbitrage detection — real-time opportunity detection via on-chain data
- Flash loan orchestrator — quote flash loans, execute composite DeFi transactions
- Token launch analyzer — monitor new tokens, analyze contracts, assess rug risk

**Oracles & Data**
- Prediction market oracle — resolve markets by fetching consensus facts, x402 fees on settlement
- Chain risk dashboard — bridge health, validator downtime, MEV trends per chain
- Cross-chain gas oracle — real-time gas prices across all supported networks

**Compliance & Identity**
- Rapid KYC/AML — $0.25 per wallet screen against sanctions lists
- On-chain reputation scoring — agent reliability based on transaction history
- Smart contract audit service — per-contract risk assessment

### The Same-Ecosystem Advantage

Traditional APIs require off-chain payment verification, API key management, fiat settlement delays, and counterparty risk. x402 + on-chain services enable:

1. Atomic verification — payment verified on-chain immediately
2. Wallet identity — no API keys, KYC, or sessions
3. Instant settlement — funds available immediately
4. Composability — payment can trigger downstream on-chain actions in the same flow

---

## Key Takeaway

The best x402 seller products are **high-frequency, deterministic, low-marginal-cost services** where each invocation is quick (<1s), pricing is clear, no manual intervention is needed, and results are reproducible. The ecosystem is mature for automated micro-scale transactions but immature for services requiring human judgment.
