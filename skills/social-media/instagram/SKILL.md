---
name: instagram
description: "Instagram Business publishing and insights via the Meta Graph API: post generated images with captions, then read likes, comments and reach."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
prerequisites:
  env: [IG_ACCESS_TOKEN, IG_BUSINESS_ACCOUNT_ID]
metadata:
  hermes:
    tags: [instagram, social-media, meta, graph-api, publishing, insights]
    homepage: https://developers.facebook.com/docs/instagram-platform/content-publishing
---

# Instagram — publish and measure via the Graph API

Publish images to an Instagram **Business** account and read back how each post
performed. Everything here is plain HTTPS against `graph.facebook.com`; run it
with `execute_code` (Python) — no `curl`, no `jq`, no pip installs.

Use this skill when the user asks to post to Instagram, schedule a post, or
check how posts are doing.

## The one hard rule

**Never publish without explicit confirmation in the current conversation.**

Generate the image, draft the caption, show both, and ask. Publishing is
irreversible in the way that matters — followers see it, and deleting a post
does not unsend the notification. "Post it" from a previous turn does not
authorise a second post. When in doubt, use `clarify` and wait.

The only exception is a cron job the user explicitly set up to auto-publish.
Even then, prefer generating a draft and messaging it for approval.

## Credentials

Read from the environment. Never print a token, never write one to a file,
never include one in a message back to the user.

| Variable | In the sandbox? | Purpose |
|---|---|---|
| `IG_ACCESS_TOKEN` | ✅ forwarded | every call below |
| `IG_BUSINESS_ACCOUNT_ID` | ✅ forwarded | the account you post to |
| `IG_PAGE_ID` | ✅ forwarded | rarely needed; the linked Facebook Page |
| `IG_APP_ID` / `IG_APP_SECRET` | ❌ **not** forwarded | host-side token refresh only |

If `IG_ACCESS_TOKEN` is empty, stop and tell the user the account is not
connected yet — do not guess at credentials or try another auth route.

## Shared helper

Every snippet below assumes this preamble:

```python
import json, os, time, urllib.error, urllib.parse, urllib.request

GRAPH = "https://graph.facebook.com/v21.0"
TOKEN = os.environ.get("IG_ACCESS_TOKEN", "")
IG_ID = os.environ.get("IG_BUSINESS_ACCOUNT_ID", "")
if not TOKEN or not IG_ID:
    raise SystemExit("Instagram is not connected: IG_ACCESS_TOKEN / IG_BUSINESS_ACCOUNT_ID are unset.")

def call(method, path, **params):
    """One Graph API call. Raises with Meta's own error text on failure."""
    params["access_token"] = TOKEN
    body = urllib.parse.urlencode(params).encode()
    if method == "GET":
        req = urllib.request.Request(f"{GRAPH}/{path}?{body.decode()}")
    else:
        req = urllib.request.Request(f"{GRAPH}/{path}", data=body, method="POST")
    try:
        with urllib.request.urlopen(req, timeout=60) as r:
            return json.load(r)
    except urllib.error.HTTPError as e:
        detail = e.read().decode("utf-8", "replace")[:600]
        raise SystemExit(f"Graph API {e.code}: {detail}")
```

## Publishing an image

Three calls, in order. Skipping the middle one is the most common cause of a
publish that fails for no visible reason.

```python
IMAGE_URL = "https://..."   # public HTTPS, JPEG — see Image requirements
CAPTION   = "..."

# 1 — create the media container. Instagram downloads the image here.
container = call("POST", f"{IG_ID}/media", image_url=IMAGE_URL, caption=CAPTION)
cid = container["id"]

# 2 — wait for the download to finish. Do NOT skip this.
for _ in range(30):
    status = call("GET", cid, fields="status_code,status").get("status_code")
    if status == "FINISHED":
        break
    if status == "ERROR":
        raise SystemExit(f"Container failed: {call('GET', cid, fields='status')}")
    time.sleep(2)
else:
    raise SystemExit("Container did not finish within 60s.")

# 3 — publish.
published = call("POST", f"{IG_ID}/media_publish", creation_id=cid)
print(json.dumps(published))          # {"id": "<media-id>"}
```

Report the resulting `permalink` back to the user:

```python
info = call("GET", published["id"], fields="permalink,timestamp")
print(info["permalink"])
```

### Image requirements

