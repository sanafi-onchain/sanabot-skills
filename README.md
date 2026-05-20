# Sanabot Skills

AI-agent skills for the [Sanafi](https://sana.bot) super-app on Solana. Install once, and your agent can answer questions about your **wallet, prices, transactions, and card** by routing through the hosted MCP gateway at `https://mcp.sana.bot`.

> No local install of Sanafi infrastructure. Every call uses your **Sanafi API key** and lives a single round trip.

## What's in this repo

```
skills/
  using-sanabot/        ← foundational: connection, tool catalogue, scopes, errors
  sanafi-portfolio/     ← net worth, holdings, prices, account, notifications
  sanafi-card/          ← card metadata, spending power, card transactions
.claude-plugin/         ← Claude Code plugin manifest
.mcp.json               ← MCP server config (loaded by plugins)
docs/
  scopes.md             ← scope vocabulary + revocation
  skills-reference.md   ← input/output per tool the gateway exposes
```

Each `skills/<name>/SKILL.md` is **prompt-time instructions for the agent** — it loads them when reasoning about Sanafi-related questions. The skills tell the agent which MCP tool to call, when, and how to interpret the response.

## Prerequisites

1. **Sanafi account.** Sign up at [sana.bot](https://sana.bot).
2. **A Sanafi API key.** Generate one at [`sana.bot/gateway/app/api-keys`](https://sana.bot/gateway/app/api-keys).
   - Default scope `read:all` covers every skill below. See [`docs/scopes.md`](docs/scopes.md) for narrower options.
3. **Expose it to your agent as an env var:**
   ```bash
   export SANABOT_API_KEY=sk_...
   ```
   Add the line to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.) so it persists across restarts.

## Install

> 🔑 **About `${SANABOT_API_KEY}` in the snippets below:** environment-variable expansion in MCP `headers` works in **config files** (`.mcp.json`, `~/.gemini/settings.json`, VS Code `settings.json`, etc.) but **does not work in interactive UI fields** on Cursor, Windsurf, and similar IDEs — those send the literal `${SANABOT_API_KEY}` string and the gateway answers 401. When a client expects the header through a UI form, **paste your literal `sk_...` key** instead.

<details>
<summary><b>Claude Code (recommended)</b></summary>

Add this plugin marketplace, then install:

```bash
claude plugin marketplace add sanafi-onchain/sanabot-skills
claude plugin install sanabot-skills@sanafi
```

> Form is `<plugin-name>@<marketplace-name>` — the marketplace is `sanafi`, the plugin inside it is `sanabot-skills`.

Restart Claude Code. Skills appear automatically; the MCP server is registered via the bundled `.mcp.json`.

> Make sure `$SANABOT_API_KEY` is exported in the shell that launches Claude Code.

</details>

<details>
<summary><b>Cursor</b></summary>

1. Clone this repo somewhere stable: `git clone https://github.com/sanafi-onchain/sanabot-skills.git`
2. Open Cursor settings → **Features** → **MCP** → **Add new server**:
   - Name: `sanabot`
   - URL: `https://mcp.sana.bot/mcp`
   - Header: `Authorization: Bearer sk_...` ← **paste your literal API key** (Cursor's MCP form does not expand env vars)
3. In Cursor settings → **Rules**, add a project or user rule pointing at the cloned skills:
   ```
   When the user asks about Sanafi, their Sanafi wallet, or Sanafi card,
   read /absolute/path/to/sanabot-skills/skills/using-sanabot/SKILL.md
   and follow it.
   ```

> ⚠️ Cursor stores this header in its own settings. **Revoke and re-issue the key in `sana.bot/gateway/app/api-keys` if it ever needs to change** — Cursor's UI doesn't re-read env vars on restart.

</details>

<details>
<summary><b>Gemini CLI</b></summary>

```bash
git clone https://github.com/sanafi-onchain/sanabot-skills.git ~/sanabot-skills
mkdir -p ~/.gemini/skills
ln -s ~/sanabot-skills/skills/using-sanabot     ~/.gemini/skills/using-sanabot
ln -s ~/sanabot-skills/skills/sanafi-portfolio  ~/.gemini/skills/sanafi-portfolio
ln -s ~/sanabot-skills/skills/sanafi-card       ~/.gemini/skills/sanafi-card
```

Then add the MCP server to `~/.gemini/settings.json`:

```jsonc
{
  "mcpServers": {
    "sanabot": {
      "url": "https://mcp.sana.bot/mcp",
      "headers": { "Authorization": "Bearer ${SANABOT_API_KEY}" }
    }
  }
}
```

</details>

<details>
<summary><b>Windsurf</b></summary>

1. Clone this repo locally.
2. Open Windsurf settings → **Cascade** → **MCP Servers** → add the MCP config. **Paste your literal `sk_...` key** if entering via the settings UI; if you can edit the underlying config file directly, `${SANABOT_API_KEY}` is fine.
   ```jsonc
   {
     "sanabot": {
       "serverUrl": "https://mcp.sana.bot/mcp",
       "headers": { "Authorization": "Bearer sk_..." }
     }
   }
   ```
3. Settings → **Memories & Rules** → add a rule pointing Windsurf at the cloned `skills/` directory for Sanafi-related questions.

</details>

<details>
<summary><b>OpenCode</b></summary>

```bash
git clone https://github.com/sanafi-onchain/sanabot-skills.git
cd sanabot-skills
opencode plugin install .
```

OpenCode reads the `.mcp.json` and `skills/` from the cloned root automatically.

</details>

<details>
<summary><b>GitHub Copilot</b></summary>

Copilot in **VS Code** (with Agent Mode enabled):

1. Open VS Code settings JSON: ⌘⇧P → "Preferences: Open User Settings (JSON)".
2. Add the MCP server:
   ```jsonc
   "github.copilot.chat.experimental.mcp.servers": {
     "sanabot": {
       "url": "https://mcp.sana.bot/mcp",
       "headers": { "Authorization": "Bearer ${env:SANABOT_API_KEY}" }
     }
   }
   ```
3. Clone this repo and reference `skills/using-sanabot/SKILL.md` in a `.github/copilot-instructions.md` for the workspace, so Copilot pulls in the guidance when relevant.

</details>

<details>
<summary><b>Kiro IDE & CLI</b></summary>

Kiro's MCP config lives in `~/.kiro/settings.json` (CLI) or workspace settings (IDE):

```jsonc
{
  "mcpServers": {
    "sanabot": {
      "url": "https://mcp.sana.bot/mcp",
      "headers": { "Authorization": "Bearer ${SANABOT_API_KEY}" }
    }
  }
}
```

Symlink the `skills/` folder into Kiro's skills directory:

```bash
git clone https://github.com/sanafi-onchain/sanabot-skills.git
ln -s "$(pwd)/sanabot-skills/skills"/* ~/.kiro/skills/
```

</details>

<details>
<summary><b>Codex / Other Agents</b></summary>

For Codex and any other MCP-capable agent that supports Streamable HTTP transport:

1. Clone this repo: `git clone https://github.com/sanafi-onchain/sanabot-skills.git`
2. Point your agent's MCP config at the URL with the Bearer header:
   ```
   url: https://mcp.sana.bot/mcp
   header: Authorization: Bearer ${SANABOT_API_KEY}
   ```
3. Configure your agent to read `skills/*/SKILL.md` files at prompt time. The exact mechanism varies; check your agent's "skills" or "system instructions" documentation. If your agent has no skills mechanism, paste the contents of `skills/using-sanabot/SKILL.md` into the system prompt.

</details>

## Verifying

Once installed, ask your agent something only Sanabot can answer:

> *"What's my Sanafi net worth?"*

If the agent returns a number, you're good. If it says it can't reach Sanabot or returns a 401/403, see the **Troubleshooting** section in [`skills/using-sanabot/SKILL.md`](skills/using-sanabot/SKILL.md#error-handling).

## What can my agent do?

| Skill | Sample questions |
| --- | --- |
| `using-sanabot` | Foundational — loads automatically, sets context. |
| `sanafi-portfolio` | "What's my net worth?" · "What tokens do I own?" · "What's the price of SOL?" · "Did my deposit arrive?" |
| `sanafi-card` | "How much can I spend on my card?" · "What did I spend last week?" · "Where do I top up my card?" |

## Privacy

- The Sanabot gateway holds **no state**. Each request lives and dies in one HTTP round trip.
- Card PAN, CVV, and cardholder PII are **never** returned to your agent — the BE strips them. Even with `read:card`, the agent only sees last-4 + balance.
- Revoke any API key instantly from [`sana.bot/gateway/app/api-keys`](https://sana.bot/gateway/app/api-keys). The next request from that key returns 401.

## Support

- Issues & feature requests: [github.com/sanafi-onchain/sanabot-skills/issues](https://github.com/sanafi-onchain/sanabot-skills/issues)
- Live help: [Discord](https://discord.gg/sanafi) → `#sanabot-mcp`
- Status: [status.sana.bot](https://status.sana.bot)

## License

MIT — see [`LICENSE`](LICENSE).
