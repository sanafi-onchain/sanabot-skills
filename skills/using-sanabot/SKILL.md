---
name: using-sanabot
description: Foundational guidance for using the Sanabot MCP gateway (mcp.sana.bot) — connection check, the tool catalogue, scopes, error handling. Load this first whenever the user mentions Sanafi, their Sanafi wallet, Sanafi card, or "sanabot". Use the more specific sanafi-portfolio and sanafi-card skills for domain workflows.
license: MIT
metadata:
  author: sanafi-onchain
  version: "0.2.0"
tags:
  - sanafi
  - sanabot
  - solana
  - defi
  - mcp
  - wallet
  - card
---

# Using Sanabot

You can query the user's Sanafi account state — wallet, prices, transactions, card — and execute a limited set of write operations through a hosted MCP server at `https://mcp.sana.bot/mcp`. This skill explains **what's available right now, when to use which tool, and how to handle the common failure modes.**

The current Sanabot tool set covers the read and write flows below. If the user asks for something not in the tool catalogue (for example, sending to an external wallet or withdrawing from their card), explain that the flow isn't available through Sanabot and direct them to the Sanafi app.

## How the user wires it up

The user installs this skill pack via their agent's plugin marketplace (Claude Code, Cursor, etc.) and exports one secret in their shell:

```bash
export SANABOT_API_KEY=sana_live_...
```

They generate this key at `https://sana.bot/gateway/app/api-keys` with the default `read:all` scope. If `$SANABOT_API_KEY` is unset, every Sanabot call returns 401.

Depending on the agent, the user **also** registers the MCP server explicitly (Claude Code uses `claude mcp add ... --transport http https://mcp.sana.bot/mcp --header "Authorization: Bearer $SANABOT_API_KEY"`; Cursor/Windsurf use their MCP settings UI; Gemini/Copilot/Kiro/OpenCode edit a settings JSON file). Plugin install alone does NOT register URL-based MCP servers on every agent — the per-platform install in the public README has the exact step for each one. If the user reports tools aren't available, the agent should suggest they confirm both pieces: skill pack installed (loaded into the session) **and** MCP server registered (visible via the agent's own MCP-list command if it has one).

## Tools available

The MCP server currently exposes eleven tools. **All names are stable; treat them as the public contract.**

| Tool | Returns | Required scope |
| --- | --- | --- |
| `get_net_worth` | Total USD net worth + per-asset breakdown | `read:wallet` |
| `get_holdings` | Non-zero token positions (symbol, amount, USD value, 24h change) | `read:wallet` |
| `get_account` | `{ wallet_address, email, chain }` — only the agent-safe projection | `read:wallet` |
| `get_supported_tokens` | Catalog of swap/transfer-eligible tokens with mint addresses | `read:wallet` |
| `get_price` | Price + 24h change for one token (with `symbol`) or the whole catalog (without) | `read:prices` |
| `get_notifications` | Recent notification feed (deposits, withdrawals, card alerts) | `read:notifications` |
| `get_transaction_history` | Paginated tx history with optional `context: 'crypto' \| 'card'` filter | `read:transactions` |
| `get_card` | Card metadata: `type`, `status`, `last4`, expiration | `read:card` |
| `get_card_balance` | Spending power: credit limit, pending/posted charges, balance due, available USD | `read:card` |
| `card_deposit` | Deposits USDC from the user's Solana wallet to their Sanafi card balance | `write:card_deposit` |
| `wallet_swap` | Executes a token swap via Jupiter; input must be a token Sanafi prices | `write:swap` |

## When to use which tool

- **"What's my net worth?" / "How much am I worth?" / "Show my balance"** → `get_net_worth`
- **"What tokens do I own?" / "Show my holdings" / "What's my portfolio?"** → `get_holdings`
- **"What's my wallet address?" / "What's my email on file?"** → `get_account`
- **"What's the price of SOL?" / "What's USDC worth?"** → `get_price` with the `symbol` argument
- **"What tokens does Sanafi support?" / "Can I swap X?"** → `get_supported_tokens`
- **"Any new activity?" / "Did my deposit arrive?"** → `get_notifications`
- **"Show my last 10 transactions" / "What did I spend on my card?"** → `get_transaction_history` (use `context: 'card'` for card-only)
- **"Show me my card" / "Is my card active?"** → `get_card`
- **"How much can I spend on my card?" / "What's left on my card?"** → `get_card_balance`
- **"Top up my card" / "Fund my card" / "Deposit USDC to my card"** → `card_deposit` (requires agent signing + `write:card_deposit` scope; see write tool prerequisites below)
- **"Swap my USDC for SOL" / "Exchange USDC for JUP" / "Trade into [token]"** → `wallet_swap` (requires agent signing + `write:swap` scope; input must be a token Sanafi prices; see write tool prerequisites below)
- **"Send to an external wallet" / "Withdraw from my card"** → not available through Sanabot; direct the user to the Sanafi app

## Write tools — what the user must set up first

Before `card_deposit` or `wallet_swap` can run, **all three of the following conditions must be true**:

### 1. Agent signing enabled (one-time per user)

The user must enable agent signing for their wallet in the Sanafi dashboard (API keys page at `https://sana.bot/gateway/app/api-keys` → agent management). This delegates a Sanafi-held authorization key as a co-signer on the user's Privy embedded wallet, bounded by an on-chain policy. Without it, all write calls return `403` with the message `Agent signing not enabled for this user`.

### 2. Write scope granted on the API key

