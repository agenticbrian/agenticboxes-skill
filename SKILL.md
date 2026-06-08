---
name: agenticboxes-email
description: "Agentic email: 40¢/1000 msgs, unlimited inboxes, no monthly."
version: 1.0.0
author: Brian Becker (agenticbrian)
license: MIT
platforms: [macos, linux, windows]
required_environment_variables:
  - name: AGENTICBOXES_API_KEY
    prompt: "agenticboxes API key (bxs_live_…)"
    help: "Get one free at https://www.agenticboxes.email — or call POST /signup/agentic from the terminal tool (no card)."
    required_for: "sending and receiving mail"
metadata:
  hermes:
    tags: [email, communication, agentic]
    category: email
---

# Agenticboxes Email Skill

Give the agent its own send/receive email address through a single hosted
HTTP API. No SMTP, no IMAP, no DKIM/SPF/DMARC setup, no monthly bill —
metered per message. The canonical reference lives at
https://docs.agenticboxes.email/openapi.yaml.

## When to Use

- The agent needs a real email address it owns and can send from.
- The agent needs to RECEIVE email (signup confirmations, 2FA codes,
  inbound replies, conversations with humans or other agents).
- The agent should NOT read the user's personal inbox — use `himalaya` or
  the `gmail` skill for that. This skill is the agent's OWN identity.

## Prerequisites

- `AGENTICBOXES_API_KEY` — see `required_environment_variables` above.
  Acquire interactively by invoking `terminal` to POST
  `/api/v1/signup/agentic`. **No card on file required to start** —
  signup includes 250 free send + receive credits so the agent can
  exercise the full skill before any payment step. Returns a
  `bxs_live_…` key plus a domain.
- **Domain options at signup** (pick one):
  - **Free third-level subdomain** under `agenticboxes.email` — assigned
    automatically (e.g. `friendly-otter-3.agenticboxes.email`). Zero
    setup, works immediately.
  - **Bring your own** — pass `{"domain": "your-domain.com",
    "domain_mode": "byo_manual"}` in the signup body; you receive DNS
    records to add at your registrar (SPF, DKIM, DMARC, MAIL FROM).
  - **Buy through the platform** — pass `{"domain": "new-domain.com",
    "domain_mode": "register"}`; signup quotes the registration price
    and Stripe-charges on confirm. The platform owns DNS + auth setup.
- An MCP server is available at `https://mcp.agenticboxes.email/mcp` —
  configure it once if you want tool-style integration (see Quick
  Reference). Otherwise the `terminal` tool is sufficient.
- The base URL is `https://api.agenticboxes.email/api/v1`. Every call
  authenticates with `Authorization: Bearer $AGENTICBOXES_API_KEY` (the
  two signup endpoints are the only exceptions).

## How to Run

Use the `terminal` tool for every interaction with this skill. The
service is plain HTTP + JSON — no SDK installation required. Two patterns:

- **Direct API** — invoke `terminal` to make HTTP requests against
  `https://api.agenticboxes.email/api/v1/...` with the bearer token.
  Standard HTTP client; no SDK or platform-specific tooling required.
- **MCP** — point your MCP client at `https://mcp.agenticboxes.email/mcp`
  with the bearer in `Authorization`. The MCP server exposes
  `send_email`, `list_messages`, `get_message`, `get_balance`. The
  account must call `POST /account/mcp/enable` once (admin scope) before
  the MCP server accepts its API keys.

## Quick Reference

Base URL: `https://api.agenticboxes.email/api/v1`. Bearer auth.

| Endpoint | Purpose |
|---|---|
| `POST /signup/agentic` | Free signup — returns API key + auto-domain |
| `POST /messages/send` | Send an email |
| `POST /messages/send/bulk` | Bulk send (async, halt-on-bounce, ≤5,000 msgs) |
| `GET /messages` | List inbound + outbound mail |
| `GET /messages/{id}` | Single message metadata + parsed body |
| `GET /events?since=<cursor>` | Pull every event the platform emits for this account |
| `GET /events/verify` | Hash-chain integrity check on this account's events |
| `GET /account/credit/balance` | Current credit balance (cents) |
| `POST /account/credit/topup/mpp` | Stripe Link top-up — agent initiates, human approves on wallet |
| `POST /account/sending-ip` | Provision a dedicated SES IP (reputation isolation) |
| `GET /account/audit/message/{id}` | KMS-signed evidence bundle for one message |
| `POST /support/questions` | Ask the platform team a question |

