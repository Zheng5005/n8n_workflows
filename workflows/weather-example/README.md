# Weather Example — First Steps

The starting point: a 3-node workflow you can run with one click, no setup, no credentials.

```
Start Here (Manual Trigger)
  -> Fetch Madrid Weather (HTTP Request, GET Open-Meteo)
  -> Format Message (Code node)
```

## What it does

Fetches live weather for Madrid from [Open-Meteo](https://open-meteo.com) (free, no API key) and turns the raw JSON response into a readable message.

## Concepts it teaches

- **Triggers** — the Manual Trigger is a "run it with a button" node. Nothing happens until you click **Execute workflow**.
- **HTTP Request** — how n8n pulls external data in. The response lands in the `json` property, the core data shape every node works with.
- **Code node** — transformation. `$input.item.json` reads the previous node's output; whatever you return in `{ json: {...} }` becomes this node's output items.

## Setup

None. Import the workflow and run it.

## How to test

1. Import `workflow.json` (Menu → **Import from File**)
2. Click **Execute workflow** on the **Start Here** node
3. Click the **Format Message** node to inspect its output

## Try this

In the **Format Message** code, add another field to the returned object and watch the next node's input update — that's how data flows forward:

```js
return {
  json: {
    message,
    temperature: current.temperature_2m,
  },
};
```

## Next step

[weather-alert-scheduled](../weather-alert-scheduled/) extends this with a schedule trigger and conditional alerts.
