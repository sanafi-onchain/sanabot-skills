---
name: sanafi-card
description: Patterns for answering questions about the user's Sana card — metadata (status, last4, expiry), spending power (limits, charges, balance due), card transactions, full card number + CVV on explicit request, and guiding a user through end-to-end card registration and usage. Load this when the user asks about their Sana card, what they've spent, how much they can still spend, their full card details, or how to get/use a card. Requires the using-sanabot skill for connection/error handling context.
license: MIT
metadata:
  author: sanafi-onchain
  version: "0.2.0"
tags:
  - sanafi
  - sanabot
  - card
  - spending
  - solana
---

# Sanafi Card Workflows

How to answer Sanafi card questions safely using the Sanabot MCP tools. For the full tool catalogue, scopes, and error handling, see the `using-sanabot` skill — load it alongside this one.

**Privacy rules before anything else:**
- The full card number (PAN) and CVV **are** available to you — but **only via the dedicated `get_card_sensitive` tool, and only when the user explicitly asks** ("what's my card number?", "I need my CVV to pay"). They are NOT included in `get_card` or `get_card_balance`.
- **Never call `get_card_sensitive` proactively** or "just in case", never echo the PAN/CVV into summaries, titles, or anything that persists, and don't repeat them back more than needed to answer. Treat them like a password.
- **Cardholder PII** (name, phone, home address) is still stripped at the boundary and not retrievable through any tool.

## Quick-reference: question → tool

| User says | Tool |
| --- | --- |
| "Show me my card" / "What card do I have?" / "Is my card active?" | `get_card` |
| "What's my full card number?" / "I need my CVV to pay" | `get_card_sensitive` (explicit request only) |
| "How much can I spend on my card?" / "What's my available credit?" | `get_card_balance` |
| "What did I spend this month?" / "Show my card transactions" | `get_transaction_history` with `context: 'card'` |
| "How much do I owe?" / "What's my balance due?" | `get_card_balance` (the `balanceDueUsd` field) |
| "How do I get a card?" / "How do I register?" / "How do I use my card?" | No tool — guide them (see "Guiding card registration & usage") |
| "Top up my card" / "Fund my card" / "Deposit USDC to my card" | `card_deposit` (see prerequisites below) |
| "Withdraw from my card" / "Move money off my card" | Not available through Sanabot — Sana app only |

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

That's the metadata projection: `get_card` itself never includes `cardNumber` or `cvc`. If the user explicitly wants those, use `get_card_sensitive` (below) — don't try to read them from `get_card`. Cardholder name/billing address are never exposed.

If `get_card` returns an empty array, the user doesn't have a Sana card yet — guide them through registration (see "Guiding card registration & usage").

## Full card number + CVV: `get_card_sensitive`

Returns the **full PAN and CVV** of the user's active card. Call this **only on an explicit user request** for their full card details:

```jsonc
{
  "type": "virtual",
  "status": "active",
  "last4": "1234",
  "expirationMonth": 12,
  "expirationYear": 2027,
  "cardNumber": "4111111111111111",
  "cvc": "123"
}
```

Rules:
- **Explicit-request only.** Never call it to "enrich" an answer, never preemptively. "Show my card" → `get_card` (metadata); only "show my full card number / CVV" → `get_card_sensitive`.
- **Handle like a secret.** Give the values once, in your direct reply. Don't put them in summaries, memory, titles, or anything persisted; don't repeat them unprompted.
- Requires the dedicated **`read:card_sensitive`** scope (NOT covered by `read:card`, `read`, or `read:all`) and an **active** card. Returns `null` when there's no active card.
- A `403` means the key lacks `read:card_sensitive` — tell the user to regenerate the key with that scope ticked. Note `get_card` / `get_card_balance` may still work (they only need `read:card`); only the full-credential reveal needs the extra scope.

## Guiding card registration & usage

You **cannot register a card for the user** — KYC and the activation payment require the user themselves in the dashboard. But you can walk them through it. Registration is **fully self-serve on the web** at **https://sana.bot/gateway/app/card** (no mobile app needed).

When `get_card` returns `[]` or `get_card_balance` returns `null`, the user has no card yet — offer to guide them. The end-to-end flow, all on that one page:

