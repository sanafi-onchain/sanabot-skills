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

## `card_deposit`

- **Scope:** `write:card_deposit`
- **Prerequisite:** Agent signing must be enabled on the user's wallet (one-time setup in the Sanafi dashboard). Phase 1 supports USDC as the deposit token only.
- **When to use:** User wants to top up, fund, or deposit to their Sanafi card balance.
- **When NOT to use:** User wants to withdraw from their card — that requires the Sanafi app.
- **Input:**
  - `amount: string` — amount in USDC as a decimal string (strictly positive; values ≤ 0 are rejected at validation).
  - `idempotency_key?: string` — optional UUID; auto-generated if omitted. Reuse the same key to safely retry after a network error.
- **Returns:** After the MCP gateway unwraps the `{ success, data }` envelope, the agent receives:

  ```jsonc
  {
    "status": "submitted",
    "signature": "<base58 transaction signature>"
  }
  ```

  Surface the confirmation to the user in plain language — e.g. "Your deposit of X USDC was submitted. Transaction: `<signature>`."

- **Common errors:** The gateway surfaces `error.message` from the upstream response body. These are the exact strings the agent will see:

  | HTTP status | Message | What to tell the user |
  | --- | --- | --- |
  | `401` | `Missing API key context` | "Your API key isn't recognized — regenerate at https://sana.bot/gateway/app/api-keys." |
  | `403` | `Agent signing not enabled for this user` | "Enable agent signing in the Sanafi dashboard first, then try again." |
  | `403` | `Transaction exceeds per-transaction cap` | "This deposit exceeds your agent's per-transaction spending cap. Raise the cap in the dashboard." |
  | `403` | `Transaction exceeds daily cap` | "This deposit exceeds your agent's 24-hour spending cap. Wait for the window to reset or raise the cap in the dashboard." |
  | `404` | `Card deposit address unavailable` | "Your card deposit address couldn't be resolved. Check that your card is active in the Sanafi app." |
  | `502` | `Signing service unavailable` | "Sanafi's signing service hit an error. Retry with the same idempotency key." |

---

## `wallet_swap`

- **Scope:** `write:swap`
- **Prerequisite:** Agent signing must be enabled on the user's wallet (one-time setup in the Sanafi dashboard). Input must be a token Sanafi has a USD price for; the swap will reject inputs without one so per-tx caps remain enforceable.
- **When to use:** User wants to swap, exchange, trade, convert, rebalance, or DCA into a token.
- **When NOT to use:**
  - User wants to send tokens to an external wallet — not available through Sanabot.
  - User wants to swap a token Sanafi doesn't price — explain that we can't enforce per-tx USD caps for unsupported tokens, so the swap is rejected.
  - Requested `output_mint` is not on the agent's allowlist (default: Sanafi's active supported-tokens catalogue; user can override per-agent in the dashboard).
- **Input:**
  - `input_mint: string` — Any token Sanafi has a USD price for. The swap rejects tokens with no price so the per-tx USD cap can be enforced.
  - `output_mint: string` — mint address of the desired output token. Must be on the agent's allowlist.
  - `amount_in: string` — amount of the input token to swap as a decimal string (strictly positive; values ≤ 0 are rejected at validation).
  - `idempotency_key?: string` — optional UUID; auto-generated if omitted. Reuse the same key to safely retry after a network error.
  - `swap_token_allowlist` semantics: absent or `null` → use Sanafi's active supported-tokens catalogue as the default allowlist. Non-empty array → that array overrides the default. Empty array `[]` → rejected at validation with `400` (pass `null` to explicitly reset to the catalogue default).
- **Returns:** After the MCP gateway unwraps the `{ success, data }` envelope, the agent receives:

  ```jsonc
  {
    "status": "submitted",
    "signature": "<base58 transaction signature>",
    "in_amount": "<decimal string, e.g. \"50\">",
    "out_amount": "<decimal string | null>"
  }
  ```

  `out_amount` is `null` in Phase 1 — the swap is confirmed on-chain but the received amount is not yet resolved at return time. Surface what you have in plain language, e.g. "Your swap of X USDC was submitted. Transaction: `<signature>`." Treat `out_amount` as a best-effort bonus when non-null.

- **Common errors:** The gateway surfaces `error.message` from the upstream response body. These are the exact strings the agent will see:

  | HTTP status | Message | What to tell the user |
  | --- | --- | --- |
  | `401` | `Missing API key context` | "Your API key isn't recognized — regenerate at https://sana.bot/gateway/app/api-keys." |
  | `403` | `Agent signing not enabled for this user` | "Enable agent signing in the Sanafi dashboard first, then try again." |
  | `400` | `Input token not supported for agent swap (USD price unavailable)` | "That token isn't on Sanafi's supported list — try a token Sanafi knows the price of." |
  | `403` | `Output token not on agent allowlist` | "That output token isn't on your agent's allowlist. Add it in the dashboard under this key's settings." |
  | `403` | `Transaction exceeds per-transaction cap` | "This swap exceeds your agent's per-transaction spending cap. Raise the cap in the dashboard." |
  | `403` | `Transaction exceeds daily cap` | "This swap exceeds your agent's 24-hour spending cap. Wait for the window to reset or raise the cap in the dashboard." |
  | `502` | `Signing service unavailable` | "Sanafi's signing service hit an error. Retry with the same idempotency key." |

---

## Tips for prompting

- **Ask in natural language.** "What did I spend on my card this month?" picks the right skill automatically.
- **Don't pass the API key in your prompt.** The agent already has it via the MCP config.
- **If a skill isn't available yet,** the agent will explain that and point you to the Sanafi app for the flow.
