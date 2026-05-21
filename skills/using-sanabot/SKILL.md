---
name: using-sanabot
description: Foundational guidance for using the Sanabot MCP gateway (mcp.sana.bot) — connection check, the tool catalogue, scopes, error handling. Load this first whenever the user mentions Sanafi, their Sanafi wallet, Sanafi card, or "sanabot". Use the more specific sanafi-portfolio and sanafi-card skills for domain workflows.
license: MIT
metadata:
  author: sanafi-onchain
  version: "0.1.0"
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

You can query the user's Sanafi account state — wallet, prices, transactions, card — through a hosted MCP server at `https://mcp.sana.bot/mcp`. This skill explains **what's available right now, when to use which tool, and how to handle the common failure modes.**

The current Sanabot tool set covers the read flows below. If the user asks for something not in the tool catalogue (for example, executing a swap or sending a token), explain that the flow isn't available through Sanabot at the moment and direct them to the Sanafi app for now.

## How the user wires it up

The user installs this skill pack via their agent's plugin marketplace (Claude Code, Cursor, etc.) and exports one secret in their shell:

```bash
export SANABOT_API_KEY=sana_live_...
```

They generate this key at `https://sana.bot/gateway/app/api-keys` with the default `read:all` scope. If `$SANABOT_API_KEY` is unset, every Sanabot call returns 401.

Depending on the agent, the user **also** registers the MCP server explicitly (Claude Code uses `claude mcp add ... --transport http https://mcp.sana.bot/mcp --header "Authorization: Bearer $SANABOT_API_KEY"`; Cursor/Windsurf use their MCP settings UI; Gemini/Copilot/Kiro/OpenCode edit a settings JSON file). Plugin install alone does NOT register URL-based MCP servers on every agent — the per-platform install in the public README has the exact step for each one. If the user reports tools aren't available, the agent should suggest they confirm both pieces: skill pack installed (loaded into the session) **and** MCP server registered (visible via the agent's own MCP-list command if it has one).

## Tools available

The MCP server currently exposes these nine tools. **All names are stable; treat them as the public contract.**

| Tool | Returns | Required scope |
| --- | --- | --- |
| `get_net_worth` | Total USD net worth + per-asset breakdown | `read:wallet` |
| `get_holdings` | Non-zero token positions (symbol, amount, USD value, 24h change) | `read:wallet` |
| `get_account` | `{ wallet_address, email, chain }` — only the agent-safe projection | `read:wallet` |
| `get_supported_tokens` | Catalog of swap/transfer-eligible tokens with mint addresses | `read:wallet` |
| `get_price` | Price + 24h change for one token (with `symbol`) or the whole catalog (without) | `read:prices` |
| `get_notifications` | Recent notification feed (deposits, withdrawals, card alerts) | `read:notifications` |
| `get_transaction_history` | Paginated tx history with optional `context: 'crypto' \| 'card'` filter | `read:transactions` |
| `get_card` | Card metadata: `id`, `type`, `status`, `last4`, expiration | `read:card` |
| `get_card_balance` | Spending power: credit limit, pending/posted charges, balance due, available USD | `read:card` |

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

For multi-step questions (e.g. "summarise my portfolio with prices"), combine tools:
1. `get_holdings` for positions
2. For each unfamiliar symbol, `get_price` to confirm USD value
3. `get_net_worth` for the rolled-up total

## Error handling

The server returns errors as MCP tool-call results with `isError: true` and a short message. **Surface the message to the user as-is — don't invent fixes.** Map the common cases:

| Error | What it means | What to tell the user |
| --- | --- | --- |
| `401 ... INVALID_API_KEY` | `$SANABOT_API_KEY` missing, expired, or revoked | "Your Sanafi API key isn't recognized. Generate a fresh one at https://sana.bot/gateway/app/api-keys." |
| `403 ... INSUFFICIENT_SCOPE` | Key doesn't carry the scope this tool needs | "Your API key doesn't have the right permission. Re-generate with `read:all` or add `<the scope from the error>`." |
| `429` | Rate limit hit | "Hitting Sanafi's rate limit — give it a moment, then try again." |
| `502/503` | Gateway upstream issue | "Sanafi is having trouble responding. Check https://status.sana.bot." |
| `INVALID_IDEMPOTENCY_KEY` | (Internal — not your concern for read tools.) | (No user-facing case.) |

## Privacy and safety boundaries

These are **non-negotiable** and your responses must respect them:

- **Never show `$SANABOT_API_KEY` in any output.** Treat it like a password — it's only used by the MCP transport.
- **Card PAN, CVV, and cardholder PII are never returned.** If the user asks for full card details, explain that `get_card` and `get_card_balance` are deliberately scoped to last-4 + balance only. Point them at the Sanafi app for the full card.
- **If the user asks for a flow not in the tool catalogue above** (swap, send, change settings, etc.), don't try to invent it. Tell them the flow isn't available through Sanabot right now and point them at the Sanafi app to complete it.
- **Don't fabricate values.** If a tool fails or returns no data, say so — never paper over with made-up numbers.

## How to verify the user's setup is working

If the user reports Sanabot "isn't working" or you're unsure whether the MCP server is connected, run `get_account` first. It needs the lowest-effort scope (`read:wallet`, the default) and tells you immediately whether:
- The MCP server is reachable (tool returned at all)
- The API key is valid (no 401)
- The scope chain is healthy (no 403)

If `get_account` returns the user's wallet address + email, the setup is good. If it fails, work through the error table above before trying other tools.
