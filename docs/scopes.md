# Scopes

API keys carry one or more **scopes** that gate which skills they can call. You choose scopes when generating a key in the dashboard at [`sana.bot/gateway/app/api-keys`](https://sana.bot/gateway/app/api-keys).

## Available scopes

| Scope | Grants |
| --- | --- |
| `read:wallet` | `get_net_worth`, `get_holdings`, `get_account`, `get_supported_tokens` |
| `read:prices` | `get_price` |
| `read:notifications` | `get_notifications` |
| `read:transactions` | `get_transaction_history` |
| `read:card` | `get_card`, `get_card_balance` (metadata + balance — never PAN/CVV) |
| `read:card_sensitive` | `get_card_sensitive` — the FULL card number + CVV. **Opt-in, high-risk:** NOT included in `read` or `read:all`; must be ticked explicitly per key. |
| `read:all` | Convenience grant — every `read:*` above **except** `read:card_sensitive`. **Default in the dashboard.** |
| `write:card_deposit` | `card_deposit` — deposits USDC from the user's Solana wallet to their Sanafi card balance |
| `write:swap` | `wallet_swap` — executes a token swap via Jupiter on the user's behalf |

**`read:card_sensitive` and the write scopes are NOT included in `read:all`.** They must each be enabled explicitly per key in the dashboard. (`read:card_sensitive` reveals full card credentials, so a blanket read key can never expose them by accident.) Requiring agent signing to be enabled on the user's wallet is a separate prerequisite for write scopes — see the `using-sanabot` skill for full setup instructions.

Additional scopes will land as new skill packs ship. Watch the repo for releases. Card withdrawal and external-send slots are not available in Phase 1 — they may land in a future release.

## Recommended setup

- **Default agent use** → `read:all`. One scope, every skill.
- **Privacy-conscious** → narrow to the scopes the agent actually needs. An agent that only summarises portfolios can run with just `read:wallet` + `read:prices`.
- **Multiple agents** → issue one key per agent so revoking access on a single platform doesn't disrupt the others.

## Backward compatibility

A legacy `read` scope (issued before this expansion) is treated as equivalent to `read:all` — so it grants every `read:*` skill **except** `read:card_sensitive`, which always requires its own explicit grant. Existing keys keep working without rotation, and none of them silently gain full-card-credential access.

## Revocation

Visit [`sana.bot/gateway/app/api-keys`](https://sana.bot/gateway/app/api-keys), pick the key, click **Revoke**. The next request from that key returns `401` — instantly, no propagation delay.
