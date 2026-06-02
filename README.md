# Sana Bot Skills

**Sana Bot** — AI-agent skills for the [Sana](https://sana.bot) super-app on Solana. Install once, and your agent can answer questions about your **wallet, prices, transactions, and card** by routing through the hosted MCP gateway at `https://mcp.sana.bot`.

> No local install of Sana infrastructure. Every call uses your **Sana API key** and lives a single round trip.

## What's in this repo

```
skills/
  using-sanabot/        ← foundational: connection, tool catalogue, scopes, errors
  sanafi-portfolio/     ← net worth, holdings, prices, account, notifications
  sanafi-card/          ← card metadata, spending power, card transactions
.claude-plugin/         ← Claude Code plugin + marketplace manifests
.codex-plugin/          ← Codex CLI plugin manifest
.agents/plugins/        ← Codex CLI marketplace manifest
.mcp.json               ← MCP server config (loaded by plugins)
llms-full.md            ← all skills + scopes + reference in ONE file (for no-skills-dir agents)
scripts/
  build-skills-bundle.ts ← regenerates llms-full.md from the sources
docs/
  scopes.md             ← scope vocabulary + revocation
  skills-reference.md   ← input/output per tool the gateway exposes
```

> `llms-full.md` is **generated** — after editing a skill or doc, run `node scripts/build-skills-bundle.ts` and commit the result. Never hand-edit it. (Runs directly on **Node ≥ 22.18 / 23.6** via native TypeScript stripping — no build step or deps.)

Each `skills/<name>/SKILL.md` is **prompt-time instructions for the agent** — it loads them when reasoning about Sanafi-related questions. The skills tell the agent which MCP tool to call, when, and how to interpret the response.

## Prerequisites

1. **Sanafi account.** Sign up at [sana.bot](https://sana.bot).
2. **A Sanafi API key.** Generate one at [`sana.bot/gateway/app/api-keys`](https://sana.bot/gateway/app/api-keys).
   - Default scope `read:all` covers every skill below. See [`docs/scopes.md`](docs/scopes.md) for narrower options.
3. **Expose it to your agent as an env var:**
   ```bash
   export SANABOT_API_KEY=sana_live_...
   ```
   Add the line to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.) so it persists across restarts.

## Install

> 🔑 **About `${SANABOT_API_KEY}` in the snippets below:** environment-variable expansion in MCP `headers` works in **config files** (`.mcp.json`, `~/.gemini/settings.json`, VS Code `settings.json`, etc.) but **does not work in interactive UI fields** on Cursor, Windsurf, and similar IDEs — those send the literal `${SANABOT_API_KEY}` string and the gateway answers 401. When a client expects the header through a UI form, **paste your literal `sana_live_...` key** instead.

<details>
<summary><b>Claude Code (recommended)</b></summary>

Three steps. Claude Code's plugin loader handles skills but not URL-based MCP servers, so the MCP server is added separately.

**1. Install the plugin** (loads the skill packs):

```bash
claude plugin marketplace add sanafi-onchain/sanabot-skills
claude plugin install sanabot-skills@sana-bot
```

> Form is `<plugin-name>@<marketplace-name>` — the marketplace is `sana-bot`, the plugin inside it is `sanabot-skills`.

**2. Export your API key in the shell that will run Claude Code:**

```bash
export SANABOT_API_KEY=sana_live_...
```

**3. Register the MCP server:**

```bash
claude mcp add sanabot --transport http https://mcp.sana.bot/mcp \
  --header "Authorization: Bearer $SANABOT_API_KEY"
```

The double-quoted `$SANABOT_API_KEY` is expanded by your shell *at this moment* — the resulting literal value is what Claude stores. Restart Claude Code; verify with `claude mcp list` (you should see `sanabot`).

> **Rotating the key:** if you ever revoke + regenerate, `export` alone is not enough — Claude has the old literal stored. Do:
>
> ```bash
> claude mcp remove sanabot
> export SANABOT_API_KEY=<new key>
> claude mcp add sanabot --transport http https://mcp.sana.bot/mcp \
>   --header "Authorization: Bearer $SANABOT_API_KEY"
> ```
>
> Then restart Claude Code.

</details>

<details>
<summary><b>Cursor</b></summary>

1. Clone this repo somewhere stable: `git clone https://github.com/sanafi-onchain/sanabot-skills.git`
2. Open Cursor settings → **Features** → **MCP** → **Add new server**:
   - Name: `sanabot`
   - URL: `https://mcp.sana.bot/mcp`
   - Header: `Authorization: Bearer sana_live_...` ← **paste your literal API key** (Cursor's MCP form does not expand env vars)
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

Export the key in the shell that starts Gemini CLI:

```bash
export SANABOT_API_KEY=sana_live_...
```

> **Rotating the key:** Gemini expands `${SANABOT_API_KEY}` at runtime, so just `export SANABOT_API_KEY=<new key>` and restart Gemini CLI — no settings edit needed.

</details>

<details>
<summary><b>Windsurf</b></summary>

1. Clone this repo locally.
2. Open Windsurf settings → **Cascade** → **MCP Servers** → add the MCP config. **Paste your literal `sana_live_...` key** if entering via the settings UI; if you can edit the underlying config file directly, `${SANABOT_API_KEY}` is fine.
   ```jsonc
   {
     "sanabot": {
       "serverUrl": "https://mcp.sana.bot/mcp",
       "headers": { "Authorization": "Bearer sana_live_..." }
     }
   }
   ```
3. Settings → **Memories & Rules** → add a rule pointing Windsurf at the cloned `skills/` directory for Sanafi-related questions.

> **Rotating the key:** if you pasted a literal `sana_live_...` into the UI, you must open the same Windsurf MCP server config and replace the bearer value with the new key, then restart Windsurf. If you edited the underlying config file with `${SANABOT_API_KEY}`, just `export SANABOT_API_KEY=<new>` and restart.

</details>

<details>
<summary><b>OpenCode</b></summary>

```bash
git clone https://github.com/sanafi-onchain/sanabot-skills.git
cd sanabot-skills
opencode plugin install .

export SANABOT_API_KEY=sana_live_...
```

OpenCode reads the `.mcp.json` and `skills/` from the cloned root automatically.

> **Rotating the key:** OpenCode reads `${SANABOT_API_KEY}` from the env at runtime — just `export SANABOT_API_KEY=<new>` and restart OpenCode.

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
4. Export the key in the shell that started VS Code (or set it as a user env var on your OS):
   ```bash
   export SANABOT_API_KEY=sana_live_...
   ```

> **Rotating the key:** VS Code expands `${env:SANABOT_API_KEY}` at runtime — `export SANABOT_API_KEY=<new>` and reload VS Code is enough.

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

export SANABOT_API_KEY=sana_live_...
```

> **Rotating the key:** Kiro expands `${SANABOT_API_KEY}` at runtime — `export SANABOT_API_KEY=<new>` and restart Kiro is enough.

</details>

<details>
<summary><b>Codex CLI (recommended)</b></summary>

This repo ships a Codex plugin manifest (`.codex-plugin/plugin.json`) plus a `sana-bot` marketplace manifest (`.agents/plugins/marketplace.json`), so Codex discovers the skill packs and the MCP server config from one install — same as Claude Code.

**1. Export your API key** in the shell that runs Codex (the bundled `.mcp.json` reads `${SANABOT_API_KEY}` at runtime):

```bash
export SANABOT_API_KEY=sana_live_...
```

**2. Add this repo as a plugin source and install it.** Inside Codex, open the plugin browser and install, or use the slash commands:

```
/plugin marketplace add sanafi-onchain/sanabot-skills
/plugin install sanabot-skills
/reload-plugins
```

> The interactive equivalent: run `/plugins`, select **Install plugin**, and pick `sanabot-skills`. Exact command surface follows the [official Codex plugins docs](https://developers.openai.com/codex/plugins) — check there if your Codex version differs.

**3. Verify** — ask *"What's my Sana net worth?"*. A number means the skills + MCP server loaded.

> **Rotating the key:** Codex expands `${SANABOT_API_KEY}` from the env at runtime — `export SANABOT_API_KEY=<new>` and restart Codex.

</details>

<details>
<summary><b>openclaw</b></summary>

openclaw takes the skills as a single instruction blob, so use the consolidated [`llms-full.md`](llms-full.md) — one file that bundles every skill + scopes + tool reference.

1. **Give openclaw [`llms-full.md`](llms-full.md) as its instructions / system context** (paste it, or point openclaw's instruction file at it). That one file is everything the agent needs to know — no `skills/` directory required.
2. **Connect the MCP server** so the agent can actually call the tools:
   ```
   url: https://mcp.sana.bot/mcp
   header: Authorization: Bearer ${SANABOT_API_KEY}
   ```
3. **Export your key** in the shell that runs openclaw:
   ```bash
   export SANABOT_API_KEY=sana_live_...
   ```

> **Rotating the key:** if openclaw's config uses `${SANABOT_API_KEY}` it re-reads on restart — just `export` the new value. If you pasted a literal key, update it there too.

</details>

<details>
<summary><b>hermes-agent</b></summary>

hermes-agent is onboarded the same way — one consolidated file plus the MCP connection.

1. **Load [`llms-full.md`](llms-full.md) as hermes-agent's instructions / context.** It bundles `using-sanabot` + `sanafi-portfolio` + `sanafi-card` + scopes + tool reference into a single document.
2. **Connect the MCP server:**
   ```
   url: https://mcp.sana.bot/mcp
   header: Authorization: Bearer ${SANABOT_API_KEY}
   ```
3. **Export your key** in the shell that runs hermes-agent:
   ```bash
   export SANABOT_API_KEY=sana_live_...
   ```

> **Rotating the key:** same as above — env-var expansion re-reads on restart; a pasted literal must be updated in place.

</details>

<details>
<summary><b>Other MCP agents</b></summary>

For any other MCP-capable agent that supports Streamable HTTP transport:

1. Clone this repo: `git clone https://github.com/sanafi-onchain/sanabot-skills.git`
2. Point your agent's MCP config at the URL with the Bearer header:
   ```
   url: https://mcp.sana.bot/mcp
   header: Authorization: Bearer ${SANABOT_API_KEY}
   ```
3. Configure your agent to read `skills/*/SKILL.md` files at prompt time. The exact mechanism varies; check your agent's "skills" or "system instructions" documentation. **If your agent has no skills mechanism, give it [`llms-full.md`](llms-full.md) as its system prompt** — that single file bundles every skill + scopes + tool reference.
4. Export the key in the shell that runs your agent:
   ```bash
   export SANABOT_API_KEY=sana_live_...
   ```

> **Rotating the key:** depends on the agent. If its config file uses `${SANABOT_API_KEY}` (env-var expansion at runtime), just `export` the new value and restart. If the agent stored a literal value (UI form, captured-at-write-time CLI), edit that stored value too.

</details>

## Verifying

Once installed, ask your agent something only Sanabot can answer:

> *"What's my Sanafi net worth?"*

If the agent returns a number, you're good. If it says it can't reach Sanabot or returns a 401/403, see the **Troubleshooting** section in [`skills/using-sanabot/SKILL.md`](skills/using-sanabot/SKILL.md#error-handling).

## Keeping up to date

New tools and skill updates ship through this repo. Claude Code (and similar marketplace clients) **cache the marketplace locally** — your agent will NOT see new tools or updated skill docs until you refresh.

**Manual refresh in Claude Code:**

```
/plugins  →  Marketplaces tab  →  sana-bot  →  Update marketplace
```

**Recommended** — enable auto-update so future releases land without manual steps:

```
/plugins  →  Marketplaces tab  →  sana-bot  →  Enable auto-update
```

**Symptom of a stale marketplace:** the agent says "I can't do X" for something the current docs describe as supported (e.g. you read about `wallet_swap` here but the agent claims swap isn't available). Update the marketplace and restart the session.

For non-Claude-Code clients (Cursor, Gemini CLI, etc.) that consume the skills via a local clone, `git pull origin main` inside the cloned repo is the equivalent step.

## What can my agent do?

| Skill | Sample questions |
| --- | --- |
| `using-sanabot` | Foundational — loads automatically, sets context. |
| `sanafi-portfolio` | "What's my net worth?" · "What tokens do I own?" · "What's the price of SOL?" · "Did my deposit arrive?" |
| `sanafi-card` | "How much can I spend on my card?" · "What did I spend last week?" · "What's my full card number?" · "How do I get a card?" · "Where do I top up my card?" |

## Privacy

- The Sana Bot gateway holds **no state**. Each request lives and dies in one HTTP round trip.
- **Full card number + CVV** are available only via the dedicated `get_card_sensitive` tool, gated by its own opt-in `read:card_sensitive` scope (a plain `read` / `read:all` key can **never** reveal them — you tick that scope explicitly per key). A well-behaved agent calls it **only when you explicitly ask** for your full card details — never proactively, and it won't persist them. `get_card` / `get_card_balance` never include them.
- **Cardholder PII** (name, phone, home address) is **never** returned to your agent — the BE strips it.
- Revoke any API key instantly from [`sana.bot/gateway/app/api-keys`](https://sana.bot/gateway/app/api-keys). The next request from that key returns 401.

## Support

- Issues & feature requests: [github.com/sanafi-onchain/sanabot-skills/issues](https://github.com/sanafi-onchain/sanabot-skills/issues)
- Live help: [Discord](https://discord.gg/sanafi) → `#sanabot-mcp`
- Status: [status.sana.bot](https://status.sana.bot)

## License

MIT — see [`LICENSE`](LICENSE).
