# amber — Architecture Document

## System Overview

amber is a Chrome browser extension paired with a Go server that creates clean, permanently offline-capable archives of web pages. It strips all tracking code, classifies each stripped snippet by tactic type using an on-device LLM, and pushes findings to a public tracker database. The system is composed of four major layers:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Chrome Browser (User's Session)                                     │
│                                                                      │
│  ┌─────────────┐   ┌───────────────────────────────────────────┐    │
│  │  popup.html │   │  content.js (injected into active tab)    │    │
│  │  popup.js   │◄──│  - Serializes rendered DOM                │    │
│  └──────┬──────┘   │  - Strips scripts / trackers / handlers   │    │
│         │          │  - Collects asset URLs + raw snippets     │    │
│         │          └─────────────────┬─────────────────────────┘    │
│         │                            │ chrome.runtime.sendMessage    │
│         │          ┌─────────────────▼─────────────────────────┐    │
│         └──────────│  background.js (Service Worker)            │    │
│                    │  - Fetches assets with session cookies     │    │
│                    │  - Classifies snippets via Gemini Nano     │    │
│                    │  - POSTs archive package to Go server      │    │
│                    └─────────────────┬─────────────────────────┘    │
└──────────────────────────────────────│──────────────────────────────┘
                                       │ HTTPS POST (API key auth)
                         ┌─────────────▼─────────────────────┐
                         │  Go Server (natsec / korh.one)    │
                         │                                   │
                         │  POST /api/v1/archive             │
                         │  POST /api/v1/snippets            │
                         │  GET  /api/v1/archives            │
                         │  GET  /health                     │
                         │                                   │
                         │  ┌────────────┐  ┌─────────────┐ │
                         │  │Archive     │  │Snippet      │ │
                         │  │Writer      │  │Processor    │ │
                         │  │(disk)      │  │(SQLite)     │ │
                         │  └────────────┘  └──────┬──────┘ │
                         └─────────────────────────│────────┘
                                                   │ GitHub API push
                              ┌────────────────────▼──────────────────┐
                              │  GitHub (spacedudem/amber-tracker-db) │
                              │  tracker_tactics.json                 │
                              │  exports/ublock_filters.txt           │
                              │  exports/hosts.txt                    │
                              │  exports/disconnect.json              │
                              │  exports/pihole.txt                   │
                              └────────────────────┬──────────────────┘
                                                   │ GitHub Actions (on push)
                              ┌────────────────────▼──────────────────┐
                              │  GitHub Pages                         │
                              │  Public filter lists (live URLs)      │
                              │  Browsable tracker tactic catalog     │
                              └───────────────────────────────────────┘
```

---

## Component Architecture

### Chrome Extension

#### Manifest V3 Design

amber uses Manifest V3 (MV3), Chrome's current extension platform, not the deprecated MV2. This is a deliberate constraint, not a fallback.

MV3 replaces the persistent background page with a service worker. Service workers are ephemeral — they spin up to handle an event, then terminate. This fits amber's use case exactly: capture is a one-shot event, not a continuous background process. There is no need for a persistent page consuming memory while the user browses normally.

MV3 removes the ability to use blocking `webRequest` (replaced by `declarativeNetRequest`). amber does not intercept requests — it captures an already-loaded page — so this restriction has no impact.

MV3 enforces stricter Content Security Policy by default. amber's extension code does not use `eval()`, inline scripts executed from strings, or remote code injection, so it is already MV3-compliant by design.

Required permissions and rationale:

| Permission | Reason |
|---|---|
| `tabs` | Read the URL and tab ID of the active tab to build archive path and send messages |
| `scripting` | Inject `content.js` into the active tab on demand (not at page load) |
| `storage` | Persist user settings: configured server URL, API key, capture count |
| `cookies` | Pass first-party cookies to `fetch()` calls in background.js for asset retrieval |
| `<all_urls>` (host permission) | Required for `scripting.executeScript` injection and `fetch()` with cookies across arbitrary domains |
| `alarms` | Schedule retry of failed POSTs if the server is temporarily unreachable |

No `webRequest`, no `declarativeNetRequest` — amber is a reader, not a blocker.

Origin Trial: the Chrome Prompt API (`window.ai.languageModel`) that gates access to Gemini Nano is currently behind an Origin Trial. For local development and personal use, the flag `#optimization-guide-on-device-model` in `chrome://flags` enables it without a token. For Chrome Web Store distribution, an Origin Trial token obtained from Google is embedded in `manifest.json` under the `trial_tokens` field. The token is tied to the extension's origin and has a fixed expiry date; it must be renewed with each Origin Trial renewal period.

---

#### content.js — DOM Serializer and Stripper

`content.js` is injected into the active tab via `chrome.scripting.executeScript()` when the user clicks Capture. It runs in the page's document context, with access to the live, fully-rendered DOM after all JavaScript has executed.

Responsibilities:

1. **DOM Serialization**: Captures `document.documentElement.outerHTML`. This is the post-JavaScript rendered state — React components have mounted, lazy-loaded content has been injected, single-page app routing has settled. The serialized string includes all markup that the browser actually built, not what the server sent.

2. **Strip Pipeline** (executed in order on a parsed DOM copy, not the live DOM):

   **Step 1 — Remove all `<script>` elements**: `document.querySelectorAll('script')` → remove each. This covers both inline scripts (`<script>alert(1)</script>`) and external script tags (`<script src="..."></script>`). External scripts whose content was already executed do not need to be re-fetched; the results of their execution are already in the DOM.

   **Step 2 — Remove all `<noscript>` elements**: These exist to serve fallback content (often tracking pixels or alternate beacons) to non-JS environments. In a static archive where no JavaScript executes, `<noscript>` content would be rendered as visible content, which is misleading — the archived page was captured in a JS-enabled context. All `<noscript>` elements are removed unconditionally.

   **Step 3 — Remove inline event handlers**: All `on*` attributes are stripped from every element. This is done by iterating all elements and removing any attribute whose name matches `/^on[a-z]/i`. Covers `onclick`, `onload`, `onmouseover`, `onsubmit`, `onerror`, `onblur`, `onkeydown`, and every other DOM event attribute. This is essential — many trackers inject handlers at initialization that would fire on interaction even without a `<script>` tag.

   **Step 4 — Remove tracking pixels**: `<img>` elements with `width="0"`, `height="0"`, `width="1"`, or `height="1"` (or equivalent inline style) are identified and removed. Additionally, `<img>` elements whose `src` matches a known tracker domain list (embedded in the extension as a compact JSON array of second-level domains) are removed. Both conditions are checked independently so either one triggers removal.

   **Step 5 — Remove `<iframe>` elements**: Iframes are almost universally used for ad containers, third-party widgets, and cross-origin tracker sandboxes in modern pages. All iframes are stripped. If a page uses a legitimate same-origin iframe (rare), the stripped archive loses that content — this is an acceptable tradeoff for the privacy guarantee.

   **Step 6 — Strip tracking data attributes**: Remove all attributes matching `data-tracking-*`, `data-analytics-*`, `data-gtm-*`, `data-ga-*`, and similar patterns. These attributes carry user session tokens, A/B test variant assignments, and other surveillance metadata that would expose the archived user's session context. Matched by regex against all element attribute names.

   **Step 7 — Collect asset URLs**: Walk the stripped DOM and collect every URL that references an external resource the archive needs to be self-contained: `<img src>`, `<link href>` (stylesheets), `@font-face` URLs in `<style>` blocks and inline style attributes, `background-image` values in inline styles, `<source srcset>` values, and `<video>/<audio> src` values. Deduplicate. Return as `string[]`.

3. **Snippet Collection**: Before stripping, for each removed element or attribute, record the raw value (script text content, handler attribute value, pixel src URL, iframe src URL) plus minimal context (element tag name, parent tag name, DOM depth). These become the input to Gemini Nano classification. Snippets are truncated to 2000 characters each to avoid context window exhaustion when batched.

4. **Return Value**: content.js sends a `CAPTURE_COMPLETE` message via `chrome.runtime.sendMessage()`:
   ```json
   {
     "type": "CAPTURE_COMPLETE",
     "payload": {
       "url": "https://example.com/article",
       "capturedAt": "2026-06-16T00:00:00Z",
       "pageTitle": "Article Title",
       "html": "<stripped HTML string>",
       "assetUrls": ["https://...", "https://..."],
       "strippedSnippets": [
         {
           "rawSnippet": "...",
           "context": "<script>!function(){...}</script>",
           "elementType": "script",
           "attributeName": null
         }
       ]
     }
   }
   ```
   See API_SPEC.md section "Extension ↔ Server Message Protocol" for the authoritative CAPTURE_COMPLETE message schema.

content.js does not fetch assets. Chrome's extension content script sandbox prevents synchronous binary fetches at scale and cannot access cookies for cross-origin requests in the same way the service worker can. Asset fetching is delegated entirely to background.js.

---

#### background.js — Orchestrator (Service Worker)

`background.js` is the extension's central coordinator. It runs as a service worker, activated by a message from content.js and terminating after the capture pipeline completes.

Responsibilities:

**Asset Fetching**: For each URL in the `assets` array, background.js calls `fetch(url, { credentials: 'include' })`. Because the service worker shares the browser's cookie store for the relevant origin, assets behind authentication (CDN-signed URLs, paywalled images, login-gated resources) are fetched successfully. Responses are read as `ArrayBuffer` and base64-encoded for JSON transport to the server. Content-type is preserved from the `Content-Type` response header.

Asset fetching is batched: a maximum of 6 concurrent fetches run at any time using a semaphore pattern to avoid exhausting available TCP connections. Failed fetches (4xx, 5xx, network error) are logged and skipped; the resulting archive will have broken image references for those assets, which is acceptable.

**Gemini Nano Classification**: The Chrome Prompt API is accessed via `window.ai.languageModel` (the `window.ai` namespace is available in extension service workers when the Origin Trial token is present).

Capability check:
```javascript
const capabilities = await window.ai.languageModel.capabilities();
if (capabilities.available === 'no') {
  // Gemini Nano not available; mark all snippets tactic_type: 'unknown'
  return snippets.map(s => ({ ...s, tactic_type: 'unknown' }));
}
```

Session creation:
```javascript
const session = await window.ai.languageModel.create({
  systemPrompt: SYSTEM_PROMPT,
  temperature: 0.1,
  topK: 3
});
```

`SYSTEM_PROMPT` instructs the model to classify each snippet as exactly one of: `canvas_fingerprint`, `webgl_fingerprint`, `font_fingerprint`, `audio_fingerprint`, `battery_api`, `beacon`, `pixel`, `localStorage_abuse`, `sessionStorage_tracking`, `indexeddb_tracking`, `cookie_sync`, `cname_cloak`, `service_worker_tracking`, `inline_handler`, `data_attribute`, `css_tracking`, `third_party_loader`, `behavioral`, `network_timing`, or `unknown`. It specifies JSON output format with fields `tactic_type`, `severity`, and `confidence`.

Chunking: snippets are concatenated into batches not exceeding 3000 tokens (estimated at 4 characters per token, so ~12000 characters per batch). Each batch is sent as a separate `session.prompt()` call. JSON output is parsed; malformed JSON triggers a retry with a stricter prompt. After 2 failed retries, affected snippets in that batch are marked `unknown`.

**Server POST**: After asset fetching and classification complete, background.js constructs a JSON payload:
```json
{
  "url": "https://example.com/article",
  "captured_at": "2026-06-16T00:00:00Z",
  "amber_version": "1.0.0",
  "page_title": "Article Title",
  "html": "<stripped HTML string>",
  "assets": [
    {
      "original_url": "https://cdn.example.com/logo.png",
      "content_type": "image/png",
      "data": "<base64>",
      "size_bytes": 12345
    }
  ],
  "snippet_count": 8
}
```

Classified snippets are submitted separately in a follow-up `POST /api/v1/snippets` call after the archive is saved. The payload is POSTed to `${serverUrl}/amber/api/v1/archive` with header `X-Amber-Key: <configured API key>`. On HTTP 201 Created response, the `archive_id` and `archive_url` from the response body are passed back to the popup. On failure, background.js schedules a retry via `chrome.alarms.create()` with exponential backoff (up to 3 attempts: 2s, 4s, 8s for server errors; 5s, 10s, 20s for network errors), storing the payload in `chrome.storage.local` for the retry attempt.

**Badge Updates**: Throughout the pipeline, background.js updates `chrome.action.setBadgeText` to show progress: "..." during capture, a checkmark on success, "!" on error.

---

#### popup.html / popup.js

The popup is intentionally minimal. It shows:

- The current tab's URL (truncated to 60 characters)
- A "Capture" button that triggers `chrome.tabs.query` → `chrome.scripting.executeScript` to inject content.js
- A progress indicator (spinner + status text string updated via message from background.js)
- On success: a clickable link to the archived page on the server
- A settings icon linking to a full options page for server URL and API key configuration

The popup has no persistent state beyond what is in `chrome.storage.local`. It closes and reopens cleanly at any point without corrupting a capture in progress (the service worker continues independently).

---

### Go Server

The server is a single Go binary exposing an HTTP API over a Unix socket, reverse-proxied by nginx on the natsec host. It handles archive persistence, SQLite operations, and GitHub synchronization.

#### Directory Structure

```
server/
  main.go              — HTTP server setup, routing, graceful shutdown
  handlers/
    archive.go         — POST /api/v1/archive handler
    snippets.go        — POST /api/v1/snippets handler (standalone endpoint)
    archives.go        — GET /api/v1/archives list handler
    health.go          — GET /health handler
  storage/
    archive.go         — write HTML + assets to disk, build index.html
    db.go              — SQLite schema init + CRUD operations
  github/
    push.go            — commit + push tracker_tactics.json to GitHub
    export.go          — generate uBlock/hosts/disconnect/pihole exports
  config/
    config.go          — env var configuration with defaults
  middleware/
    auth.go            — X-Amber-Key validation middleware
    logger.go          — structured request logging
```

#### HTTP Handlers

**POST /api/v1/archive**

Accepts the JSON payload from background.js. Processing steps:

1. Validate `X-Amber-Key` header against `AMBER_API_KEY` env var.
2. Decode and validate request JSON. Reject payloads over 50MB (configurable via `AMBER_MAX_PAYLOAD_MB`).
3. Sanitize `url`: parse as URL, extract domain and path, sanitize path components for filesystem safety (remove `..`, replace non-alphanumeric with `-`).
4. Call `storage.WriteArchive()`: writes base64-decoded HTML to `<base>/<domain>/<path>/index.html`, writes each asset to `<base>/<domain>/<path>/_assets/<sha256-of-url>.<ext>`. Rewrites asset URLs in the HTML from absolute CDN URLs to relative paths pointing to the `_assets/` directory.
5. Write `_amber-meta.json` alongside the archive with: source URL, capture timestamp, asset count, snippet count, extension version.
6. Call `storage.StoreSnippets()` to write snippets to SQLite.
7. Call `github.PushUpdate()` asynchronously (non-blocking — archive response is returned immediately, GitHub sync happens in background goroutine).
8. Return HTTP 201 with `{ "archive_id": <integer>, "archive_url": "https://korh.one/amber/archives/<domain>/<path>/", "file_path": "<absolute-path>", "assets_saved": N, "assets_failed": N }`.

**POST /api/v1/snippets**

A standalone endpoint for submitting snippet batches without a full archive (future use: allows browser history analysis tools to submit snippets from pages they visited without capturing the full HTML). Uses the same SQLite write path as the archive handler's snippet step. Validates the API key, accepts JSON array of snippet objects, returns `{ "stored": N }`.

**GET /api/v1/archives**

Returns a JSON array of archive metadata from `_amber-meta.json` files on disk. Supports `?domain=` filter and `?limit=` / `?offset=` pagination. Response is cached in memory for 30 seconds to avoid repeated filesystem walks.

**GET /health**

Returns `{ "status": "ok", "db": "ok", "github": "ok" }` with individual component health checks. Used by nginx upstream health check and any external monitoring.

#### Archive Storage Strategy

Base path: `/var/www/html/sharex/uploads/kage/` (shared with kage's archive output, intentional — both tools populate the same browsable archive tree on korh.one).

Archive layout for `https://example.com/blog/post-123`:
```
/var/www/html/sharex/uploads/kage/
  example.com/
    blog/
      post-123/
        index.html          -- stripped, static, self-contained HTML
        _assets/
          a3f8c1d2...png    -- image, named by SHA-256 of original URL
          b9e2f4a7...woff2  -- font
          c1d3e5f9...css    -- stylesheet
        _amber-meta.json    -- capture metadata
```

Asset URL rewriting in `index.html`: all `src` and `href` attributes pointing to the original CDN URLs are rewritten to relative paths (`../_assets/a3f8c1d2...png`). This is done with a simple string replace pass over the HTML after asset hashing, before writing to disk. CSS `url()` references within inline `<style>` blocks are also rewritten.

The `index.html` file includes a single injected `<meta>` tag at the top of `<head>`:
```html
<meta name="generator" content="amber/1.0 (https://github.com/spacedudem/amber)">
```
No other injection. No amber scripts, no amber CSS, no amber UI elements in the archived page.

#### SQLite Schema

See `DATA_MODEL.md` for the complete, authoritative SQLite schema including all tables, indexes, field definitions, and migration strategy. The key tables are:

- `archives` — one row per captured page; root entity with `domain`, `url`, `url_hash`, `file_path`, `status`, and denormalized counters.
- `snippets` — one row per stripped element; FK to `archives(id)` via `archive_id INTEGER`, FK to `tactic_types(id)` via `tactic_type TEXT`. Severity is stored as `TEXT` (`'low'`, `'medium'`, `'high'`, `'critical'`).
- `tactic_types` — reference/seed table; primary key is `TEXT` (snake_case tactic name, e.g. `'canvas_fingerprint'`).
- `assets` — one row per fetched asset; FK to `archives(id)`.

Domain hashing: the apex domain (e.g., `example.com` extracted from `subdomain.example.com`) is SHA-256 hashed before storage in the public export. The local `archives` table retains the plain domain string for operational queries. Domain hashes can be verified by anyone who already knows the domain (the hash is not a secret), but the public database cannot be used to enumerate which domains have been visited.

No URLs, no user identifiers, no session tokens, no IP addresses are stored anywhere in the public export. The private SQLite database retains full URLs for operational use by the server operator.

#### GitHub Push Strategy

`github/push.go` uses `os/exec` to invoke `git` commands against a local clone of `spacedudem/amber-tracker-db` maintained at a configured path (e.g., `/opt/amber/tracker-db-repo`). The clone uses a deploy key with write access to the repository.

Push flow:
1. Query SQLite for all snippets where `exported_at IS NULL`.
2. Read existing `tracker_tactics.json` from the local repo clone.
3. Append new records (with domain hashes, not plain domains).
4. Write updated `tracker_tactics.json`.
5. Run export generators (Go functions that produce the uBlock/hosts/disconnect/pihole text from the full snippet set).
6. Write export files to `exports/`.
7. `git add tracker_tactics.json exports/`
8. `git commit -m "add N snippets [automated]"`
9. `git push origin main`
10. On success, update `exported_at` for all pushed snippets in SQLite.
11. Record export event in `exports` table.

This runs in a background goroutine after each archive ingestion. If the push fails (network, auth), the snippets remain with `exported_at IS NULL` and will be included in the next push attempt. There is no deduplication issue because the same snippet content from different captures will have different `submitted_at` timestamps and will appear as separate records.

---

### Public Data Repository (spacedudem/amber-tracker-db)

The public repository serves two audiences: developers who want to consume the raw tracker tactic database, and end users who want ready-to-use filter lists for their ad blockers or DNS blockers.

**tracker_tactics.json**: Array of JSON objects. Each record:
```json
{
  "id": 1423,
  "domain_hash": "a3f8c1d2e4b9f7...",
  "tactic_type": "fingerprinting",
  "raw_snippet": "canvas.toDataURL('image/webp')...",
  "context": { "tag": "script", "parent": "body", "depth": 2 },
  "severity": 3,
  "submitted_at": "2026-06-16T00:00:00Z"
}
```

`raw_snippet` contains the actual code. This is deliberately public — the goal is to document tracking implementations at the code level, not just the domain level. Publishing raw snippets enables privacy researchers, browser vendors, and filter list maintainers to understand exactly what techniques are deployed in the wild.

**GitHub Actions Workflow** (`.github/workflows/export.yml`): Triggers on every push to `main`. Runs a Go binary checked into the repo as a pre-built artifact (or alternatively, a shell script using `jq`). Generates all four export formats from `tracker_tactics.json` and commits the results back to `exports/`. The commit is a machine commit by the `amber-bot` GitHub App identity.

Export formats generated by GitHub Actions:
- `exports/ublock_filters.txt` — uBlock Origin static filter syntax; cosmetic filters (`##`) where the snippet reveals a DOM structure, network filters (`||domain^`) for pixel/beacon URLs extracted from snippet context
- `exports/hosts.txt` — standard UNIX hosts file format (`0.0.0.0 tracker.example.com`)
- `exports/disconnect.json` — Disconnect.me JSON schema for import into browser extensions that consume it
- `exports/pihole.txt` — Pi-hole adlist-compatible domain list, one per line

---

## Data Flow Diagrams

### Capture Flow

```
User              popup.js         content.js        background.js    Go Server       GitHub
  |                   |                 |                  |               |              |
  |--[click]-------->|                 |                  |               |              |
  |                   |--[inject]------>|                  |               |              |
  |                   |                | serialize DOM     |               |              |
  |                   |                | strip pipeline    |               |              |
  |                   |                | collect snippets  |               |              |
  |                   |                |--[sendMessage]--->|               |              |
  |                   |                |  {html,assets,    |               |              |
  |                   |                |   snippets}       |               |              |
  |                   |                |                  | fetch assets  |              |
  |                   |                |                  | (w/cookies)   |              |
  |                   |                |                  | Gemini Nano   |              |
  |                   |                |                  | classify      |              |
  |                   |                |                  |--POST /arch-->|              |
  |                   |                |                  |               | write HTML   |
  |                   |                |                  |               | write assets |
  |                   |                |                  |               | write SQLite |
  |                   |                |                  |               |--[push]----->|
  |                   |                |                  |<-{archiveUrl}-|              |
  |                   |<-[sendMessage]-|------------------|               |              |
  |<-[show link]-----|                 |                  |               |              |
```

### Public Feed Update Flow

```
Go Server              Local git clone        GitHub (amber-tracker-db)   GitHub Actions
    |                        |                          |                       |
    |--[query unexported]--> |                          |                       |
    |--[append records]----->| tracker_tactics.json     |                       |
    |--[gen exports]-------->| exports/*.txt            |                       |
    |--[git add+commit]----->|                          |                       |
    |--[git push]----------->|------------------------->|                       |
    |                        |                          |--[trigger on push]--->|
    |                        |                          |                       | run export.yml
    |--[mark exported_at]--> |                          |<--[commit exports]----|
    |                        |                          |                       |
```

---

## Security Architecture

Full threat model analysis is in `docs/THREAT_MODEL.md`. This section covers the key security controls.

### Extension to Server Authentication

Every request from background.js to the Go server includes the header `X-Amber-Key: <secret>`. The secret is a 256-bit random value stored in `chrome.storage.local` and configured once by the user in the extension's options page. The server validates this header in auth middleware before routing to any handler. Requests without a valid key receive HTTP 401 with no response body.

The key is not embedded in the extension source code or the published extension. Users who install the extension must configure it with their own server URL and key. This makes amber a personal tool by default, not a shared public API.

### HTTPS Enforcement

The extension only allows the configured server URL to use `https://`. If a user attempts to configure `http://`, the options page rejects it with an error. The nginx reverse proxy on natsec terminates TLS with a Let's Encrypt certificate. TLS 1.2 minimum, TLS 1.3 preferred, nginx configured with HSTS.

### No User PII in Public Database

The following data transformations ensure user privacy in the public repository:

- Domain names are SHA-256 hashed before storage and export
- Source URLs are never stored in SQLite or exported to GitHub
- `capturedAt` timestamps in public exports are rounded to the nearest hour (prevents correlation with browsing sessions)
- No user agent strings, IP addresses, or extension installation IDs are transmitted to the server or stored

### Content Security Policy (Extension)

The extension's `manifest.json` specifies a strict CSP:
```json
"content_security_policy": {
  "extension_pages": "script-src 'self'; object-src 'none';"
}
```

No remote scripts, no `eval()`, no inline scripts in extension HTML pages. popup.html and options.html load scripts only via `<script src="popup.js">` referencing bundled files.

### Server Input Validation

- JSON body size limit: 50MB (configurable). Enforced via `http.MaxBytesReader` before decoding.
- `sourceUrl` validated as a parseable URL with `http` or `https` scheme. Domain and path components sanitized before use as filesystem paths.
- Asset content-type validated against an allowlist (image/*, text/css, font/*, text/plain) before writing to disk.
- Snippet `raw_snippet` field capped at 2000 characters, `context` field capped at 500 characters.
- All SQLite queries use parameterized statements via `database/sql` — no string concatenation in SQL.
- Archive output directory confined to the configured base path; path traversal attempts (e.g., `../../etc/passwd` in domain or path) are rejected with HTTP 400.

---

## Deployment Architecture

### Extension Deployment

**Local unpacked install (development / personal use)**: Load `extension/` directory via `chrome://extensions` → Developer mode → Load unpacked. Enable `#optimization-guide-on-device-model` in `chrome://flags`. Configure server URL and API key in extension options. This is the primary use case for the initial version.

**Chrome Web Store (future)**: Requires Origin Trial token from Google for the Prompt API. Token obtained via the Google Origin Trial console. Token embedded in `manifest.json` under `trial_tokens`. Store listing requires privacy disclosure specifying that no browsing data is transmitted to third parties (true — only the user's own server receives data).

### Server Deployment

The Go server runs as a Docker container on the natsec host. Container spec:

```dockerfile
FROM golang:1.22-alpine AS builder
# build server binary with CGO_ENABLED=1 for go-sqlite3

FROM debian:12-slim
# copy binary, configure volumes
```

`docker-compose.yml` mounts:
- `/var/www/html/sharex/uploads/kage/` for archive storage
- `/opt/amber/tracker-db-repo/` for the local GitHub clone
- `/opt/amber/amber.db` for the SQLite database
- SSH deploy key for GitHub push (mounted as read-only secret)

nginx configuration on natsec reverse-proxies `https://korh.one/amber/api/` to the container's loopback port. The archive storage path is also served directly by nginx as static files at `https://korh.one/amber/archives/`.

The server binds to `127.0.0.1:8090` (not `0.0.0.0`) inside the container network. No external port exposure.

### Public Database Deployment

The GitHub repository `spacedudem/amber-tracker-db` is public from day one. GitHub Pages is enabled on the `main` branch, serving `exports/` at `https://spacedudem.github.io/amber-tracker-db/exports/`. This gives end users a stable URL for subscribing to filter lists in their ad blockers:

- uBlock Origin: `https://spacedudem.github.io/amber-tracker-db/exports/ublock_filters.txt`
- Pi-hole: `https://spacedudem.github.io/amber-tracker-db/exports/pihole.txt`
- Hosts-based blockers: `https://spacedudem.github.io/amber-tracker-db/exports/hosts.txt`

---

## Technology Decisions and Rationale

### Go vs Node.js for the Server

Go was chosen. The server's primary operations are disk I/O (writing archive files), SQLite writes, and occasional git subprocess calls — none of which benefit from Node.js's strength in async I/O for high concurrency. Go's `net/http` standard library is production-ready without a framework. The compiled binary is approximately 8MB and starts in under 100ms, convenient for container deployments. The existing kage project (which amber's archive storage path intentionally mirrors) is written in Go, enabling potential future code sharing for the archive browsing layer. Go's `database/sql` interface with `mattn/go-sqlite3` is battle-tested. Node.js was ruled out because it introduces npm dependency management complexity and a larger attack surface for a single-binary server tool.

### SQLite vs PostgreSQL

SQLite was chosen. The server handles at most one capture per user interaction — this is not a high-write-throughput service. SQLite's single-file database is simpler to back up (copy the file), simpler to deploy (no separate process), and simpler to reason about for a single-server personal tool. WAL mode (`PRAGMA journal_mode=WAL`) is enabled to allow concurrent reads during writes without blocking. If amber were deployed as a shared multi-user service with hundreds of concurrent captures, PostgreSQL would be appropriate. For a personal archiving tool, SQLite is the correct choice and avoids operational overhead that would make the project harder to self-host.

### Gemini Nano vs Server-Side LLM

Gemini Nano was chosen. The classification task runs on the user's device, inside Chrome, with no data leaving the browser to any third-party AI service. This is the core privacy argument: a user archiving a sensitive page (financial, medical, legal) should not have their tracking snippets sent to Anthropic, OpenAI, or Google Cloud for classification — only the local model sees the raw content. Gemini Nano at approximately 1.8B parameters is sufficient for the classification task, which is constrained to 12 well-defined tactic categories. The context window limitation (4096 tokens) requires chunking, but this is a solved engineering problem. Fallback to `tactic_type: 'unknown'` when Gemini Nano is unavailable means classification is a best-effort enhancement, not a critical path dependency.

Server-side LLM (e.g., calling an Anthropic API from the Go server) was considered and rejected because: it requires the Go server to receive raw tracking code snippets before classification, which contradicts the local-processing privacy model; it introduces per-classification API cost; and it creates a network dependency in the capture pipeline.

### GitHub vs Custom Database for Public Feed

GitHub was chosen. The public tracker tactic database does not need query capability beyond what `grep` and `jq` provide on the raw JSON. GitHub provides version history (every new snippet batch is a commit, providing a timeline of when tactics were first observed), a pull request workflow for community corrections, free static hosting via GitHub Pages for filter list distribution, and Actions for automated export generation. A custom web service would require hosting, uptime monitoring, and database backup — none of which are necessary when GitHub provides equivalent functionality for this use case.

### MV3 vs MV2

MV3 was chosen and is required. Chrome has deprecated MV2 and will remove it. MV3 is the only viable path for a new extension intended to have any longevity. The specific MV3 constraints that matter for amber (service worker lifetime, no blocking webRequest) do not conflict with amber's design. The service worker model is a better fit than a persistent background page for a one-shot capture tool that should not consume resources while the user browses normally. MV3's stricter CSP aligns with amber's security posture.

---

## Relationship to kage

amber is designed to complement kage (`github.com/spacedudem/kage`), not replace it. kage is a headless scraping tool that handles sites without bot protection. amber handles sites that block headless scrapers (Cloudflare, Akamai, sites requiring real user interaction or login).

Both tools write to the same archive directory tree (`/var/www/html/sharex/uploads/kage/<domain>/`). A future unified archive browser on korh.one will surface captures from both tools in a single interface, with amber captures tagged as `source: extension` and kage captures tagged as `source: headless`.

The amber server does not call kage and kage does not call amber. They are independent tools that share a storage convention.
