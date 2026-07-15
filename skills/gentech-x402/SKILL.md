---
name: gentech-x402
description: Patterns for using GenTech Labs' x402 payment-gated API services — DeFi yields, token risk assessment, blockchain RPC, wallet analytics, market intelligence, and agent identity. Each endpoint charges $0.001–$0.10 via x402 (USDC on Base, Solana, Avalanche, BNB, or OKX). Load this when the user asks about blockchain data, token prices, DeFi yields, wallet analysis, or agent identity services.
license: MIT
metadata:
  author: GenTech Labs
  version: "1.0.0"
tags:
  - gentech
  - x402
  - defi
  - blockchain
  - rpc
  - token
  - wallet
  - agent
  - payments
  - solana
  - base
  - avalanche
---

# GenTech x402 Services — Agent API Gateway

GenTech Labs operates a multi-chain x402 payment gateway at `https://gentech-x402-gateway.jordanjones0902.workers.dev`. Every endpoint is pay-per-request via x402 protocol — USDC on Base, Solana, Avalanche, BNB Chain, or OKX X Layer. No subscriptions, no API keys needed. Just send a 402-accepting request and settle the micropayment.

**16 paid endpoints / 3 free endpoints** across 9 service categories.

## How x402 Works (One-Time Setup)

Agents discover x402 through the standard 402 flow:

1. Send a request to any paid endpoint → get `402 Payment Required`
2. The 402 response includes payment details (chain, asset, amount, pay-to address)
3. Agent constructs an x402 payload, signs it, and retries with `X-Payment-Proof` header
4. Server verifies the settlement proof and returns `200 OK` with the response data

Reference: [x402.org](https://x402.org) for protocol details.

## Service Catalogue

### DeFi Yields — Lending Rates Across Protocols
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_yields_search` | `GET /api/intel/search?q=` | $0.005 | Search yield rates across 200+ protocols |
| `gentech_yields_best` | `GET /api/intel/cheapest?q=` | $0.005 | Best APY across all protocols |

**Trigger phrases:** "best lending yield", "compare APY", "highest yield on Base", "DeFi rates"

### Token Risk Assessment — AI Security Analysis
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_token_risk` | `GET /api/token/risk?address=&chain=` | $0.01 | AI-powered token risk score |

**Trigger phrases:** "is this token safe", "check token risk", "rug pull check", "token security score"

### Wallet Analytics — Smart Money Tracking
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_wallet_analyze` | `GET /api/wallet/analyze?address=` | $0.025 | AI wallet analytics, P&L, smart money tracking |

**Trigger phrases:** "analyze this wallet", "wallet P&L", "smart money", "top trader"

### Game Intelligence
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_games_search` | `GET /api/games/search?q=` | $0.005 | Multi-store game search with prices |
| `gentech_games_cheapest` | `GET /api/games/cheapest?q=` | $0.005 | Cheapest game price finder |
| `gentech_games_news` | `GET /api/games/news` | $0.001 | Game news and patch notes |
| `gentech_games_release` | `GET /api/games/release?q=` | $0.001 | Release dates and info |

### Movie Intelligence
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_movies_search` | `GET /api/movies/search?q=` | $0.005 | Movie search with price comparison |
| `gentech_movies_cheapest` | `GET /api/movies/cheapest?q=` | $0.005 | Cheapest place to watch |
| `gentech_movies_details` | `GET /api/movies/details?q=` | $0.001 | Cast, crew, studio, genres |
| `gentech_movies_trailers` | `GET /api/movies/trailers?q=` | $0.001 | YouTube trailer links |

### Airdrop Checker
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_airdrop_check` | `GET /api/airdrops/check?address=` | $0.01 | Multi-chain airdrop eligibility |

### NFT Search
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_nft_search` | `GET /api/nft/search?q=` | $0.005 | Multi-chain NFT collection search |

### Shipping Tracker
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_shipping_track` | `GET /api/shipping/track?number=` | $0.005 | Multi-carrier package tracking |

### Agent Reconnaissance
| Tool | Endpoint | Cost | Description |
|------|----------|------|-------------|
| `gentech_agent_scan` | `GET /api/agentscan?query=` | $0.10 | AI-powered agent identity recon |

## Spend-aware usage

- Narrow searches by specific names/addresses for fastest results and lowest cost.
- Cache results when possible — most data changes slowly (minutes to hours).
- Use the cheapest endpoint that answers the question ($0.001 micro tier for simple lookups).
- Chain identification: pass chain name (base, solana, avalanche, bnb, okx) where applicable.

## Pricing tiers

| Tier | Price | Endpoints |
|------|-------|-----------|
| Micro | $0.001 | News, details, trailers, release dates |
| Standard | $0.005 | Search, cheapest, NFT, shipping, yields |
| Premium | $0.01 | Token risk, airdrop check |
| Pro | $0.025 | Wallet analytics |
| Ultra | $0.10 | Agent scan |

## Gateway info

- **Health:** `https://gentech-x402-gateway.jordanjones0902.workers.dev/health` (free)
- **Pricing:** `https://gentech-x402-gateway.jordanjones0902.workers.dev/pricing` (free)
- **OpenAPI:** `https://gentech-x402-gateway.jordanjones0902.workers.dev/openapi.json` (free)
- **Networks:** Base, Solana, Avalanche, BNB Chain, OKX X Layer
- **Token:** USDC
- **Version:** 6.0.0
- **Bazaar indexed:** ✅

## References

See `references/` for:
- `endpoint-details.md` — Full request/response shapes for every endpoint
- `x402-flow.md` — x402 payment protocol details for agent integration