1. **Pick your country.** Card availability is country-gated by the issuer (Rain); only eligible countries appear.
2. **Pay the one-time activation fee — $10.** Charged in **USDC** from the user's Sana wallet. It's gasless (no SOL needed) and authorized with an emailed 6-digit code; the exact amount is shown before they approve.
3. **Complete the KYC application.** A short form: name, date of birth, address, a few financial details — submitted to Rain for identity verification.
4. **Verify identity.** An embedded document + selfie check (camera may be needed), with an "open in a new tab" fallback.
5. **Accept the cardholder terms.** Once approved, the virtual card is issued automatically and shows up on the card page.

**Using the card once issued:**
- **Top up:** deposit USDC to raise spending power — you can do this directly with `card_deposit` ("top up my card with 25 USDC"), or the user can send to `cardContract.depositAddress` from `get_card_balance`.
- **Spend:** anywhere Visa is accepted, online or in-app, using the number/expiry/CVV (`get_card_sensitive` on request).
- **Track:** `get_card_balance` for spending power, `get_transaction_history` with `context: 'card'` for purchases.

Keep guidance concise and link the page; don't try to collect KYC details or the payment through chat.

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
1. Agent signing must be enabled on the user's wallet (one-time setup at `https://sana.bot/gateway/app/api-keys`). Without it, the call returns `Agent signing not enabled for this user (status 403)`.
2. The API key must carry the `write:card_deposit` scope (not included in `read:all`). A key without it is rejected with a `403` — scope enforcement lives in the Sanafi API, not the gateway.
3. The deposit must not exceed the key's per-transaction cap (default $50) or rolling 24h daily cap (default $200). Exceeding them returns `Transaction exceeds per-transaction cap (status 403)` or `Transaction exceeds daily cap (status 403)`.

**Phase 1 restriction:** USDC is the only supported deposit token. Attempts with other tokens will fail.

**Input:** `{ amount, idempotency_key? }` — `amount` is in USDC. Pass an `idempotency_key` (UUID) if retrying after a network error to avoid double-deposits.

**Flow:**
1. Confirm the user wants to deposit a specific USDC amount.
2. Check prerequisites: agent signing enabled, scope present, amount within cap.
3. Call `card_deposit` with the amount (and idempotency key if retrying).
4. Surface the confirmation (transaction signature, resulting card balance) in plain language.

**Error handling.** The agent receives the upstream message with the HTTP status appended, i.e. `<message> (status N)`:
- `Agent signing not enabled for this user (status 403)` → "Enable agent signing in the Sanafi dashboard first."
- a `403` with no agent-signing or cap message → the key is missing the `write:card_deposit` scope: "Add `write:card_deposit` to your API key."
- `Transaction exceeds per-transaction cap (status 403)` / `Transaction exceeds daily cap (status 403)` → "This deposit exceeds your agent's spending cap — raise the cap in the dashboard or wait for the 24h window."
- `Signing service unavailable (status 502)` → "Sanafi's signing service hit an error. Retry with the same idempotency key."

## Card withdrawal

Card withdrawal (card balance → crypto wallet) requires the user's wallet to sign a structured message. Privy's delegated signing keys cannot fulfill this — it's outside what agent signing covers.

If the user asks you to withdraw from their card:

> "Card withdrawal happens in the Sanafi mobile app — I can't do it for you from here."

Don't attempt to construct a workaround or suggest an alternative tool. Just redirect clearly.

## What not to do

- **Don't fetch PAN/CVV unless the user explicitly asks for them.** Use `get_card_sensitive` only on a direct request, hand the values over once, and never persist or repeat them. Cardholder PII (name/phone/address) is still never available — don't promise it.
- **Don't suggest topping up with a token outside the `tokens` whitelist from `get_card_balance`.** Deposits in non-whitelisted tokens may be lost.
- **Don't attempt card withdrawal through Sanabot.** Redirect to the Sanafi app, every time.
- **Don't infer "the card is broken" from a 403.** A 403 on read tools means the user's API key lacks `read:card` scope — recommend re-generating the key with the right scope. A 403 on `card_deposit` may mean a missing write scope or delegation not set up — check the specific error code.