These are enforced by Meta, not by this skill. Violating them produces vague
errors, so check before you call.

| Requirement | Value | Note |
|---|---|---|
| Format | **JPEG only** | PNG containers are rejected outright |
| Aspect ratio | **0.80 – 1.91** | 4:5 to 1.91:1 |
| Hosting | public **HTTPS** URL | no uploads, no `http://`, no localhost |
| Max size | 8 MB | |
| Container lifetime | 24 h | generate and publish in the same session |

When generating with `image_generate`, pass `aspect_ratio="square"` (1:1) or
`"landscape"` (1.33) only. **Never `"portrait"`** — it maps to 768×1024 = 0.75,
below Instagram's 0.80 floor, and the post will be rejected or hard-cropped.

Once published, Instagram re-hosts the image on its own CDN, so the source URL
may expire afterwards without affecting the post.

### Caption rules

- 2,200 characters max; the first ~125 show before "more" — put the hook there.
- 30 hashtags max. Exceeding this silently strips them all.
- `@mentions` notify real accounts. Only use handles the user gave you.
- Line breaks survive. Emoji survive.

## Reading performance

For **likes and comments**, read the fields directly. This is more reliable
than the Insights API, whose metric names Meta changes between versions.

```python
feed = call("GET", f"{IG_ID}/media",
            fields="id,caption,permalink,timestamp,media_type,like_count,comments_count",
            limit=25)
for m in feed.get("data", []):
    print(m["timestamp"], m.get("like_count", 0), m.get("comments_count", 0), m["permalink"])
```

For **reach and saves**, use Insights on a single media id:

```python
ins = call("GET", f"{media_id}/insights", metric="reach,saved,shares,total_interactions")
for row in ins.get("data", []):
    print(row["name"], row["values"][0]["value"])
```

If a metric name errors, drop it and retry with the rest — Meta deprecates
individual metrics without removing the endpoint. Never invent a metric name.

To read the actual comment text:

```python
comments = call("GET", f"{media_id}/comments", fields="text,username,timestamp", limit=50)
```

## Scheduled reporting

Use the `cronjob` toolset. A daily digest is the useful default:

> Every morning at 9, read the last 7 days of Instagram posts, and message me
> likes, comments and reach per post plus which one did best and why.

Keep cron jobs read-only unless the user explicitly asked for auto-publishing.
A cron job that posts without a human in the loop should be rare and named as
such when you create it.

## Token refresh — every 60 days

`IG_ACCESS_TOKEN` expires 60 days after it is issued. When it lapses, every
call above returns a `190` error and posting stops silently.

Refreshing needs `IG_APP_SECRET`, which is **not** available in the sandbox by
design. So this cannot be run by the agent — tell the user to run it on the
host and paste the new value into `~/.hermes/.env`:

```
GET https://graph.facebook.com/v21.0/oauth/access_token
    ?grant_type=fb_exchange_token
    &client_id=$IG_APP_ID
    &client_secret=$IG_APP_SECRET
    &fb_exchange_token=$IG_ACCESS_TOKEN
```

The old token must still be valid. Once expired, the whole Graph API Explorer
flow has to be repeated.

## Errors

| Code | Meaning | What to do |
|---|---|---|
| `190` | token expired or revoked | Refresh (above). Do not retry. |
| `100` subcode `2207052` | aspect ratio out of range | Regenerate square or landscape |
| `100` subcode `2207003` | image could not be downloaded | URL not public HTTPS, or expired |
| `100` subcode `2207004` | format rejected | Not JPEG |
| `9` / `4` | rate limited | Back off; 25 posts per 24 h, ~200 calls/h |
| `10` / `200` | missing permission | App lacks `instagram_content_publish` |
| `24` | container not ready | You skipped step 2 |

Surface Meta's own error text to the user. Do not paraphrase it into something
friendlier — the subcode is what makes it diagnosable.

## Where this changes for production

`IMAGE_URL` is the single seam. Today it is whatever `image_generate` returns
(a FAL-hosted URL). Swapping to S3 + CloudFront means changing only how that
one variable is produced — upload the generated file, use the CDN URL. Nothing
else in this skill moves.

The credential model, however, does **not** carry over. Forwarding
`IG_ACCESS_TOKEN` into the agent's sandbox is acceptable for a single throwaway
account. With real client accounts the token must live in a backend service
that the agent can request a publish from, never in the agent's own execution
environment.