## Procedure

### 1. Sign up (one time)

If the agent doesn't already have a key, invoke `terminal`:

```
POST https://api.agenticboxes.email/api/v1/signup/agentic
Body: {}
```

The response contains `api_key` (a `bxs_live_…` string), `domain`, and
`primary_box_address`. Persist the key into the environment via the
`required_environment_variables` flow.

### 2. Send a message

Invoke `terminal`:

```
POST https://api.agenticboxes.email/api/v1/messages/send
Authorization: Bearer $AGENTICBOXES_API_KEY
Body: {"to": "someone@example.com", "subject": "Hello", "text": "Sent by my agent."}
```

Optional fields: `from` (defaults to primary box), `cc`, `bcc`,
`reply_to`, `html`, `attachments` (`[{filename, content_b64, content_type}]`),
`idempotency_key`, `context` (opaque JSON echoed back on any inbound reply).
Response includes `message_id` and a `billing` breakdown.

### 3. Receive — three modes onto one stream

- **Polling** — invoke `terminal` to `GET /events?since=<cursor>`. The
  unified feed delivers `mail.received`, `support.answered`,
  `domain.ready`, and other events in one ordered stream. Process a
  page, then poll again with `since=<response.next_cursor>`.
- **Webhook** — invoke `terminal` to `PUT /account/callback-webhook` with
  `{"url": "https://your.host/webhook", "secret": "..."}`. The platform
  POSTs each event to the URL with an `X-Agenticboxes-Signature` header.
- **IoT/MQTT** — invoke `terminal` to `POST /account/iot/provision`. The
  response gives an MQTT client cert + endpoint for sub-second push.

### 4. Top up credit

When the balance is low, invoke `terminal`:

```
POST https://api.agenticboxes.email/api/v1/account/credit/topup/mpp
Body: {"amount_cents": 1000}
```

The response contains a `pay_url`. Hand that URL to the human owner —
they approve the charge in their Stripe Link wallet (no agent-side
secret handling). On approval, the account credit increases and
`account.credit.topped_up` lands in the event feed.

## Pitfalls

- **Authentication on POST /messages/send** uses `Bearer`, not basic
  auth or query string. Headers are case-insensitive; the value is the
  `bxs_live_…` string verbatim.
- **402 on send** means the account is in deficit. Response includes a
  `top_up` block with the exact recovery call. Inbound mail still flows;
  only outbound sends are gated.
- **403 with `reputation_status: suspended`** means the account's bounce
  or complaint rate hit the per-account threshold. Operator can lift
  via the platform's reinstate endpoint; the structural fix for repeat
  cases is `POST /account/sending-ip` (dedicated IP, $25/mo).
- **Bulk sends** (`POST /messages/send/bulk`) return `202` and a
  `job_id` — they're async. Poll `GET /messages/send/bulk/{id}` or watch
  the events feed for `bulk.batch_completed`, `bulk.paused`,
  `bulk.completed`. Halt thresholds (default 2% bounce / 0.5% complaint
  per batch) pause the whole job rather than burn a bad list.
- **Inbound bounce/complaint lag**: hard bounces land within 60s of SES
  accepting; soft bounces over hours; Gmail/Yahoo complaints over hours
  to days. Bulk-send halt eval honors this — the complaint gate fires
  on a lagged batch.

## Verification

Invoke `terminal` with the configured key against the balance endpoint —
a `200` response with `{"credit_balance_cents": ...}` proves the key
works end-to-end (auth, network, account exists, balance retrievable):

```
GET https://api.agenticboxes.email/api/v1/account/credit/balance
Authorization: Bearer $AGENTICBOXES_API_KEY
```

A `401` means the key is wrong; a `404` means the account is closed.

## References

- API reference: https://docs.agenticboxes.email/openapi.yaml
- Full skill (long form): https://docs.agenticboxes.email/skills/hermes/SKILL.md
- Public audit + transparency page: https://docs.agenticboxes.email/audit.html
- Pricing + product: https://www.agenticboxes.email
