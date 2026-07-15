# Endpoint Details — GenTech x402 Gateway

## Health (Free)
```
GET /health
→ 200 { status, service, version, networks, paid_endpoints, free_endpoints }
```

## Pricing (Free)
```
GET /pricing
→ 200 { service, networks, token, tiers, total_endpoints }
```

## OpenAPI Spec (Free)
```
GET /openapi.json
→ 200 OpenAPI 3.1.0 spec with all 16 endpoint definitions
```

## DeFi Yields
```
GET /api/intel/search?q=<product name>
→ 200 { results: [{ title, price, store, url }] }
Cost: $0.005

GET /api/intel/cheapest?q=<product name>
→ 200 { results: [{ title, price, store, url }] }
Cost: $0.005
```

## Token Risk Assessment
```
GET /api/token/risk?address=<token_address>&chain=<chain>
→ 200 { score, risk_level, findings: [{ category, severity, detail }] }
Cost: $0.01
```
Parameters:
- `address` (required): Token contract address
- `chain` (required): Blockchain name (base, solana, avalanche, bnb, okx)

## Wallet Analytics
```
GET /api/wallet/analyze?address=<wallet_address>
→ 200 { score, pnl, portfolio, top_trades, patterns }
Cost: $0.025
```

## Game Intelligence
```
GET /api/games/search?q=<game name>
→ 200 { results: [{ title, platform, price, store, url }] }
Cost: $0.005

GET /api/games/cheapest?q=<game name>
→ 200 { results: [{ title, platform, price, store, url }] }
Cost: $0.005

GET /api/games/news
→ 200 { articles: [{ title, source, date, summary, url }] }
Cost: $0.001

GET /api/games/release?q=<game name>
→ 200 { results: [{ title, platform, release_date }] }
Cost: $0.001
```

## Movie Intelligence
```
GET /api/movies/search?q=<movie name>
→ 200 { results: [{ title, year, price, store, url }] }
Cost: $0.005

GET /api/movies/cheapest?q=<movie name>
→ 200 { results: [{ title, year, price, store, url }] }
Cost: $0.005

GET /api/movies/details?q=<movie name>
→ 200 { title, year, cast, crew, studio, genres, rating }
Cost: $0.001

GET /api/movies/trailers?q=<movie name>
→ 200 { results: [{ title, url, platform }] }
Cost: $0.001
```

## Airdrop Checker
```
GET /api/airdrops/check?address=<wallet_address>
→ 200 { results: [{ protocol, eligible, estimated_value, deadline, tier }] }
Cost: $0.01
```

## NFT Search
```
GET /api/nft/search?q=<collection or asset name>
→ 200 { results: [{ name, collection, floor_price, volume, chain }] }
Cost: $0.005
```

## Shipping Tracker
```
GET /api/shipping/track?number=<tracking_number>&carrier=<carrier>
→ 200 { status, estimated_delivery, events: [{ date, location, description }] }
Cost: $0.005
```

## Agent Scan
```
GET /api/agentscan?query=<agent_name_or_address>
→ 200 { identity, type, revenue, location, contacts, socials }
Cost: $0.10
```
