# Contributing

Thanks for considering a contribution to Sanabot Skills.

This is the public, dev-facing repo for the hosted MCP gateway at `mcp.sana.bot`. It ships **AgentSkills-format skill packs** (markdown `SKILL.md` files agents load at prompt time) plus per-platform install guidance. The Worker that actually runs `mcp.sana.bot` lives in a separate private repo and is **not** part of this codebase.

## What we accept

- **Skill content improvements** — clearer instructions, better workflow examples, tighter wording in any `skills/*/SKILL.md`.
- **New skill packs** for distinct workflow domains (e.g. a future `sanafi-defi-strategies/` once the right primitives exist).
- **Per-platform install fixes** in the README. If you're the first to successfully wire Sanabot into a new agent, send a PR adding it.
- **Doc fixes** in `docs/` — broken links, typos, clarifications.
- **Plugin manifest fixes** if `.claude-plugin/marketplace.json` or `.mcp.json` need tightening.
- **Tests / validators** for the markdown frontmatter, if you feel motivated.

## What we don't accept here

- Server-side / Worker changes (the implementation is private).
- New MCP **tools** — those have to land in the gateway repo first; the skills here only describe what the server already exposes.
- Anything that requires a `read:*` scope we haven't published yet.

If your idea fits one of those, please open an issue describing it — we'll route it internally.

## Adding a new skill pack

1. Pick a clear, dash-separated name (`sanafi-<thing>`).
2. Create `skills/<name>/SKILL.md` with this frontmatter shape (copy verbatim from any existing `skills/*/SKILL.md` and edit the values):
   ```yaml
   ---
   name: <name>
   description: <one paragraph — what it does, when to load it, what it depends on>
   license: MIT
   metadata:
     author: sanafi-onchain
     version: "0.1.0"
   tags:
     - sanafi
     - sanabot
     - <topic-tag>
     - <topic-tag>
   ---
   ```
   Required fields: top-level `name`, `description`, `license`; nested `metadata.author` + `metadata.version`; a non-empty `tags` array. Frontmatter is YAML — use block-list (`- value` on its own line) for `tags`, not flow-list (`[a, b]`), to match the existing skills.
3. Body content: instructions written for the **agent**, not the human reader. Use the `using-sanabot`, `sanafi-portfolio`, and `sanafi-card` skills as templates — they share a structure (quick-reference table → tool-by-tool details → combined workflows → what-not-to-do).
4. List the skill in the README's "What can my agent do?" table.
5. Open a PR.

## Skill writing principles

- **Be specific.** "Use `get_holdings`" is better than "query the portfolio API."
- **Cover failure modes.** Tell the agent what `401`, `403`, `429` mean and how to surface them.
- **Respect the privacy contract.** Sanabot deliberately doesn't return PAN, CVV, cardholder PII, or internal IDs. Skills must never claim those are available.
- **Don't fabricate.** If a tool returns nothing for a symbol, the skill should tell the agent to say so — not to invent prices.
- **Keep skills focused.** One workflow domain per skill. Cross-link via the `description` field's "Requires the X skill" pattern.

## PR checklist

- [ ] Each new or modified skill has valid YAML frontmatter (`name`, `description`, `license`, `metadata`, `tags`).
- [ ] Instructions tell the agent the **right tool** for each user-facing question.
- [ ] Privacy boundaries (no PAN/CVV/PII, no write claims, no key echoing) are respected.
- [ ] After editing any `skills/*/SKILL.md` or `docs/*.md`, **regenerated the bundle**: `node scripts/build-skills-bundle.ts`, and committed the updated `SKILL.md`. (It's auto-generated — never hand-edit it.)
- [ ] No drive-by reformatting unrelated to the change.
- [ ] One PR per topic.

## Reporting security issues

**Do not** file vulnerabilities on the public issue tracker. See [`SECURITY.md`](SECURITY.md) for the disclosure process.
