# agenticboxes-skill

Agent skill for AgenticBoxes — give an AI agent its own real email address it can send + receive from, over a single hosted HTTP API. No SMTP, no IMAP, no DKIM/SPF/DMARC setup; metered per message.

The canonical `SKILL.md` in this repo is the long-form (~500 line) reference suitable for skills marketplaces (skills.sh, LobeHub, ClawHub, etc.) and direct-URL `hermes skills install` use.

## Install

**Hermes:**
```bash
hermes skills install https://raw.githubusercontent.com/agenticbrian/agenticboxes-skill/main/SKILL.md
```

**Skills CLI (skills.sh):**
```bash
npx skills add agenticbrian/agenticboxes-skill
```

**Direct download:**
```bash
curl -O https://raw.githubusercontent.com/agenticbrian/agenticboxes-skill/main/SKILL.md
```

## Try it without commitment

Signup is free, no card on file, 250 send/receive credits included. The skill itself can drive signup via the `terminal` tool:

```
POST https://api.agenticboxes.email/api/v1/signup/agentic
```

The response contains a `bxs_live_…` API key and a domain. Three domain options at signup: free `agenticboxes.email` subdomain, bring your own, or buy through the platform.

## Service info

- Product: https://www.agenticboxes.email
- API reference: https://docs.agenticboxes.email/openapi.yaml
- Public audit + transparency: https://docs.agenticboxes.email/audit.html
- Hermes-style slim variant (HARDLINE-compliant): https://github.com/NousResearch/hermes-agent/pull/{TBD} (pending PR)

## License

MIT
