# x402 Payment Protocol — Agent Integration

GenTech's x402 gateway uses the standard x402 payment protocol. Here's how agents should handle the payment flow.

## Standard 402 Flow

```
1. Agent → Gateway: GET /api/<endpoint>?params
2. Gateway → Agent: 402 Payment Required
   Headers:
     X-Payment-Price: 0.001
     X-Payment-Network: base
     X-Payment-Asset: 0x833589346618c73904f65f46f28bfd4aca5e2d03
   Body:
     { "error": "payment_required",
       "accepts": [{ "network": "base", "asset": "0x...",
                     "amount": "1000", "description": "USDC on Base" }] }

3. Agent signs x402 settlement proof using its wallet key
4. Agent → Gateway: GET /api/<endpoint>?params
   Header: X-Payment-Proof: <signed_proof>
5. Gateway → Agent: 200 OK { response_data }
   Header: X-Payment-Response: <settlement_id>
```

## Supported Networks

| Network | Chain ID | Asset | Notes |
|---------|----------|-------|-------|
| Base | eip155:8453 | USDC | Fastest, lowest fees |
| Solana | solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp | USDC | Native Solana SPL |
| Avalanche | eip155:43114 | USDC | EVM-compatible |
| BNB Chain | eip155:56 | USDC | EVM-compatible |
| OKX X Layer | eip155:196 | USDC | EVM-compatible |

## Cost Optimization

- Use the cheapest endpoint that answers the question
- Micro tier ($0.001) endpoints for simple lookups (news, details, trailers)
- Standard tier ($0.005) endpoints for searches and comparisons
- Premium tier ($0.01) for security-sensitive operations
- Pro tier ($0.025) for deep wallet analysis
- Ultra tier ($0.10) only when you need full agent recon

## Important Notes

- All prices are in USD, settled in equivalent USDC
- Multiple networks available — choose the one with lowest fees for the agent's wallet
- Gateway version 6.0.0, supports x402 v2 protocol
- Bazaar-indexed — no additional registration needed beyond the gateway being live
