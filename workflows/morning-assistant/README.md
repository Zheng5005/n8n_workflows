# Morning Assistant — Daily Briefing

Every day at 7:00 AM (America/El_Salvador) this workflow collects today's Google Calendar events, the current weather in San Salvador (Open-Meteo, free, no API key), and your pending Google Tasks (due today or overdue), then emails you a single text summary — one compact email that answers "what does today look like?" before you check any app.

```
Schedule Trigger (Every Day at 7 AM, daily)
  -> Day Bounds (Code, $now -> day start/end ISO)          # workflow timezone
  -> Fetch Weather (HTTP GET, Open-Meteo, San Salvador)     ①
  -> Fetch Tasks (Google Tasks, due today / overdue)        ①
  -> Fetch Calendar (Google Calendar, today's events)       ①
  -> Build Summary (Code, composes subject + text)
  -> Send Email (SMTP / Gmail app password)

① onError: continueRegularOutput — if a source fails, the
  briefing still goes out with "No disponible" in its section.
```

## What it does

- **One daily email** — a single text summary with three sections: weather (San Salvador), today's events, and pending tasks. Sent at 7:00 AM sharp.
- **Time zone-aware day bounds** — the **Day Bounds** Code node uses `$now` (Luxon, in the workflow's timezone) to compute the exact start and end of today, and every fetcher filters against those ISO bounds.
- **Weather without an API key** — Open-Meteo is free and keyless; the workflow maps the WMO weather code to a Spanish label ("Cielo despejado", "Tormenta", ...) and includes temperature, "feels like", humidity, and wind.
- **Calendar events with recurring series expanded** — the Google Calendar node fetches today's events with recurring events expanded, so a weekly meeting shows up on the right day.
- **Tasks that are due today or overdue** — Google Tasks is filtered with Due (Max) = end of today and `showCompleted: false`, on your default task list.
- **Failure-tolerant** — all three fetchers run with `onError: continueRegularOutput`; a failed source shows "No disponible" in the email instead of killing the run.

## Concepts it teaches

- **Time-of-day schedule + workflow timezone** — the Schedule Trigger uses `field: days`, `daysInterval: 1`, `triggerAtHour: 7`, `triggerAtMinute: 0`, and `settings.timezone: America/El_Salvador` drives both the trigger and `$now`.
- **Code node `$now` (Luxon) for day bounds** — `$now.startOf('day')` and `$now.plus({ days: 1 }).minus({ milliseconds: 1 })` produce the today window in the workflow's timezone, without any UTC math by hand.
- **`executeOnce: true` on list-returning nodes** — Google Calendar and Google Tasks return one item per event/task; `executeOnce` makes the Code node run once per execution instead of re-running once per event/task.
- **Google Calendar Get Many with After/Before** — `timeMin`/`timeMax` bound the query to today, and `options.recurringEventHandling: expand` turns instances of recurring events into individual items.
- **Google Tasks filtering** — `additionalFields.showCompleted: false` plus `dueMax` (end of today) keeps only open tasks that are due today or overdue; `task: @default` targets your default task list.
- **Continue-on-error on data fetchers** — `onError: continueRegularOutput` keeps the run alive when a source fails, and the defensive `try/catch` lookups in **Build Summary** turn a missing source into "No disponible".
- **Email Send (SMTP) with a Gmail app password** — the standard Gmail SMTP setup; the app password lives in the credential, never in the workflow file.

## Prerequisites

- An n8n instance (self-hosted or cloud).
- A Google account (for Calendar and Tasks OAuth2 — same account works for both).
- A Gmail account with an **app password** enabled (SMTP).
- No API key needed for the weather data.

## Setup (one time, ~10 minutes)

### 1. Import the workflow

n8n menu → **Import from File** → select `workflows/morning-assistant/workflow.json`.

### 2. Create the Google Calendar OAuth2 credential

n8n menu → **Settings → Credentials → Add credential** → **Google Calendar OAuth2**. Sign in with the Google account that owns the calendar and allow the scopes. Attach it to **Fetch Calendar**.

### 3. Create the Google Tasks OAuth2 credential

Same flow, **Google Tasks OAuth2**, same Google account. Attach it to **Fetch Tasks**. (If you connect both with the same Google account, reuse the same OAuth2 credentials if n8n shows them as compatible.)

### 4. Create the SMTP credential and set the addresses

**Settings → Credentials → Add credential** → **SMTP**:

- Host: `smtp.gmail.com`
- Port: `465`
- SSL/TLS enabled
- User: your full Gmail address
- Password: the **app password** (Google Account → Security → App passwords — the account password will not work for SMTP)

Attach it to **Send Email**, then replace the placeholders in that node:

```
REPLACE_WITH_SENDER_EMAIL     -> the Gmail address that sends
REPLACE_WITH_RECIPIENT_EMAIL  -> who receives the briefing
```

### 5. (Optional) Pick a different list or calendar

- **Tasks**: `task: @default` targets your default task list. To use another list, pick it from the dropdown in **Fetch Tasks**.
- **Calendar**: `calendar: primary` uses your primary calendar. To use another one, pick it in **Fetch Calendar**.

### 6. Save, activate, and test the timing

Save the workflow and toggle it to **Active**. To test right away: click **Execute workflow** once — it sends a real email immediately. To verify the schedule, wait for 7:00 AM, or temporarily change the trigger time (e.g. `triggerAtMinute: 5`) to confirm it fires on time.

## How to test

1. Click **Execute workflow** once — you get a real email with the three sections.
2. Check the weather section reads the current conditions for San Salvador.
3. Add a calendar event or a task with a due date today, run again, and confirm both appear.
4. To test failure tolerance, temporarily set a wrong timezone or an unreachable weather URL on **Fetch Weather** and run again: the email still arrives with "No disponible" under CLIMA.
5. Confirm the daily trigger: activate the workflow and verify an execution at 7:00 AM (or change the trigger minute to a near-future time).

## Customization

- **Weather location** — change the `latitude`/`longitude` in the **Fetch Weather** URL to any city Open-Meteo supports.
- **Weather wording** — the `WEATHER_LABELS` map in **Build Summary** translates WMO codes to Spanish labels; edit the strings or add missing codes there.
- **Email format** — the subject and body are built in **Build Summary**; change the section headers, the ordering, or add a greeting/name.

## Security

Workflow files contain **no secrets** — tokens live in n8n's encrypted credential store. This file uses `REPLACE_WITH_*` placeholders for credential IDs and email addresses. The SMTP credential uses a Gmail **app password**, never the account password. If you export this workflow from your instance, credential references export but tokens do not.