Write scopes are **not** included in `read:all` — they must be explicitly enabled per key:

- `write:card_deposit` — required for `card_deposit`
- `write:swap` — required for `wallet_swap`

The user enables these when generating or editing a key in the dashboard. A key missing the scope is rejected with a `403` — scope enforcement lives in the Sanafi API, not the gateway (the gateway forwards the call rather than checking the scope itself).

### 3. Per-agent spending caps not exceeded

Each API key has a **per-transaction USD cap** (default $50) and a **rolling 24-hour daily cap** (default $200). Both are configurable per key in the dashboard. A write call that would exceed the per-transaction cap returns `403` with the message `Transaction exceeds per-transaction cap`; exceeding the daily cap returns `403` with `Transaction exceeds daily cap`. Tell the user to check their key's cap settings or wait for the 24h window to reset.

### Additional constraint: swap output-token allowlist

`wallet_swap` rejects any `output_mint` not on the agent's token allowlist. The default allowlist is Sanafi's active supported-tokens catalogue (refreshed automatically as Sanafi adds support). The user can override it per-agent in the dashboard. Attempts with a non-allowlisted output token return `403` with the message `Output token not on agent allowlist`.

### Input token requirement for swaps

`wallet_swap` requires `input_mint` to be a token Sanafi has a USD price for. Tokens without a price can't have their swap value validated against the per-tx USD cap, so they are rejected with `400` and the message `Input token not supported for agent swap (USD price unavailable)`.

### Idempotency

Both write tools accept an optional `idempotency_key` (UUID). If the agent retries with the same key, the original result is returned — safe to retry on network errors. If omitted, the gateway auto-generates one. Always pass an idempotency key when retrying after a timeout or `502`.

---

For multi-step questions (e.g. "summarise my portfolio with prices"), combine tools:
1. `get_holdings` for positions
2. For each unfamiliar symbol, `get_price` to confirm USD value
3. `get_net_worth` for the rolled-up total

## Error handling

The server returns errors as MCP tool-call results with `isError: true` and a short message. **Surface the message to the user as-is — don't invent fixes.** Map the common cases:

The message the agent sees is taken directly from `error.message` in the upstream JSON body (`{ success: false, error: { message: "..." } }`). These are the actual strings the services emit:

| HTTP status | Message (exact) | What it means | What to tell the user |
| --- | --- | --- | --- |
| `401` | `Missing API key context` | `$SANABOT_API_KEY` missing, expired, or revoked | "Your Sanafi API key isn't recognized. Generate a fresh one at https://sana.bot/gateway/app/api-keys." |
| `403` | *(missing scope — rejected by the Sanafi API)* | Key doesn't carry the scope this tool needs | "Your API key doesn't have the right permission. Re-generate with `read:all` or add the required scope." |
| `403` | `Agent signing not enabled for this user` | Agent signing not set up on this wallet | "Agent signing isn't enabled on your wallet. Go to the API keys page in the Sanafi dashboard and enable agent signing first." |
| `403` | `Transaction exceeds per-transaction cap` | Write call exceeds the per-tx USD cap | "This action exceeds your agent's per-transaction cap. Raise it in the dashboard." |
| `403` | `Transaction exceeds daily cap` | Write call exceeds the rolling 24h USD cap | "This action exceeds your agent's 24-hour spending cap. Wait for the window to reset or raise the cap in the dashboard." |
| `403` | `Output token not on agent allowlist` | Swap output token not on the agent's allowlist | "That token isn't on your agent's swap allowlist. Add it in the dashboard under this key's settings." |
| `400` | `Input token not supported for agent swap (USD price unavailable)` | Swap input has no Sanafi USD price (required for cap enforcement) | "That token isn't on Sanafi's supported list — try a token Sanafi knows the price of." |
| `429` | *(rate limit)* | Rate limit hit | "Hitting Sanafi's rate limit — give it a moment, then try again." |
| `502` | `Signing service unavailable` | Upstream signing service error during a write call | "Sanafi's signing service hit an error. Try again in a moment — use the same idempotency key if you have one. Check https://status.sana.bot if it persists." |
| `502/503` | `Upstream Sanafi API unreachable` or non-JSON body | Gateway can't reach sanafi-api | "Sanafi is having trouble responding. Check https://status.sana.bot." |

## Privacy and safety boundaries

These are **non-negotiable** and your responses must respect them:

- **Never show `$SANABOT_API_KEY` in any output.** Treat it like a password — it's only used by the MCP transport.
- **Card PAN, CVV, and cardholder PII are never returned.** If the user asks for full card details, explain that `get_card` and `get_card_balance` are deliberately scoped to last-4 + balance only. Point them at the Sanafi app for the full card.
- **If the user asks for a flow not in the tool catalogue above** (external send, card withdrawal, change settings, etc.), don't try to invent it. Tell them the flow isn't available through Sanabot right now and point them at the Sanafi app to complete it.
- **Don't fabricate values.** If a tool fails or returns no data, say so — never paper over with made-up numbers.

## How to verify the user's setup is working

If the user reports Sanabot "isn't working" or you're unsure whether the MCP server is connected, run `get_account` first. It needs the lowest-effort scope (`read:wallet`, the default) and tells you immediately whether:
- The MCP server is reachable (tool returned at all)
- The API key is valid (no 401)
- The scope chain is healthy (no 403)

If `get_account` returns the user's wallet address + email, the setup is good. If it fails, work through the error table above before trying other tools.
