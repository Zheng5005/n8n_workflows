# WhatsApp Booking Bot — Google Calendar

A WhatsApp Business bot that books appointments in a Google Calendar. The customer chats with an option-driven menu (no AI): choose a service, pick a date, pick a free time slot, type their name, and confirm. The slot is created in the calendar and a confirmation message is sent back — all from a single webhook workflow.

```
WhatsApp Webhook (GET = verification, POST = messages)
  |-- GET -> Verify Challenge -> Challenge Verified? (IF)
  |          |-> Respond Challenge            [verified]
  |          |-> Respond Verification Failed  [rejected]
  |
  |-- POST -> Extract Inbound (Code)
          -> Config (Code)
          -> State Machine (Code, $getWorkflowStaticData)
          -> Step Router (Switch on the current step)
               |-> Handle Menu
               |-> Handle Service
               |-> Handle Date -> Date Valid? (IF)
               |       |-> Check Day Availability (Google Calendar)
               |       |     -> Compute Free Slots (Code)
               |       |     -> Slots Available? (IF)
               |       |          |-> Show Slots            -> Send WhatsApp Message
               |       |          |-> Handle No Slots       -> Send WhatsApp Message
               |       |-> Send WhatsApp Message            [invalid date]
               |-> Handle Time          -> Send WhatsApp Message
               |-> Handle Name          -> Send WhatsApp Message
               |-> Handle Confirm -> Confirmed? (IF)
               |       |-> Create Booking (Google Calendar) -> Booking Result -> Send WhatsApp Message
               |       |-> Send WhatsApp Message            [cancelled / re-ask]
               |-> Respond OK                               [status events, no text]
```

Every message branch ends in the **single HTTP sink** `Send WhatsApp Message` → `Respond OK`, so the webhook always answers 200 and WhatsApp never retries an already-handled message.

## What it does

- **Verification challenge** — Meta verifies the webhook with a `GET` containing `hub.verify_token`; the workflow echoes `hub.challenge` back so the subscription activates.
- **Conversation state per phone** — each phone number gets its own `{ step, data }` object in `$getWorkflowStaticData('global')`, so a dozen customers can be mid-conversation at the same time with no database.
- **Real availability** — the Google Calendar *Availability* node returns the busy ranges for the chosen day, and a Code node computes the free time slots (service duration, business hours, time zone offset).
- **One-click booking** — on confirmation the *Create* operation adds the event to the calendar, then the bot confirms the booking and resets the conversation to the menu.

## Concepts it teaches

- **Webhook with two HTTP methods** — `multipleMethods` + `httpMethod: [GET, POST]` gives the webhook node two outputs: output 0 fires on GET, output 1 on POST.
- **Respond to Webhook node** — with the webhook's *Response Mode* set to *Using Respond to Webhook Node*, the reply is decided by a node deep in the flow (the challenge echo, or a silent 200).
- **`$getWorkflowStaticData('global')`** — n8n's built-in key-value store for workflows; mutations persist across executions. Here it is keyed by phone number.
- **A Code node as a state machine** — the current step is just a string; a Switch node routes each step to its own handler.
- **Slots math in UTC** — the calendar works in UTC; the bot shifts by the business time zone offset and shows the customer local times.

## Prerequisites

- n8n reachable at a **public HTTPS** URL (required by Meta).
- A **Meta developer app** with a WhatsApp Business number, its **phone number ID**, and an **access token**.
- A **Google Calendar** (a normal personal calendar is fine).
- Two n8n credentials: **Google Calendar OAuth2** and **Header Auth**.

## Setup (one time, ~20 minutes)

### 1. Connect Google Calendar

1. n8n menu → **Settings → Credentials → Add credential** → **Google Calendar OAuth2**
2. Sign in with the Google account that owns the calendar and allow the scopes
3. Attach this credential to **Check Day Availability** and **Create Booking**

### 2. Add the webhook in the Meta Developer Portal

1. Open your app → **WhatsApp → Configuration**
2. Under **Webhook**, click **Edit** and set:
   - **Callback URL**: `https://<your-n8n-host>/webhook/whatsapp-booking`
   - **Verify token**: a random string you choose (e.g. `my-secret-verify-token`)
3. Activate the webhook and subscribe to the **messages** field
4. Note the **Phone number ID** shown on the WhatsApp → API Setup page
5. The **Send WhatsApp Message** node posts to `https://graph.facebook.com/v21.0/...`.
   Meta deprecates Graph API versions roughly every two years — if you ever get a
   version error from that endpoint, open the Meta docs, take the current version
   (e.g. `v23.0`), and replace `v21.0` in the node's URL.

### 3. Create the Header Auth credential

1. n8n → **Settings → Credentials → Add credential** → **Header Auth**
2. **Name**: `Authorization`
3. **Value**: `Bearer <access token>` (the token from step 2 / API Setup)
4. Attach it to **Send WhatsApp Message**

### 4. Replace the placeholders

In **Verify Challenge**, replace:

```
REPLACE_WITH_VERIFY_TOKEN
```

with the verify token from step 2.

In **Config**, replace:

```
REPLACE_WITH_BUSINESS_NAME
REPLACE_WITH_CALENDAR_ID
REPLACE_WITH_PHONE_NUMBER_ID
```

with the business name, the calendar id (e.g. `primary`, or `me@example.com`), and the phone number ID. Then adjust `openingHour`, `closingHour`, `slotMinutes`, `utcOffsetMinutes` (negative west of UTC; El Salvador is `-360`) and the `services` list to your business. The two credential references on the calendar nodes and the Send node also show `REPLACE_WITH_*` ids/names until you attach your own credentials in the UI.

### 5. Activate the workflow

Toggle the workflow to **Active**. Without activation, the production webhook `…/webhook/whatsapp-booking` is not registered and Meta can't reach it.

## How to test

1. In the Meta Developer Portal, the webhook should now show **Active** (the challenge echo worked)
2. Send a message to the phone number linked to the app: `hola`
3. Answer the menu: `1` (reserve) → service number → a date `DD/MM/AAAA` → a slot number → your name → `1` to confirm
4. Check the calendar: the event appears; the bot confirms and resets to the menu

## Limitations & upgrade path

- **Verify token is a code placeholder** — like the weather workflows, the token string lives in the workflow (see step 4). For a production app, move it to n8n **variables** and reference it via `$vars`.
- **Static data resets on workflow edits** — n8n clears workflow static data when the workflow is updated; in-flight conversations start over. A database or n8n variables per phone is the production upgrade.
- **Time zone by fixed offset** — the slots use a single `utcOffsetMinutes`; daylight saving requires either two offsets or computing local time differently.
- **No retry/dead-letter** — if a handler errors before `Respond OK`, the webhook times out and Meta retries. Add a Try/Catch node around the Google Calendar calls for production.
- **No cancellation of created events** — confirming creates the event; deleting it from the calendar manually is the current "cancel appointment" path.
- **Messages limited to text** — images, locations and interactive buttons (WhatsApp button/list UIs) are left as an exercise; the same sink node handles them once parsed.

## Security

Workflow files contain **no secrets** — tokens live in n8n's encrypted credential store. The verify token and config use `REPLACE_WITH_*` placeholders. If you export this workflow from your instance, credential references export but tokens do not.
