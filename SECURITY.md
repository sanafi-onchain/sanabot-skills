# Security Policy

## Reporting a vulnerability

If you discover a security vulnerability in this package, in the hosted gateway at `mcp.sana.bot`, or in any other Sanafi-published infrastructure that touches the agent-skills surface, **please do not open a public issue or pull request**.

Instead, email **jume@sanafi.xyz** with:

- A description of the issue
- Steps to reproduce
- Any proof-of-concept code or payloads
- Your name and contact details (we'll credit you in the advisory if you'd like)

We aim to:

- Acknowledge your report within **2 business days**.
- Provide an initial assessment within **5 business days**.
- Coordinate disclosure timing with you for issues that warrant a published advisory.

## In scope

- The skill packs published in this repo (`skills/*/SKILL.md`).
- The plugin manifests (`.claude-plugin/`, `.mcp.json`) and per-platform install guidance.
- The hosted gateway at `mcp.sana.bot`.

## Out of scope

- Issues that require an attacker to already possess your API key or to control your device.
- Rate-limit or quota tuning concerns that aren't a denial-of-service vector.
- Self-XSS or social-engineering against your own account.

## Hall of fame

Researchers who responsibly disclose verified vulnerabilities are listed in our advisory acknowledgements with their permission.

## Out-of-band questions

For non-security questions about scopes, API keys, or skill behaviour, please use the [public issue tracker](https://github.com/sanafi-onchain/sanabot-skills/issues) or the Sanafi Discord.
