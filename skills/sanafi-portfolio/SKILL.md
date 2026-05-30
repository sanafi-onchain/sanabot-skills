---
name: sanafi-portfolio
description: Patterns for answering portfolio and wallet questions about a Sanafi user — net worth, holdings, prices, account info, supported tokens, notifications. Load this when the user asks about their tokens, balances, portfolio value, or token prices. Requires the using-sanabot skill for connection/error handling context.
license: MIT
metadata:
  author: sanafi-onchain
  version: "0.1.0"
tags:
  - sanafi
  - sanabot
  - portfolio
  - wallet
  - holdings
  - prices
  - solana
---

# Sanafi Portfolio Workflows

How to answer the most common portfolio-style questions about a Sanafi user using the Sanabot MCP tools. For tool catalogue, scopes, and error handling, see the `using-sanabot` skill — load it alongside this one.

## Quick-reference: question → tool

| User says | First tool | Then maybe |
| --- | --- | --- |
| "What's my net worth?" | `get_net_worth` | — |
| "What do I own?" / "Show me my portfolio" | `get_holdings` | — |
| "What's my [token X] balance?" | `get_holdings` | filter result to symbol |
| "What's the price of SOL?" | `get_price` with `symbol: 'SOL'` | — |
| "What tokens does Sanafi support?" | `get_supported_tokens` | — |
| "What's my wallet address?" / "My email?" | `get_account` | — |
| "Has anything happened recently?" | `get_notifications` | — |
| "Did my deposit arrive?" | `get_notifications` | + `get_holdings` if they want the new balance |
| "Swap SOL for USDC" / "Exchange into JUP" / "Convert / rebalance" | `wallet_swap` (see prerequisites below) | `get_holdings` to confirm post-swap balance |
| "Send tokens to an external wallet" | Not available in Phase 1 — Sanafi app only | — |

## Net worth: one tool, one number

`get_net_worth` returns the rolled-up USD total. Use it when the user wants a single number. If they want the breakdown by asset, prefer `get_holdings` instead — it's strictly more informative.

```jsonc
// Example response shape from get_net_worth
{ "totalUsd": 12345.67, "solUsd": 8900.00, "spendingUsd": 3445.67 }
```

Don't compute net worth manually from `get_holdings` — the BE may include staking, claimable rewards, or other balance sources you can't see.

## Holdings: filter dust at source

`get_holdings` returns **non-zero** positions only. Dust (amount = 0) is filtered server-side. Each entry includes symbol, amount, USD value, 24h change, and metadata. Use it for portfolio breakdowns.

When the user asks "do I own X?", scan the result and answer based on what's there — don't claim a token is missing because it wasn't returned; the user may simply have zero of it.

## Prices: one call for one token, one for the catalog

- `get_price` with `symbol: 'SOL'` → details for that token only.
- `get_price` with no arguments → the entire supported-token catalog. **Heavier response — use this only when the user wants a list or comparison.**

The `symbol` argument is validated as alphanumeric, max 16 characters. Lowercase is fine. Send `"SOL"`, not `"$SOL"` or `"sol-usdc"`.

If a user asks for a token that isn't in the catalog (e.g. exotic memecoin), `get_price` may return null/empty. Suggest `get_supported_tokens` to see what's available.

## Account info: agent-safe projection only

`get_account` returns exactly three fields:

```jsonc
{ "wallet_address": "...", "email": "user@example.com", "chain": "solana" }
```

Internal identifiers (Privy ID, DB id, timestamps) are **deliberately not exposed** — don't ask for them, don't construct them, don't claim they exist. The wallet address is a Solana base-58 string; never claim it's an Ethereum address.

## Supported tokens: useful, low-frequency

`get_supported_tokens` is rarely needed mid-conversation. Reach for it when:
- The user wants to know if Sanafi supports a specific token.
- You're explaining a swap/transfer plan and need the mint address — the catalog is the canonical source.
- A `get_price` call returned nothing for a symbol the user expected.

## Notifications: lightweight activity feed

`get_notifications` returns a list of recent events — deposits, withdrawals, card alerts, etc. Use it to answer "did anything happen?" or "is my deposit through?" Quick check; doesn't replace `get_transaction_history` for full audit trails.

## Combining tools cleanly

