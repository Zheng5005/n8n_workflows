# Invoice Backup — Gmail to Google Drive

Watches Gmail for unread emails with PDF attachments and invoice-like subjects, saves each PDF into a monthly folder in Google Drive, then labels the email and marks it read.

```
Gmail Trigger (polls, unread, subject keywords, downloads attachments)
  -> Prepare PDF Attachments (Code: keeps only PDFs, one item per file)
  -> Carry Binary as JSON (MoveBinaryData: file -> base64 string)
       |----> Find Month Folder (HTTP GET to Drive API)  [folder lookup]
       |         -> Month Folder Exists? (IF)
       |              [yes] -> (use it)
       |              [no]  -> Create Month Folder
       |         -> Combine Folder ID (Merge: re-join data + folder id)
  -> Restore Binary (base64 string -> real file)
  -> Upload Invoice (Google Drive)
  -> Add Invoice Backup Label (Gmail)
  -> Mark as Read (Gmail)
```

## What it does

Every 5 minutes (configurable), the trigger polls Gmail for **unread** emails matching `has:attachment subject:(invoice OR factura)`. For each **PDF** attachment:

1. Finds (or creates) a `YYYY-MM` folder inside your **Invoice Backups** folder in Drive
2. Uploads the PDF there, keeping its original file name
3. Adds the **Invoice Backup** label to the email and marks it read

Emails without PDFs are silently ignored. Once read+labeled, an email is never processed again — that's the built-in dedupe.

## Concepts it teaches

- **Polling triggers with a search query** — the `q` filter runs inside Gmail itself, so only candidate emails are fetched
- **Binary data** — attachments live in `attachment_0`, `attachment_1`, ... binary properties
- **MoveBinaryData** — the trick for carrying files across nodes that drop them (HTTP, Drive): file becomes a base64 string in JSON, restored later
- **Folder auto-creation** — why this uses a raw Drive API call instead of the Search node (see the sticky note in the editor): an empty search result would silently end the branch
- **Merge node** — re-joins two parallel branches back into one flow
- **Google OAuth2 credentials** — the real-world setup for any Google integration

## Setup (one time, ~10 minutes)

### 1. Google Cloud: project + APIs

1. Go to https://console.cloud.google.com and create a project (or reuse one)
2. **APIs & Services → Library** → enable **Gmail API** and **Google Drive API**
3. **APIs & Services → OAuth consent screen** → External → add yourself as test user → save
4. **APIs & Services → Credentials → Create Credentials → OAuth client ID**
   - Application type: **Web application**
   - Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
   - Copy the **Client ID** and **Client secret**

### 2. n8n credentials (two, same OAuth client)

1. **Settings → Credentials → Add credential → Gmail OAuth2 API**
   - Paste Client ID + Secret → click **Sign in with Google** → approve (Gmail scopes are pre-filled)
2. **Settings → Credentials → Add credential → Google Drive OAuth2 API**
   - Same Client ID + Secret → approve (Drive scopes pre-filled)

These are the two credential types this n8n version uses — Gmail and Drive are deliberately separate.

### 3. Wire the placeholders

| Placeholder | Where | Replace with |
|---|---|---|
| `REPLACE_WITH_GMAIL_CREDENTIAL_ID/NAME` | Gmail Trigger, Add Label, Mark as Read nodes | the Gmail credential (credential dropdown) |
| `REPLACE_WITH_DRIVE_CREDENTIAL_ID/NAME` | Find Month Folder, Create Month Folder, Upload nodes | the Drive credential |
| `REPLACE_WITH_INVOICE_BACKUPS_FOLDER_URL` | **Create Month Folder** node → Folder | URL of a folder you create in Drive, e.g. create **Invoice Backups** in your Drive and copy `https://drive.google.com/drive/folders/<id>` |
| `REPLACE_WITH_LABEL_ID` | **Add Invoice Backup Label** node → Label | select your label in the dropdown (create it first in Gmail: Settings → Labels → Create new label → "Invoice Backup") |

### 4. Adjust the invoice keywords (optional)

In the **Gmail Trigger** node, the search query is:

```
has:attachment subject:(invoice OR factura)
```

Edit `invoice OR factura` to match your senders' subject style. Gmail search syntax supports `subject:(a OR b)`, `from:someone@example.com`, etc.

## How to test

1. Import `workflow.json` (Menu → **Import from File**) — credentials will show red dots until attached (step 3)
2. Send yourself an email with **"invoice" in the subject** and a **PDF attachment**
3. Open the **Gmail Trigger** node → **Execute node** (in manual test mode the trigger processes only the most recent matching message — in production it handles up to 10 per poll)
4. Watch the steps: the PDF should appear in `Invoice Backups/2026-08/`, and the email gets labeled + marked read

If an email doesn't have a PDF (e.g. only a .docx), the workflow ends quietly at **Prepare PDF Attachments** — that's the intended "ignore" path.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Trigger shows "Invalid credentials" | Credential not attached or OAuth not approved — redo step 2 and re-sign-in |
| `Error: No binary data` at Upload | The attachment wasn't downloaded — check the trigger has **Download Attachments** on (it is in this file) |
| Folder never created / file uploaded to wrong place | The **Invoice Backups** folder URL placeholder wasn't replaced, or the Drive credential isn't attached to **Create Month Folder** |
| Label error | The `REPLACE_WITH_LABEL_ID` wasn't replaced — open the node and pick the label from the dropdown |
| Same email processed again | It wasn't marked read (check the Mark as Read node ran) |

## Security

Like every workflow in this repo, `workflow.json` contains **no secrets**: tokens live in n8n's credential store, and the file only references them by placeholder id/name. Safe to commit to a public repo.
