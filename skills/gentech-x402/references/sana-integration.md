# Sana Bot Integration — Two-Gateway Pattern

## How GenTech + Sana Work Together

Agents use **both** gateways depending on the task:

```
Agent
  ├── mcp.sana.bot/mcp  →  Sana tools (Solana banking, card, wallet)
  └── gentech-x402-gateway... →  GenTech tools (multi-chain intelligence, yields, risk)
```

## When to Use Which Gateway

| Task | Gateway | Why |
|------|---------|-----|
| Check wallet balance | Sana `get_holdings` | Direct Solana wallet access |
| Deposit to card | Sana `card_deposit` | Card is Sana's product |
| Check card spending power | Sana `get_card_balance` | Card-specific |
| Swap tokens | Sana `wallet_swap` | Jupiter integration |
| Token price check | Sana `get_price` | Fast, solana-native |
| **Token risk analysis** | **GenTech `/token/risk`** | **Multi-chain, AI-powered** |
| **Wallet analytics** | **GenTech `/wallet/analyze`** | **Smart money tracking** |
| **DeFi yield comparison** | **GenTech `/intel/search`** | **200+ protocols** |
| **Multi-chain NFT search** | **GenTech `/nft/search`** | **EVM + Solana** |
| **Agent recon** | **GenTech `/agentscan`** | **Identity verification** |

## Auth Patterns

| Gateway | Auth Method | Header |
|---------|-------------|--------|
| Sana MCP | Bearer token (API key) | `Authorization: Bearer sana_live_...` |
| GenTech x402 | x402 signature | `X-Payment-Proof: <signed_proof>` |

## Error Handling

| Error | Likely Cause | Action |
|-------|-------------|--------|
| Sana: 401 | API key expired or invalid | Check creds |
| Sana: 403 | Geo-blocked (US) | Use VPN or alternative gateway |
| GenTech: 402 | No payment proof | Handle x402 flow |
| GenTech: 401 | Signature expired | Re-sign payment |
