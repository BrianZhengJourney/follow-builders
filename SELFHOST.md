# Self-Hosting Follow Builders (with your own sources)

This fork lets you run the whole pipeline yourself and add your own sources —
including robotics builders, Chinese RSS blogs, and WeChat 公众号 — instead of
relying on the upstream curated feeds.

## How it fits together

```
config/default-sources.json     ← you edit this: who/what to track
        │
        ▼  scripts/generate-feed.js   (GitHub Action, daily)
        ▼     needs secrets: X_BEARER_TOKEN (X only), POD2TXT_API_KEY (podcasts only)
        ▼
feed-x.json / feed-podcasts.json / feed-blogs.json   ← committed to YOUR fork
        │
        ▼  scripts/prepare-digest.js   ← reads feeds from the repo set in config.feedRepo
        ▼
   your digest  (via /ai)
```

## One-time setup

1. **Fork** this repo to your GitHub account.
2. **Point the digest at your fork.** In `~/.follow-builders/config.json` add:
   ```json
   "feedRepo": "YOUR_GH_USERNAME/follow-builders"
   ```
   (Or set the env var `FOLLOW_BUILDERS_FEED_REPO=YOUR_GH_USERNAME/follow-builders`.)
   If unset, it falls back to the upstream `zarazhangrui/follow-builders`.
3. **Enable GitHub Actions** on the fork (Actions tab → enable workflows).
4. **Add repo secrets** (Settings → Secrets and variables → Actions):
   - `POD2TXT_API_KEY` — required only if you keep podcasts.
   - `X_BEARER_TOKEN` — required only if you keep X/Twitter. The X API v2 is paid
     (~$200/mo Basic; the free tier won't pull timelines). Leave the X accounts in
     place but **dormant** until you add this — the workflow just logs a non-fatal
     error and still produces the other feeds.
5. Trigger a run: Actions → "Generate Feeds" → Run workflow. It commits the feed
   JSON files to your fork. Then `/ai` will read them.

> Note: GitHub disables scheduled workflows on a fork after 60 days of no repo
> activity. A commit (or a manual run) re-arms the schedule.

## Source types in `config/default-sources.json`

| Key | Shape | Needs |
|-----|-------|-------|
| `x_accounts` | `{ "name", "handle" }` | X_BEARER_TOKEN |
| `podcasts` | `{ "name", "rssUrl", "url" }` | POD2TXT_API_KEY |
| `blogs` | `{ "name", "indexUrl", ... }` | per-site HTML parser (only Anthropic/Claude wired) |
| `rss_blogs` | `{ "name", "rssUrl", "lang" }` | nothing — generic RSS, works out of the box |

**`rss_blogs` is the general-purpose path.** Any RSS/Atom feed with full text in
`<content:encoded>` (or `<description>`/Atom `<content>`) works: IEEE Spectrum,
Substacks, 机器之心, 知乎专栏, and WeChat-via-bridge. Lookback is 7 days.

## Adding WeChat 公众号

Tencent exposes no native RSS, so you need a **bridge** that turns a 公众号 into an
RSS URL. The bridge must be reachable over a **public URL** (GitHub Actions runs in
the cloud and can't see `localhost`).

Two recommended routes:

- **wechat2rss (hosted, paid, easiest):** subscribe, get a per-account RSS URL,
  drop it straight into `rss_blogs`. ~Zero maintenance.
- **WeWe RSS (self-host, free):** deploy the Docker app to a public host
  (Railway / Fly.io / a VPS), log in with a 微信读书 account (QR scan), subscribe to
  accounts, and it exposes an RSS endpoint per account. Re-scan the QR occasionally.

Either way, once you have the RSS URL:

```json
"rss_blogs": [
  { "name": "机器之心", "rssUrl": "https://your-bridge.example/feed/jiqizhixin", "lang": "zh" }
]
```

If your readers want Chinese output, set `"language": "zh"` (or `"bilingual"`) in
`~/.follow-builders/config.json`.

## Local run (instead of GitHub Actions)

```bash
cd scripts
POD2TXT_API_KEY=... X_BEARER_TOKEN=... node generate-feed.js          # all feeds
node generate-feed.js --blogs-only                                    # blogs/rss only, no tokens
```
Then point `prepare-digest.js` at local files, or commit/push the feeds to your fork.