**"Summarise my portfolio with current prices."**
1. `get_holdings` for positions.
2. The response already includes per-token USD value and 24h change — usually no extra `get_price` calls needed.
3. Optionally `get_net_worth` if the user wants the rolled-up total.

**"How am I doing today?"**
1. `get_net_worth` for the headline figure.
2. `get_holdings` to spot which positions moved most.
3. Report movers > ±2% as "notable changes today" (use the 24h change field).

**"Did my USDC deposit arrive?"**
1. `get_notifications` to check for the deposit event.
2. `get_holdings` to confirm the USDC balance reflects it.

## Swap: `wallet_swap`

Triggers: user wants to swap, exchange, trade, convert, rebalance, or DCA into a token.

**Prerequisites before calling this tool:**
1. Agent signing must be enabled on the user's wallet (one-time setup at `https://sana.bot/gateway/app/api-keys`). Without it, the call returns `Agent signing not enabled for this user (status 403)`.
2. The API key must carry the `write:swap` scope (not included in `read:all`). A key without it is rejected with a `403` — scope enforcement lives in the Sanafi API, not the gateway.
3. The swap value must not exceed the key's per-transaction cap (default $50) or rolling 24h daily cap (default $200). Exceeding them returns `Transaction exceeds per-transaction cap (status 403)` or `Transaction exceeds daily cap (status 403)`.

**Input and output constraints:**
- `input_mint` must be a token Sanafi has a USD price for. Tokens without a price can't be capped in USD terms, so they are rejected with `Input token not supported for agent swap (USD price unavailable) (status 400)`.
- `output_mint` must be on the agent's token allowlist (defaults to Sanafi's active supported-tokens catalogue). Non-allowlisted output returns `Output token not on agent allowlist (status 403)`. The user can override the allowlist per-agent in the dashboard.

**Input:** `{ input_mint, output_mint, amount_in, idempotency_key? }` — mint addresses (not symbols). Use `get_supported_tokens` to resolve a symbol to a mint address if needed. Pass an `idempotency_key` (UUID) if retrying after a network error to avoid duplicate swaps.

**Flow:**
1. Confirm: which token the user wants to swap FROM (must be a token Sanafi prices), which token they want TO, and the amount.
2. If the output token symbol is ambiguous, call `get_supported_tokens` to resolve the mint address.
3. Check prerequisites: agent signing, scope, amount within cap, output on allowlist.
4. Call `wallet_swap`.
5. Surface the confirmation (transaction signature, `amountIn`, `amountOut`) in plain language.
6. Optionally call `get_holdings` to show the updated balance.

**Error handling.** The agent receives the upstream message with the HTTP status appended, i.e. `<message> (status N)`:
- `Agent signing not enabled for this user (status 403)` → "Enable agent signing in the Sanafi dashboard first."
- a `403` with no agent-signing or cap message → the key is missing the `write:swap` scope: "Add `write:swap` to your API key."
- `Transaction exceeds per-transaction cap (status 403)` / `Transaction exceeds daily cap (status 403)` → "This swap exceeds your agent's spending cap — raise the cap in the dashboard or wait for the 24h window."
- `Output token not on agent allowlist (status 403)` → "That output token isn't on your agent's allowlist. Add it in the dashboard."
- `Input token not supported for agent swap (USD price unavailable) (status 400)` → "That token isn't on Sanafi's supported list — try a token Sanafi knows the price of."
- `Signing service unavailable (status 502)` → "Sanafi's signing service hit an error. Retry with the same idempotency key."

## External send

Sending tokens to an external wallet is **not available in Phase 1**. It's coming in a future release with additional gating. If the user asks, redirect them:

> "I can't send to an external wallet through Sanabot yet. Use the Sanafi app for that for now."

## What not to do

- **Don't compute net worth by summing holdings.** Trust the BE figure.
- **Don't fabricate prices** for symbols not in the catalog.
- **Don't expose the API key** in any response — it's set via env var, never echoed.
- **Don't attempt a swap with a token Sanafi doesn't price.** Explain that tokens without a USD price can't be capped, so the swap is rejected — use `get_supported_tokens` to check what Sanafi supports.
- **Don't attempt to send to an external wallet.** Redirect to the Sanafi app.
