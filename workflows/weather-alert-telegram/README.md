# Weather Alert — Telegram

The scheduled monitor, with the alert delivered as a Telegram message instead of a webhook POST.

```
Every 6 Hours (Schedule Trigger)
  -> Fetch Madrid Weather (HTTP Request, GET Open-Meteo)
  -> Format + Extract Temp (Code node)
  -> Is It Cold? (< 10°C) (IF node)
       -> Send Telegram Alert (Telegram node)  [true]
       -> All Good (NoOp)                      [false]
```

## What it does

Fetches Madrid weather on a schedule. If the temperature is below 10°C, it sends a Telegram message to your chat.

## Concepts it teaches

- **Credentials** — the bot token lives in n8n's encrypted credential store, **never in the workflow file**. The JSON only references a credential by id/name; anyone importing this file gets a "missing credential" indicator until they create and attach their own. That's what makes this file safe to commit to a public repo.
- **Telegram node** — a chat-app integration. The same pattern applies to Slack, Discord, email, and similar notification nodes.

## Setup (one time, ~3 minutes)

### 1. Create the bot

1. In Telegram, open **@BotFather**
2. Send `/newbot`, choose a name and username
3. Copy the **token** it replies with (something like `1234567890:AAH...`)

### 2. Get your chat ID

1. Message your bot once (any text)
2. Open **@userinfobot** and send it anything — it replies with your numeric ID

### 3. Create the credential in n8n

1. n8n menu → **Settings → Credentials → Add credential**
2. Search for **Telegram API**
3. Paste the bot token, save

### 4. Attach the credential

On the **Send Telegram Alert** node, click the credential dropdown and select the credential you just created.

### 5. Replace the chat ID

In the node's **Chat ID** field, replace:

```
REPLACE_WITH_CHAT_ID
```

with your numeric ID from step 2.

## How to test

1. Import `workflow.json` (Menu → **Import from File**)
2. Click **Execute workflow** (the button still works with a Schedule Trigger)
3. If it's below 10°C in Madrid, you get a Telegram message

To force the alert path regardless of weather, change the threshold in the **Is It Cold?** node from `10` to `50`, run again, then change it back.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| "Chat not found" error | The Chat ID is wrong or the bot never received a message from you — redo step 2 |
| "Unauthorized" error | The token in the credential is wrong, or you attached a different bot — redo step 3 |
| Node shows a red credential dot | The credential isn't attached — redo step 4 |

## Security

The token is only ever stored in n8n's credential store, never in `workflow.json`. If you export this workflow from your instance, the credential reference exports but the token does not.
