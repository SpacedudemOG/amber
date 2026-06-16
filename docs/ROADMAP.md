# amber — Development Roadmap

## Guiding Principles

- **Ship working software at every milestone.** Each milestone ends with a concrete, testable deliverable — not a demo, not a prototype, but a working system that does what it claims.
- **Personal use (single user) before public release.** The tool must prove itself in daily use before we ask anyone else to depend on it. No premature generalization.
- **Privacy by default, not by policy.** Architecture decisions enforce privacy. The system should be incapable of leaking user data, not merely configured not to.
- **Open source from day one.** Both the extension and server are public. The tracker database is public. Trust is earned by transparency.

---

## Milestone 0: Foundation (Week 1–2)

**Goal:** Repository initialized, server scaffold running locally, extension loads in Chrome without errors.

This milestone is infrastructure only. No real features ship, but the skeleton must be solid: wrong architectural decisions made here are expensive to undo. The Go server needs its module structure, SQLite migration runner, and HTTP server wired up. The Chrome extension needs a valid Manifest V3 structure so Chrome accepts it. Both repos need CI so regressions are caught automatically.

### Tasks

- [ ] Initialize `spacedudem/amber` GitHub repo with MIT license, .gitignore for Go and Node, and a minimal README
- [ ] Initialize `spacedudem/amber-tracker-db` GitHub repo with an empty `tracker_tactics.json` and placeholder export files
- [ ] `server/`: `go mod init github.com/spacedudem/amber` with dependencies declared (`mattn/go-sqlite3`, standard library only for HTTP)
- [ ] `server/`: HTTP server on configurable port, `/health` endpoint returning `{"status":"ok","version":"0.1.0"}`
- [ ] `server/`: SQLite migration runner that applies `schema.sql` on startup; schema covers `archives`, `snippets`, `tactic_types`, `domains`, `exports` tables
- [ ] `server/`: Environment-based configuration (port, DB path, archive storage path, GitHub token)
- [ ] `extension/manifest.json`: Manifest V3, permissions `["activeTab","scripting","storage"]`, host permissions `<all_urls>`, empty content/background/popup stubs declared
- [ ] `extension/content.js`: empty module that logs "amber content script loaded" on injection
- [ ] `extension/background.js`: empty service worker that logs "amber background loaded" on install
- [ ] `extension/popup.html` + `extension/popup.js`: minimal popup with amber logo and "Capture" button (disabled, not wired yet)
- [ ] GitHub Actions: `.github/workflows/server.yml` — `go build ./...` and `go test ./...` on every PR to main
- [ ] nginx config stub at korh.one: `/amber/api/` proxied to `127.0.0.1:8090` (not yet deployed, committed for reference)

### Definition of Done

`curl https://korh.one/amber/api/v1/health` returns HTTP 200 with `{"status":"ok"}`. Extension icon appears in Chrome toolbar. `go test ./...` passes in CI.

---

## Milestone 1: Basic Capture (Week 3–4)

**Goal:** Click the extension button → page's rendered DOM is captured → clean HTML (scripts stripped) appears on server disk.

No assets yet, no AI classification yet. Just the core message-passing pipeline between content script, background service worker, and Go server. This milestone proves the architectural plumbing works end to end.

The critical implementation detail here is that `content.js` must capture the DOM *after* JavaScript has executed and built the full markup — `document.documentElement.outerHTML` gives the live rendered tree, not the original HTML source. This is what makes amber fundamentally different from source-download tools like wget.

### Tasks

- [ ] `content.js`: serialize rendered DOM via `document.documentElement.outerHTML` (live DOM, not original source)
- [ ] `content.js`: `NodeIterator`-based pass to remove all `<script>` elements (inline and `src=""` external)
- [ ] `content.js`: second pass to strip inline event handler attributes (`onclick`, `onload`, `onmouseover`, `onerror`, `onsubmit`, and the full 60+ `on*` attribute list from the HTML spec)
- [ ] `content.js`: collect removed snippet text and context (tag name, parent tag, attribute source) into `strippedSnippets[]` array
- [ ] `content.js`: `chrome.runtime.sendMessage()` with `{type:'CAPTURE', html: cleanedHTML, url: location.href, snippets: strippedSnippets}`
- [ ] `background.js`: `chrome.runtime.onMessage` listener routing CAPTURE messages
- [ ] `background.js`: POST JSON body to `${serverUrl}/amber/api/v1/archive` with body matching the API_SPEC `POST /archive` schema
- [ ] `server/`: `POST /amber/api/v1/archive` handler validates request, writes HTML to `archives/{domain}/{slug}/index.html`
- [ ] `server/`: insert row into `archives` table: `id`, `url`, `domain`, `captured_at`, `file_path`, `snippet_count`
- [ ] `server/`: respond with `{archiveId, url: "/kage/{domain}/{slug}/"}` so extension can link the user to the result
- [ ] `popup/`: "Capture" button sends message to active tab's content script and enters loading state
- [ ] `popup/`: on success response, show green checkmark and clickable link to the archive URL
- [ ] `popup/`: on error response, show red error message with retry button

