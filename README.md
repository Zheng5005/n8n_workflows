# n8n Workflows

A growing showcase of [n8n](https://n8n.io) workflow examples. Each workflow is a self-contained directory with its importable `workflow.json` and its own README explaining setup and the concepts it teaches.

The goal is simple: small, runnable examples that build understanding — not production bundles. Start with the first one, run it, then compare the diffs between versions to see how real automations evolve.

## Repository layout

```
workflows/
├── weather-example/            # Manual trigger -> HTTP request -> Code node
├── weather-alert-scheduled/    # + Schedule trigger, IF branching, webhook alert
├── weather-alert-telegram/     # + Telegram notification with credentials
├── whatsapp-booking/           # Webhook + state machine + Google Calendar booking + Postgres guard
├── social-cross-poster/        # Cross-post to Facebook, Instagram, X, LinkedIn with per-platform text
├── morning-assistant/          # Daily briefing email: schedule + Google Calendar/Tasks + weather + SMTP
└── gmail-invoice-backup/       # Gmail trigger -> PDF filtering -> auto-created Drive folders
```

Each workflow README covers:

- What the workflow does and how it's wired
- The concepts it introduces
- Step-by-step setup (credentials, keys, URLs)
- How to test it

## Getting started

### 1. Run n8n

The easiest way is Docker:

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

Then open http://localhost:5678.

### 2. Import a workflow

1. Open n8n in your browser
2. Menu (top-left) → **Import from File**
3. Select a `workflows/<name>/workflow.json`

### 3. Follow that workflow's README

Every workflow needs slightly different setup — some need a webhook URL, some need a Telegram bot. Each README has the exact steps.

## The progression

| Workflow | Trigger | Data | Branching | Notification | Concepts |
|---|---|---|---|---|---|
| [weather-example](workflows/weather-example/) | Manual | HTTP GET (Open-Meteo) | — | — | Triggers, `json` data shape, Code node |
| [weather-alert-scheduled](workflows/weather-alert-scheduled/) | Schedule (every 6h) | HTTP GET | IF (temp < 10°C) | Webhook POST | Cron, branching, sending data out |
| [weather-alert-telegram](workflows/weather-alert-telegram/) | Schedule (every 6h) | HTTP GET | IF (temp < 10°C) | Telegram message | Credentials, chat app integrations |
| [whatsapp-booking](workflows/whatsapp-booking/) | Webhook (GET + POST) | Google Calendar (availability, create) | IF + Switch on conversation state | WhatsApp message (HTTP) | Webhooks, conversation state, scheduling a real event, DB constraint as a concurrency guard |
| [social-cross-poster](workflows/social-cross-poster/) | Manual | Code (per-platform text) | IF + Merge (image / no image) | Facebook, Instagram, X, LinkedIn | Cross-posting, continue-on-error, per-platform message adaptation |
| [morning-assistant](workflows/morning-assistant/) | Schedule (daily 7 AM, workflow timezone) | Google Calendar + Google Tasks + HTTP GET (Open-Meteo) | — | Email (SMTP) | Workflow timezone, `$now` day bounds, `executeOnce`, continue-on-error on data sources, Gmail app password |
| [gmail-invoice-backup](workflows/gmail-invoice-backup/) | Gmail polling (unread, subject query) | Email PDF attachments (binary) | IF (folder exists) + Merge (parallel branches) | Google Drive upload | Binary data, MoveBinaryData, folder auto-creation via raw API, OAuth2 credentials |

Diff adjacent pairs to see exactly what changes as a workflow grows.

## Security note

**Workflow files contain no secrets.** API tokens live in n8n's encrypted credential store, never in `workflow.json`. Files in this repo use placeholders (for example `REPLACE_WITH_CHAT_ID`) where a real value would be needed at runtime. If you export workflows from your own n8n instance and share them, double-check the exported JSON for embedded credentials first.

## Contributing

Want to add an example? Follow the same pattern:

1. Create `workflows/<name>/`
2. Add an importable `workflow.json` with placeholders instead of real credentials
3. Add a `README.md` with setup and testing steps
4. Keep it small and single-purpose
