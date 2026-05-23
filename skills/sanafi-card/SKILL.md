---
name: sanafi-card
description: Patterns for answering questions about the user's Sanafi card — metadata (status, last4, expiry), spending power (limits, charges, balance due), and card transactions. Load this when the user asks about their Sanafi card, what they've spent, or how much they can still spend. Requires the using-sanabot skill for connection/error handling context.
license: MIT
metadata:
  author: sanafi-onchain
  version: "0.1.0"
tags:
  - sanafi
  - sanabot
  - card
  - spending
  - solana
---

# Sanafi Card Workflows

How to answer Sanafi card questions safely using the Sanabot MCP tools. For the full tool catalogue, scopes, and error handling, see the `using-sanabot` skill — load it alongside this one.

**Privacy rule before anything else:** the gateway **strips PAN (full card number), CVV, and cardholder PII** at the boundary. You will never see them, and you should never ask the user to share them with you. If the user wants to view full card details, point them at the Sanafi app.

## Quick-reference: question → tool

| User says | Tool |
| --- | --- |
| "Show me my card" / "What card do I have?" / "Is my card active?" | `get_card` |
| "How much can I spend on my card?" / "What's my available credit?" | `get_card_balance` |
| "What did I spend this month?" / "Show my card transactions" | `get_transaction_history` with `context: 'card'` |
| "How much do I owe?" / "What's my balance due?" | `get_card_balance` (the `balanceDueUsd` field) |
| "Top up my card" / "Fund my card" / "Deposit USDC to my card" | `card_deposit` (see prerequisites below) |
| "Withdraw from my card" / "Move money off my card" | Not available through Sanabot — Sanafi app only |

## Card metadata: `get_card`

Returns one or more cards' agent-safe metadata:

```jsonc
[
  {
    "type": "virtual",
    "status": "active",
    "last4": "1234",
    "expirationMonth": 12,
    "expirationYear": 2027
  }
]
```

That's the complete agent-visible projection. **There's no `cardNumber`, no `cvc`, no cardholder name, no billing address — those are not in the response and not retrievable through any tool.** If the user asks for them, explain:

> "I can only see your card's last 4 digits and metadata. For the full card number or CVV, open the Sanafi app — it's deliberately scoped that way to keep agent integrations safe."

If `get_card` returns an empty array, the user doesn't have a Sanafi card yet — direct them to onboard one in the app.

## Spending power: `get_card_balance`

Returns the balance breakdown plus contract addresses (useful if the user wants to fund the card):

```jsonc
{
  "creditLimitUsd": 500,
  "pendingChargesUsd": 12.34,
  "postedChargesUsd": 56.78,
  "balanceDueUsd": 69.12,
  "spendingPowerUsd": 430.88,
  "cardContract": {
    "proxyAddress": "...",
    "programAddress": "...",
    "depositAddress": "..."
  },
  "tokens": ["USDC"]
}
```

**Field cheat sheet for answering common questions:**
- "How much can I spend?" → `spendingPowerUsd` (credit limit minus pending + posted)
- "What do I owe?" → `balanceDueUsd`
- "What's my credit limit?" → `creditLimitUsd`
- "Pending charges?" → `pendingChargesUsd`
- "Where do I send funds to top up my card?" → `cardContract.depositAddress`, paired with the `tokens` whitelist (only fund with tokens in that list — e.g. USDC)

If `get_card_balance` returns null, the user doesn't have an active card yet.

## Card transactions: `get_transaction_history` with `context: 'card'`

The transaction-history tool covers both crypto and card activity. To restrict to card spending:

```jsonc
// arguments
{ "context": "card", "page": 1, "limit": 20 }
```

`limit` maxes at 100. Default page size is 20 — usually enough for "what did I spend this week?"

For mixed contexts ("what's happened in my account?"), omit `context` and you'll get both crypto and card events interleaved.

## Combining tools

**"How am I doing on my card?"**
1. `get_card` to confirm a card exists + status.
2. `get_card_balance` for spending power.
3. Report `spendingPowerUsd` + `balanceDueUsd`. Mention pending charges if non-zero.

**"Where can I send USDC to top up?"**
1. `get_card_balance` → read `cardContract.depositAddress` and the `tokens` list.
2. Confirm the user wants to send a token in the `tokens` whitelist.
3. Surface the deposit address. **Do not auto-send anything** — funding the card is not a Sanabot flow today; the user does it from their wallet or the Sanafi app.

**"Show me what I spent on coffee last week."**
1. `get_transaction_history` with `context: 'card'`.
2. Filter the results client-side by merchant/date — the BE doesn't filter by merchant name.

## Card deposit: `card_deposit`

Triggers: user wants to top up, fund, or deposit to their card balance.

**Prerequisites before calling this tool:**
1. Agent signing must be enabled on the user's wallet (one-time setup at `https://sana.bot/gateway/app/api-keys`). Without it, the call returns `403 DELEGATION_NOT_ENABLED`.
2. The API key must carry the `write:card_deposit` scope (not included in `read:all`). Missing scope returns `403 INSUFFICIENT_SCOPE`.
3. The deposit must not exceed the key's per-transaction cap (default $50) or rolling 24h daily cap (default $200). Exceeding either returns `403 CAP_EXCEEDED`.

**Phase 1 restriction:** USDC is the only supported deposit token. Attempts with other tokens will fail.

**Input:** `{ amount, idempotency_key? }` — `amount` is in USDC. Pass an `idempotency_key` (UUID) if retrying after a network error to avoid double-deposits.

**Flow:**
1. Confirm the user wants to deposit a specific USDC amount.
2. Check prerequisites: agent signing enabled, scope present, amount within cap.
3. Call `card_deposit` with the amount (and idempotency key if retrying).
4. Surface the confirmation (transaction signature, resulting card balance) in plain language.

**Error handling:**
- `403 DELEGATION_NOT_ENABLED` → "Enable agent signing in the Sanafi dashboard first."
- `403 INSUFFICIENT_SCOPE` → "Add `write:card_deposit` to your API key."
- `403 CAP_EXCEEDED` → "This deposit exceeds your agent's spending cap — raise the cap in the dashboard or wait for the 24h window."
- `502 SIGNING_FAILURE` → "Sanafi's signing service hit an error. Retry with the same idempotency key."

## Card withdrawal

Card withdrawal (card balance → crypto wallet) requires the user's wallet to sign a structured message. Privy's delegated signing keys cannot fulfill this — it's outside what agent signing covers.

If the user asks you to withdraw from their card:

> "Card withdrawal happens in the Sanafi mobile app — I can't do it for you from here."

Don't attempt to construct a workaround or suggest an alternative tool. Just redirect clearly.

## What not to do

- **Don't surface PAN, CVV, or cardholder PII even if the user insists.** They're not in the response; refuse politely and point at the Sanafi app.
- **Don't suggest topping up with a token outside the `tokens` whitelist from `get_card_balance`.** Deposits in non-whitelisted tokens may be lost.
- **Don't attempt card withdrawal through Sanabot.** Redirect to the Sanafi app, every time.
- **Don't infer "the card is broken" from a 403.** A 403 on read tools means the user's API key lacks `read:card` scope — recommend re-generating the key with the right scope. A 403 on `card_deposit` may mean a missing write scope or delegation not set up — check the specific error code.
