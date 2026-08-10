# Social Cross-Poster — Facebook, Instagram, X, LinkedIn

A community manager fills in the post once — text, an optional image URL, and hashtags — and this workflow publishes it to a Facebook Page, an Instagram Business account (via the Meta Graph API), X, and LinkedIn. Each platform gets its own text adaptation (X is capped at 280 characters, Instagram captions at 2200), and each platform fails independently without blocking the others: `onError: continueRegularOutput` keeps the run going and a final Code node summarizes what actually happened.

```
Manual Trigger
  -> Prepare Post (Code)                        # text + image URL + hashtags + account IDs
  -> Has Image? (IF)
       |-> Facebook: Post Photo (HTTP) ①          [has image]
       |     -> Instagram: Create Container (HTTP) ①
       |     -> Instagram: Publish Container (HTTP) ①
       |-> Facebook: Post Text (HTTP) ①            [no image]
       |-> Merge Branches (Merge, append) ②        <-- both branches rejoin here
  -> X: Post Tweet (X node) ①
  -> LinkedIn: Post Update (HTTP) ①
  -> Post Summary (Code)

① onError: continueRegularOutput — the node reports its own
  failure and the workflow keeps going.
② Merge re-joins the image and no-image branches so X and
  LinkedIn always run exactly once.
```

## What it does

- **One post, four platforms** — text, optional image URL, and hashtags are written once in the **Prepare Post** Code node and adapted per platform: X is truncated to 280 characters, Instagram captions to 2200; Facebook and LinkedIn keep the full text.
- **Two Meta paths** — with an image, Facebook and Instagram get a *photo post* (Graph API two-step: create a container, then publish it); without one, Facebook gets a *text post* and Instagram is skipped.
- **Independent failures** — every publishing node runs with `onError: continueRegularOutput`, so a bad LinkedIn token or an expired X scope never stops the other platforms.
- **A status summary** — the final Code node reports `ok` / `failed` / `skipped` per platform, with the error detail when something went wrong.

## Concepts it teaches

- **Manual trigger** — the workflow runs when you click **Execute Workflow**; the content lives in the Code node, not in the trigger.
- **A Code node as a single content/config source** — **Prepare Post** holds both the post and the account IDs, so the HTTP nodes only ever reference `$('Prepare Post').item.json...` instead of each having hardcoded values.
- **Per-platform text adaptation** — the same source text becomes four strings: X is truncated to 280 characters, Instagram captions to 2200, while Facebook and LinkedIn keep the full text with hashtags.
- **IF branching and re-joining with a Merge (append)** — **Has Image?** splits the run; **Merge Branches** appends both branches back into a single line so X and LinkedIn always execute exactly once.
- **`onError: continueRegularOutput`** — a failing node marks itself failed but hands its input onward; one platform failing never blocks the others. That's why the summary node can still run at the end.
- **Native node vs raw HTTP Request** — X uses the native node because it handles the OAuth2 signing for you; the Meta Graph API (two-step photo publishing) and LinkedIn UGC need the exact payload shape, so they use raw HTTP Request with their own credentials.

## Prerequisites

- **Meta (Facebook + Instagram)**
  - A Facebook Page and an Instagram **Professional** (Business or Creator) account connected to that Page — Instagram publishing via the Graph API only works for Business accounts.
  - A Meta developer app with permissions: `pages_read_engagement`, `pages_manage_posts`, `instagram_basic`, `instagram_content_publish`.
  - The image URL must be **publicly reachable** — Meta downloads it server-side, so `localhost` or private URLs will fail.
- **X** — a developer app on a **paid tier** (Basic or higher); the free tier is read-only and cannot post. Scopes: `tweet.read`, `tweet.write`, `users.read`, `offline.access`.
- **LinkedIn** — a developer app with the **Share on LinkedIn** product (or the Marketing Developer Platform + UGC posting). Scopes: `w_member_social` (person) or `w_organization_social` (organization).

## Setup (one time, ~15 minutes)

### 1. Create the three credentials

n8n menu → **Settings → Credentials → Add credential**:

- **Facebook Graph API** (for the four Meta nodes)
- **X (Twitter) OAuth2 API**
- **LinkedIn OAuth2 API**

### 2. Attach them to the nodes

- **Facebook Graph API** → **Facebook: Post Photo**, **Instagram: Create Container**, **Instagram: Publish Container**, **Facebook: Post Text**
- **X (Twitter) OAuth2 API** → **X: Post Tweet**
- **LinkedIn OAuth2 API** → **LinkedIn: Post Update**

Until you attach them, the nodes show `REPLACE_WITH_*` ids/names.

### 3. Replace the placeholders in Prepare Post

Replace:

```
REPLACE_WITH_POST_TEXT
REPLACE_WITH_IMAGE_URL_OR_LEAVE_EMPTY
REPLACE_WITH_FACEBOOK_PAGE_ID
REPLACE_WITH_INSTAGRAM_BUSINESS_USER_ID
REPLACE_WITH_LINKEDIN_AUTHOR_URN
```

with the post text, the image URL (leave empty for text-only), the Facebook Page ID, the Instagram Business User ID, and the LinkedIn author URN (`urn:li:person:xxx` for a personal profile, `urn:li:organization:xxx` for a company page).

**Finding the Instagram Business User ID** — Meta Business Suite → **Settings**, or via the Graph API:

```
GET /{page-id}?fields=instagram_business_account
```

## How to test

1. Run with **no image** (leave the image URL empty): text-only path — Facebook feed post + X tweet + LinkedIn update; Instagram shows `skipped` in the summary.
2. Run with an **image**: both Meta branches run; Instagram does the container → publish two-step.
3. Break something on purpose (for example a wrong LinkedIn author URN) to see the other platforms still publish and the summary mark that platform `failed`.

## Security

Workflow files contain **no secrets** — tokens live in n8n's encrypted credential store. All credentials here are `REPLACE_WITH_*` placeholders. If you export this workflow from your instance, credential references export but tokens do not.
