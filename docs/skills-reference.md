# Skills Reference

Every skill the Sanabot gateway exposes, with its input contract, what it returns, and the scope your API key must carry.

Errors come back as MCP `tools/call` results with `isError: true` and a short message — your agent will surface those naturally.

---

## `get_net_worth`

- **Scope:** `read:wallet`
- **Input:** none
- **Returns:** Total net worth in USD plus a breakdown by asset.

## `get_holdings`

- **Scope:** `read:wallet`
- **Input:** none
- **Returns:** Array of non-zero token positions — symbol, amount, USD value, 24h change. Dust positions (amount = 0) are filtered out.

## `get_account`

- **Scope:** `read:wallet`
- **Input:** none
- **Returns:** `{ wallet_address, email, chain }`. Internal identifiers are intentionally not exposed.

## `get_price`

- **Scope:** `read:prices`
- **Input:** `{ symbol?: string }` — token ticker, alphanumeric, max 16 chars. Omit for the full catalog.
- **Returns:** Current USD price + 24h change for one token, or the full catalog when no symbol given.

## `get_supported_tokens`

- **Scope:** `read:wallet`
- **Input:** none
- **Returns:** Catalog of tradeable tokens — symbol, mint address, decimals, gasless eligibility.

## `get_notifications`

- **Scope:** `read:notifications`
- **Input:** none
- **Returns:** Your recent notification feed (deposits, withdrawals, card alerts, etc.).

## `get_transaction_history`

- **Scope:** `read:transactions`
- **Input:**
  - `context?: 'crypto' | 'card'` — filter by source. Omit for both.
  - `page?: number` — 1-based, default `1`.
  - `limit?: number` — items per page, default `20`, max `100`.
- **Returns:** Paginated transactions across crypto wallet activity and card spending.

## `get_card`

- **Scope:** `read:card`
- **Input:** none
- **Returns:** Card metadata — `id`, `type`, `status`, `last4`, `expirationMonth`, `expirationYear`. **PAN, CVV, and cardholder PII are never returned.** Use the Sanafi app to view those.

## `get_card_balance`

- **Scope:** `read:card`
- **Input:** none
- **Returns:** Card spending power — credit limit, pending charges, posted charges, balance due, available USD. PAN/CVV/PII are stripped before the response leaves the Sanafi API.

---

## Tips for prompting

- **Ask in natural language.** "What did I spend on my card this month?" picks the right skill automatically.
- **Don't pass the API key in your prompt.** The agent already has it via the MCP config.
- **If a skill isn't available yet,** the agent will explain that and point you to the Sanafi app for the flow.
