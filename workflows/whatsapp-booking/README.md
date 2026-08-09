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
                |       |-> Reserve Slot (Postgres, UNIQUE) -> IF Inserted?
                |       |       |-> Create Booking (Google Calendar) -> Booking Succeeded? (IF)
                |       |       |       |-> Booking Result -> Send WhatsApp Message      [created]
                |       |       |       |-> Release Reservation -> Booking Failed -> Send  [create failed]
                |       |       |-> Re-offer Slots -> (re-enters Check Day Availability)
                |       |-> Send WhatsApp Message            [cancelled / re-ask]
                |-> Respond OK                               [status events, no text]
```

Every message branch ends in the **single HTTP sink** `Send WhatsApp Message` → `Respond OK`, so the webhook always answers 200 and WhatsApp never retries an already-handled message.

The `Reserve Slot` → `IF Inserted?` step is the concurrency guard: on confirm the slot is atomically reserved in a PostgreSQL `reservations` table (UNIQUE primary key) *before* the calendar event is created. Two customers confirming the same slot at the same time can never both win — the loser's insert fails and `Re-offer Slots` sends them the next free slots, re-entering the existing availability chain. The `reservations` table is created and pruned by the companion `cleanup-workflow.json` (schedule trigger, every 15 minutes).

## What it does

- **Verification challenge** — Meta verifies the webhook with a `GET` containing `hub.verify_token`; the workflow echoes `hub.challenge` back so the subscription activates.
- **Conversation state per phone** — each phone number gets its own `{ step, data }` object in `$getWorkflowStaticData('global')`, so a dozen customers can be mid-conversation at the same time with no database.
- **Real availability** — the Google Calendar *Availability* node returns the busy ranges for the chosen day, and a Code node computes the free time slots (service duration, business hours, time zone offset).
- **One-click booking** — on confirmation the *Create* operation adds the event to the calendar, then the bot confirms the booking and resets the conversation to the menu.
- **Double-booking guard** — before the event is created, the slot is atomically reserved in a PostgreSQL table; a UNIQUE primary key makes two simultaneous bookings of the same slot impossible, and the loser is offered the remaining free slots.

## Concepts it teaches

- **Webhook with two HTTP methods** — `multipleMethods` + `httpMethod: [GET, POST]` gives the webhook node two outputs: output 0 fires on GET, output 1 on POST.
- **Respond to Webhook node** — with the webhook's *Response Mode* set to *Using Respond to Webhook Node*, the reply is decided by a node deep in the flow (the challenge echo, or a silent 200).
- **`$getWorkflowStaticData('global')`** — n8n's built-in key-value store for workflows; mutations persist across executions. Here it is keyed by phone number.
- **A Code node as a state machine** — the current step is just a string; a Switch node routes each step to its own handler.
- **Slots math in UTC** — the calendar works in UTC; the bot shifts by the business time zone offset and shows the customer local times.
- **A database constraint as a concurrency guard** — the UNIQUE primary key on `reservations` turns a race into a clean error: the losing insert fails, the node is set to continue on error, `IF Inserted?` receives the input without an `available` field and routes to `Re-offer Slots`.
- **Continue-on-error plus a success-marker branch** — `Reserve Slot` and `Create Booking` run with `onError: continueRegularOutput`; after `Create Booking` an `IF` on `$json.status === 'confirmed'` routes to the confirmation or to `Release Reservation`. (n8n's `continueErrorOutput` port is only populated when a node *returns* error-shaped data, not when it throws, so it cannot drive rollback branches.)

## Prerequisites

- n8n reachable at a **public HTTPS** URL (required by Meta).
- A **Meta developer app** with a WhatsApp Business number, its **phone number ID**, and an **access token**.
- A **Google Calendar** (a normal personal calendar is fine).
- A **PostgreSQL** database — the bundled `docker-compose.yml` starts n8n and Postgres together.
- Three n8n credentials: **Google Calendar OAuth2**, **Header Auth**, and **Postgres**.

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

with the business name, the calendar id (e.g. `primary`, or `me@example.com`), and the phone number ID. Then adjust `openingHour`, `closingHour`, `slotMinutes`, `utcOffsetMinutes` (negative west of UTC; El Salvador is `-360`) and the `services` list to your business. The credential references on the calendar nodes, the Send node, and the Postgres nodes also show `REPLACE_WITH_*` ids/names until you attach your own credentials in the UI.

### 5. Start PostgreSQL and connect it

The concurrency guard needs a database. The bundled `docker-compose.yml`
starts n8n and Postgres together:

```bash
cd workflows/whatsapp-booking
# edit REPLACE_WITH_DB_PASSWORD in docker-compose.yml first
docker compose up -d
```

Then:

1. n8n → **Settings → Credentials → Add credential** → **Postgres**
   - Host: `postgres`, Port: `5432`, Database: `n8n_booking`
   - User: `n8n`, Password: the one you set in `docker-compose.yml`
2. Attach the credential to **Reserve Slot** and **Release Reservation**
   in this workflow, and to **Ensure Table** and **Prune Stale** in
   `cleanup-workflow.json`
3. Import `cleanup-workflow.json` (Repository → **Import from File**)
   and activate it — it creates the `reservations` table on first run
   and prunes stale pending rows every 15 minutes. To accept bookings
   immediately (rather than waiting up to 15 minutes), run the cleanup
   workflow once manually right after importing it.

### 6. Activate the workflows

Toggle **this** workflow to **Active** (without it, the production webhook `…/webhook/whatsapp-booking` is not registered and Meta can't reach it) and the **cleanup** workflow to Active.

## How to test

1. In the Meta Developer Portal, the webhook should now show **Active** (the challenge echo worked)
2. Send a message to the phone number linked to the app: `hola`
3. Answer the menu: `1` (reserve) → service number → a date `DD/MM/AAAA` → a slot number → your name → `1` to confirm
4. Check the calendar: the event appears; the bot confirms and resets to the menu

To see the concurrency guard in action, confirm the same slot from two
phone numbers at the same time: the first booking wins, the second gets
a "recently reserved" message with the remaining slots.

## Limitations & upgrade path

- **Verify token is a code placeholder** — like the weather workflows, the token string lives in the workflow (see step 4). For a production app, move it to n8n **variables** and reference it via `$vars`.
- **Static data resets on workflow edits** — n8n clears workflow static data when the workflow is updated; in-flight conversations start over. A database or n8n variables per phone is the production upgrade.
- **Time zone by fixed offset** — the slots use a single `utcOffsetMinutes`; daylight saving requires either two offsets or computing local time differently.
- **No retry/dead-letter** — the calendar failure is answered with an apology, but other errors before `Respond OK` still time out the webhook and Meta retries. For production, wrap the create step in an explicit error-handling chain (the pattern `Booking Succeeded?` after `Create Booking` demonstrates) and add monitoring on workflow executions.
- **Reservations are a guard, not a ledger** — pending rows are pruned after 30 minutes and confirmed bookings exist only as calendar events. For appointment history, reminders, or reporting, write confirmed bookings to a real `bookings` table instead of only guarding with `reservations`.
- **No cancellation of created events** — confirming creates the event; deleting it from the calendar manually is the current "cancel appointment" path.
- **Messages limited to text** — images, locations and interactive buttons (WhatsApp button/list UIs) are left as an exercise; the same sink node handles them once parsed.

## Security

Workflow files contain **no secrets** — tokens live in n8n's encrypted credential store. The verify token and config use `REPLACE_WITH_*` placeholders. If you export this workflow from your instance, credential references export but tokens do not.