### Definition of Done

Visit paulgraham.com/articles.html, click Capture, visit `korh.one/amber/archives/paulgraham.com/articles/index.html` in a new tab — page renders with text content intact. Open page in airplane mode — it loads with no network requests. `grep -i 'script' index.html` returns no `<script` tags.

---

## Milestone 2: Asset Localization (Week 5–6)

**Goal:** Captured pages include all images, CSS, and fonts fetched through the user's real browser session. Pages work offline completely — not just structurally, but visually.

This is where amber's core advantage over headless scrapers becomes concrete. The service worker fetches assets using `fetch()` with `credentials:'include'`, which means the request goes out with all the user's cookies for that domain. Pages that serve personalized images, paywalled CSS, or CDN assets gated by auth tokens all download correctly.

The transport format uses base64 encoding for assets inside the JSON POST body. For large pages with many images, this will be chunked or the server will need to accept multipart. A realistic ceiling for the initial implementation is 50MB total payload; assets above 10MB individually are skipped with a warning.

### Tasks

- [ ] `content.js`: collect all asset URLs — `<img src>`, `<img srcset>`, `<link rel="stylesheet" href>`, `<link rel="icon" href>`, CSS `url()` references from inline `<style>` blocks, `<source srcset>` in `<picture>` elements, `@font-face` src values in inline stylesheets
- [ ] `background.js`: for each asset URL, `fetch(url, {credentials:'include'})` — runs in service worker with full cookie access
- [ ] `background.js`: convert response `ArrayBuffer` to base64 string, record MIME type from `Content-Type` header
- [ ] `background.js`: skip assets larger than 10MB, record skip in manifest; skip data: URIs (already embedded); skip URLs that 404 or timeout after 5s
- [ ] `background.js`: build `assets[]` array: `[{originalUrl, base64Data, mimeType, size, fetchedAt}]`
- [ ] `background.js`: include `assets[]` in the archive POST body alongside `html`
- [ ] `server/`: decode base64 assets, write to `archives/{domain}/{slug}/_assets/{hash}.{ext}`
- [ ] `server/`: rewrite asset URLs in the captured HTML — `<img src="https://cdn.example.com/photo.jpg">` becomes `<img src="_assets/a3f4b2.jpg">`; handle srcset rewrites; handle CSS url() rewrites in `<style>` blocks
- [ ] `server/`: write `_amber-meta.json` sidecar alongside index.html, containing: original URL, capture timestamp, asset count, snippet count, extension version, and a list of failed assets with their original URLs
- [ ] `server/`: handle asset fetch failures gracefully — if asset missing from payload, keep original external URL as fallback (don't break the page)
- [ ] `popup/`: progress indicator showing "Fetching assets… 12/47" during background fetch phase

### Definition of Done

Visit a news article (logged-in account, paywalled images), click Capture. Disable network adapter. Reload saved archive from disk — all images render, all CSS styles apply, all fonts load. `_amber-meta.json` shows `asset_count` matching the number of assets in `_assets/`. No external network requests visible in DevTools network panel on the saved page.

---

## Milestone 3: Tracking Strip Quality (Week 7–8)

**Goal:** Strip ALL meaningful tracking patterns reliably. Zero tracking code survives in saved pages. Stripped snippets are collected and stored for classification.

Milestones 1–2 stripped scripts. This milestone hardens the strip to cover the full taxonomy of tracking patterns that appear in real pages: tracking pixels disguised as `<img>` tags, iframes used as ad containers and beacon frames, `data-*` attributes that encode user identifiers and session data, and `<noscript>` blocks that implement fallback tracking.

The output of this milestone feeds Milestone 4: every stripped item is stored with enough context (the raw snippet text, the DOM location, the parent element type) for Gemini Nano to classify it.

### Tasks

- [ ] `content.js`: strip tracking pixels — `<img>` elements with `width="1"` or `height="1"` or matching known tracker domains (pixel.facebook.com, bat.bing.com, analytics.google.com, etc.)
- [ ] `content.js`: strip `<iframe>` elements — ad containers, beacon frames, cross-origin iframes with no user-visible purpose; preserve iframes serving user content (embedded video from YouTube, embedded maps) via allowlist
- [ ] `content.js`: strip `data-tracking-*`, `data-analytics-*`, `data-gtm-*`, `data-fb-*`, `data-pixel-*`, and similar tracking attribute families from all elements
- [ ] `content.js`: strip `<link rel="preload">` and `<link rel="prefetch">` pointing to external tracker domains
- [ ] `content.js`: strip `<noscript>` blocks that contain tracking pixels or iframe beacons
- [ ] `content.js`: strip `<meta>` tags used for tracking — Open Graph only if they contain user-identifying data; always strip `<meta name="robots">` override tags
- [ ] `content.js`: strip `<object>` and `<embed>` elements (historical Flash/Silverlight tracking vectors, still appear in archival content)
- [ ] `content.js`: for each stripped element or attribute, record `{type:'element'|'attribute', tagName, attributeName, rawValue, parentTagName, position}` into `strippedSnippets[]`
- [ ] `server/`: `POST /amber/api/v1/snippets` handler (separate from archive POST) accepts body matching the API_SPEC `POST /snippets` schema and inserts into `snippets` table
- [ ] `server/`: `snippets` table fields: `id`, `archive_id`, `domain`, `tactic_type`, `raw_snippet`, `context_json`, `severity`, `submitted_at`, `classified_at`
- [ ] `popup/`: success state shows strip summary: "12 scripts, 3 tracking pixels, 7 ad iframes, 44 tracking attributes removed"
- [ ] Testing: manually verify 20 real-world sites across categories (news, e-commerce, social, SaaS, forums). For each: confirm zero external network requests on saved page, confirm grep for known tracker signatures returns nothing.

### Definition of Done

Open saved page in browser DevTools network panel — zero external requests fire on load or interaction. Open `index.html` in a text editor: `grep -iE 'analytics|gtag|pixel|fingerprint|fbq|_ga|beacon|bat\.js'` returns no matches. SQLite `snippets` table has rows with `raw_snippet` populated and `tactic_type='unclassified'`.

---

## Milestone 4: Gemini Nano Classifier (Week 9–10)

**Goal:** Every stripped snippet classified by tactic type using on-device AI. Classifications stored in SQLite. No snippet leaves the device for classification — Gemini Nano runs locally.

Gemini Nano (~1.8B parameters) is available in Chrome 127+ via the `window.ai.languageModel` API. It runs entirely on-device: no API key, no network request, no cost per call. Classification happens in the background service worker after capture completes and does not block the user's archive from being saved.

The classification taxonomy for the system prompt covers the known tracking tactic space: `canvas_fingerprint`, `webgl_fingerprint`, `audio_fingerprint`, `font_enumeration`, `battery_api_fingerprint`, `behavioral_analytics`, `session_recording`, `conversion_pixel`, `retargeting_pixel`, `a_b_test`, `ad_auction`, `ad_impression`, `ad_click`, `third_party_analytics`, `first_party_analytics`, `user_id_sync`, `cname_cloaking`, and `unknown`. The system prompt instructs the model to return a JSON array with `{tactic_type, confidence, reasoning}` per snippet.

Because Gemini Nano's context window is approximately 4096 tokens, minified JavaScript chunks must be truncated or split. A safe batch size is 3000 tokens of snippet content, leaving room for the system prompt and response.

### Tasks

- [ ] `background.js`: on capture completion, call `window.ai.languageModel.capabilities()` to check availability; if `capabilities.available === 'no'`, skip classification and mark all snippets `tactic_type='unknown'`, `classified_at=null`
- [ ] `background.js`: if available, create model session: `window.ai.languageModel.create({systemPrompt: CLASSIFICATION_SYSTEM_PROMPT})`
- [ ] `background.js`: `CLASSIFICATION_SYSTEM_PROMPT` — instructs model to classify tracking code snippets, return JSON array, defines the tactic taxonomy, specifies confidence score 0.0–1.0, instructs to respond only with valid JSON
- [ ] `background.js`: chunk `strippedSnippets[]` into batches where total token estimate (chars / 4) stays under 3000; include snippet index in each batch so responses can be correlated back
- [ ] `background.js`: for each batch, `session.prompt(batchText)` and parse JSON response; handle JSON parse errors by retrying with a simplified prompt for that batch
- [ ] `background.js`: merge classifications back to snippet array by index; attach to snippet POST payload
- [ ] `background.js`: destroy model session after classification to free memory: `session.destroy()`
- [ ] `server/`: update snippet insert to include `tactic_type`, `confidence`, `reasoning`, `classified_at` fields when provided
- [ ] `server/`: `tactic_types` table: normalized list of known tactic types with human-readable descriptions for the public DB
- [ ] `popup/`: if Gemini Nano available, show tactic summary in success state: "canvas fingerprint (2), behavioral analytics (5), retargeting pixel (3)"
- [ ] `popup/`: if Gemini Nano unavailable, show hint: "Install Chrome Canary or enable #optimization-guide-on-device-model for AI classification"

### Definition of Done

Capture a page known to use canvas fingerprinting (any major ad-supported news site). Query SQLite: `SELECT tactic_type, confidence FROM snippets WHERE archive_id = ?` — at least one row shows `tactic_type='canvas_fingerprint'` with `confidence >= 0.7`. Capture completes without classification if Gemini Nano flag is disabled (graceful fallback confirmed).

---

## Milestone 5: Public Tracker DB (Week 11–12)

**Goal:** Anonymized tracker findings automatically pushed to the public GitHub repo. Export files regenerated on every push. Public can subscribe to filter lists.

The key constraint here is anonymization. The public DB must contain zero information about which user captured a page or when. The domain is hashed before export. The raw snippet is included but any string literals that might identify the capturing user (email addresses, usernames, session tokens that appeared in the JS) are redacted. The server-side anonymization pipeline runs before the GitHub push, not as a post-processing step on the public repo — the private DB may retain more detail, but the public export is clean by construction.

The GitHub push uses the GitHub Contents API (PUT `/repos/spacedudem/amber-tracker-db/contents/tracker_tactics.json`) with a personal access token scoped to that repo only. GitHub Actions on the `amber-tracker-db` repo triggers on push to main and regenerates the four export formats.

### Tasks

- [ ] `server/`: anonymization pipeline function: takes a `snippets` batch, hashes domain with SHA-256 (domain is not reversed from hash), strips string literals matching email/UUID/JWT patterns from `raw_snippet`, removes `archive_id` foreign key from export record
- [ ] `server/`: build `tracker_tactics.json` structure: `{schema_version, generated_at, entries: [{id, domain_hash, tactic_type, confidence, raw_snippet_redacted, context_summary, severity, first_seen}]}`
- [ ] `server/`: GitHub API push — on each new batch of classified snippets, rebuild the full `tracker_tactics.json` and PUT to `spacedudem/amber-tracker-db` via GitHub Contents API; include SHA of current file in PUT request to handle concurrent updates
- [ ] `amber-tracker-db`: initial `tracker_tactics.json` with empty `entries[]` array and correct schema
- [ ] `amber-tracker-db/.github/workflows/exports.yml`: trigger on push to main; run export script generating four formats
- [ ] Export script: `ublock_filters.txt` — one uBlock Origin static filter per unique `(domain_hash, tactic_type)` pair; format `! amber-tracker-db: {tactic_type}\n||{domain}^$third-party` (using a reverse-lookup table of known domains)
- [ ] Export script: `hosts.txt` — standard hosts file format, `0.0.0.0 {domain}` per tracker domain, with comment header
- [ ] Export script: `disconnect.json` — Disconnect.me JSON schema with categories mapped from amber tactic types
- [ ] Export script: `pihole.txt` — Pi-hole compatible domain blocklist
- [ ] GitHub Pages: serve `amber-tracker-db.github.io/` with index linking to all four export URLs
- [ ] `server/`: API endpoint `GET /amber/api/v1/stats` — returns `{total_snippets, classified_snippets, tactic_type_counts, domains_seen}` for public dashboard use

### Definition of Done

Capture 3 pages with known trackers. Check `github.com/spacedudem/amber-tracker-db` — `tracker_tactics.json` has new entries. GitHub Actions run completed successfully. Download `exports/ublock_filters.txt` from GitHub Pages URL, import into uBlock Origin — new rules appear in My Filters. Confirm zero user-identifying information in any public export by inspection.

---

## Milestone 6: Polish and Personal Release (Week 13–14)

**Goal:** Polished daily driver for personal use. First-time setup flow works without touching config files. Server runs in Docker. Self-hosting is documented.

At this point the system works end to end for a single user who is also the developer. This milestone closes the gap between "it works if you know what you're doing" and "it works on a fresh Chrome install after following the README." The options page replaces hardcoded server URL constants. The Docker Compose file replaces manual Go build steps. The nginx config is committed and documented.

### Tasks

- [ ] `extension/options.html` + `extension/options.js`: settings page with server URL field, API key field (masked), "Test Connection" button that hits `/health`
- [ ] `extension/options.js`: Gemini Nano status display — "Available (Chrome 127+)", "Not Available — enable flag", "Checking…"
- [ ] `extension/options.js`: persist settings to `chrome.storage.sync` so they survive extension updates
- [ ] `extension/`: first-time setup wizard triggered when no server URL configured — 3 steps: (1) paste server URL, (2) paste API key, (3) test connection + confirm Gemini Nano status
- [ ] `background.js`: retry queue — failed archive POSTs stored in `chrome.storage.local` and retried with exponential backoff (1m, 5m, 30m, 2h, 24h) up to 5 attempts
- [ ] `server/`: simple API key middleware — single key from environment variable, checked on all write endpoints; GET endpoints and `/health` are unauthenticated
- [ ] `server/Dockerfile`: multi-stage build, final image `FROM scratch` with static binary + CA certificates + SQLite libs
- [ ] `server/docker-compose.yml`: amber service + volume mounts for archives and SQLite DB; port 8090 bound to 127.0.0.1 only
- [ ] `nginx/amber.conf`: location block for `korh.one/amber/api/` proxied to amber server; location block for `korh.one/amber/archives/` serving static files from archive directory with `Content-Security-Policy: default-src 'none'; img-src 'self'; style-src 'self'; font-src 'self'` (enforces that saved pages truly have no external dependencies)
- [ ] `docs/SELF_HOSTING.md`: step-by-step guide — Docker Compose, nginx config, extension options, GitHub token setup for tracker DB push
- [ ] `docs/`: ensure ARCHITECTURE.md, DATA_MODEL.md, PVD.md, USER_STORIES.md are all current and consistent with implementation

### Definition of Done

Fresh Chrome profile → install unpacked extension → wizard prompts for server URL and API key → enter values → test connection shows green → capture a page → archive appears at `korh.one/amber/archives/` → no manual config file edits required. Docker Compose `up` starts server cleanly from a fresh clone. nginx CSP header confirmed on saved archive pages via `curl -I`.

---

## Milestone 7: Public Beta (Month 4)

**Goal:** Other people can install and use amber. Chrome Web Store listing. Multi-user support. Rate limiting.

The jump from personal to public involves two blockers: Chrome Web Store distribution and Origin Trial. The Chrome Prompt API is currently behind an Origin Trial — the Origin Trial token must be embedded in the extension manifest for the `window.ai` API to be available in production Chrome (as opposed to Chrome Canary with flags). Apply for the Prompt API Origin Trial at [developer.chrome.com/origintrials](https://developer.chrome.com/origintrials/) and embed the returned token in `manifest.json` under `"trial_tokens"`.

Multi-user support is limited: the server supports multiple users with per-user archive directories, but the system is not designed as a hosted SaaS — users self-host. The extension is configured to point at the user's own server.

### Tasks

- [ ] Apply for Chrome Extension Origin Trial for Prompt API; embed trial token in `manifest.json`
- [ ] Chrome Web Store listing: description, screenshots, privacy policy (hosted at `korh.one/amber/privacy`), category "Productivity"
- [ ] `privacy policy`: explicit statement that no browsing data is sent to amber servers; all AI classification is on-device; only stripped code snippets (not URLs, not page content) are in the public DB; user controls what gets submitted to public DB
- [ ] `server/`: per-user archive directories keyed by API key hash — `archives/{user_hash}/{domain}/{slug}/`
- [ ] `server/`: rate limiting on `/amber/api/v1/snippets` — max 500 snippets per hour per API key using in-memory token bucket
- [ ] `server/`: rate limiting on `/amber/api/v1/archive` — max 100 archives per day per API key
- [ ] `docs/CONTRIBUTING.md`: guide for contributing tactic classifications, correcting misclassifications, and adding new tactic types to the taxonomy
- [ ] `tracker-db/`: web interface on GitHub Pages showing tactic type breakdown, most common patterns, recent additions — rendered from `tracker_tactics.json` with vanilla JS, no build step

### Definition of Done

Chrome Web Store submission accepted (or under review). A user who has never seen the codebase can install the extension from the Web Store, follow the self-hosting README, and capture their first page within 30 minutes. Rate limiting confirmed by sending 501 snippets and observing 429 on the 501st.

---

## Future Milestones (v2+)

These are pre-approved and will be scheduled after the public beta stabilizes.

**Session Recording.** Save an entire browsing session as a timestamped folder — every page visited in sequence, full offline archive, local-only map of the session path. Useful for research workflows where you want to reconstruct exactly what you read and in what order. Implementation extends the existing capture pipeline with a session start/stop control and automatic capture on tab navigation events.

**OSINT Evidence Package.** Cryptographically sealed, timestamped page captures suitable for legal proceedings. Each capture gets a SHA-256 hash of the archive contents, a timestamp from a trusted time source (RFC 3161 timestamp authority), and a full network request log showing exactly what resources the page loaded before stripping. The sealed package is a tar.gz with a detached GPG signature. Implementation adds a "Legal Mode" toggle to the popup that activates network logging via the `chrome.webRequest` API and invokes the timestamp authority API after capture.

**Firefox Support.** Dependent on Firefox shipping a Web Extensions AI API equivalent to Chrome's Prompt API. Monitor [mozilla/standards-positions](https://github.com/mozilla/standards-positions) — the classification step degrades to `tactic_type='unknown'` on Firefox until then, but all other functionality works with the standard WebExtensions API.

**Mobile (Chrome Android).** Chrome for Android will eventually ship Gemini Nano via the Prompt API. When it does, the extension architecture is unchanged — the same manifest and scripts work. The blocker is that Chrome for Android does not support extensions at the manifest level. Monitor the Chrome Extensions roadmap for Android support.

---

## Dependency Timeline

| Dependency | Status | Risk | Mitigation |
|---|---|---|---|
| Chrome Prompt API (window.ai) | Available in Chrome 127+ with flag; Origin Trial active | Medium — API surface may change | Graceful fallback to `tactic_type='unknown'`; classification is enhancement not core |
| Chrome Prompt API stable (no flag) | Targeted ~Chrome 132 | Low | Origin Trial token bridges the gap for Web Store distribution |
| Gemini Nano model updates | Automatic via Chrome update | Low — API is stable, model improves | No action required; classifications may improve automatically |
| GitHub Actions (free tier) | 2000 minutes/month on free | Low for single-user tracker DB | Export script runs in under 30 seconds; well within limits |
| mattn/go-sqlite3 CGO dependency | Stable, widely used | Low | Pin version in go.mod; Docker build handles CGO environment |
| Chrome Web Store Origin Trial | Requires Google approval | Medium — timeline unknown | Personal use works without it via unpacked extension; public launch gated on approval |

---

## What Is Not In Scope

**No user accounts on the amber server.** The server uses API keys, not accounts. There is no registration flow, no email verification, no password reset. Each user runs their own server.

**No cloud classification.** Snippets are classified by Gemini Nano on-device or not at all. There is no fallback to a hosted LLM API. This is a hard privacy boundary.

**No automatic crawling.** amber captures what the user manually browses. It does not spider links, does not capture pages in the background, does not have a scheduler. It is a capture tool, not a crawler. Use kage for crawling.

**No real-time collaboration.** The public tracker DB is a one-way push. There is no comment system, no dispute mechanism, no voting on classifications. Contribution is via GitHub Pull Request to `amber-tracker-db`.

**No mobile app.** amber is a Chrome extension. There is no React Native app, no iOS Safari extension, no standalone Android app.
