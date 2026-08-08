# Weather Alert — Scheduled Monitor (Webhook)

A scheduled version of the example workflow: runs every 6 hours and, only when it's cold, POSTs an alert to a webhook.

```
Every 6 Hours (Schedule Trigger)
  -> Fetch Madrid Weather (HTTP Request, GET Open-Meteo)
  -> Format + Extract Temp (Code node)
  -> Is It Cold? (< 10°C) (IF node)
       -> Send Cold Alert (HTTP Request, POST webhook)   [true]
       -> All Good (NoOp)                                [false]
```

## What it does

Fetches Madrid weather on a schedule. If the temperature is below 10°C, it sends a JSON alert to a [webhook.site](https://webhook.site) URL. Otherwise it does nothing.

## Concepts it teaches

- **Schedule Trigger** — time-based automation replaces the manual button. Configure the interval in the node (the example runs every 6 hours).
- **IF node** — branching. Data goes down exactly one of two paths based on a condition (`temperature < 10`).
- **HTTP POST** — sending data *out* instead of only fetching it.
- **NoOp** — the explicit "do nothing" branch, so the false path is visible on the canvas.

## Setup

### 1. Get a webhook URL

1. Open https://webhook.site — you get a unique URL like `https://webhook.site/abc-123...`
2. Keep that page open; incoming requests appear there live

### 2. Paste the URL

In the **Send Cold Alert** node, replace the default URL:

```
https://webhook.site/REPLACE-ME
```

with your webhook.site URL.

## How to test

1. Import `workflow.json` (Menu → **Import from File**)
2. Click **Execute workflow** (the button still works with a Schedule Trigger)
3. If it's below 10°C in Madrid, the request appears on your webhook.site page

To force the alert path regardless of weather, change the threshold in the **Is It Cold?** node from `10` to `50`, run again, then change it back.

## Why webhooks

A webhook is the same pattern used by Slack incoming webhooks and most POST APIs — the concept transfers directly. Replacing the webhook with Telegram is exactly what the [next version](../weather-alert-telegram/) does.
