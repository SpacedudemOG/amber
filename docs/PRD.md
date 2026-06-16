# amber — Product Requirements Document

**Version:** 1.0  
**Date:** 2026-06-16  
**Status:** Draft  
**Repository:** github.com/spacedudem/amber

---

## Overview

amber is a Chrome browser extension that creates clean, static, script-free archives of web pages from the user's live browser session, then catalogs every tracking tactic it strips so the privacy community can build better countermeasures.

The core insight is that existing archiving tools operate at the wrong layer. Headless scrapers get blocked by bot-detection infrastructure. Tools like SingleFile or HTTrack preserve JavaScript intact, meaning the archived page still phones home when reopened. amber operates from inside a real, authenticated browser session — bot detection is already defeated because the user is already on the page — then performs a thorough, ordered stripping pass on the fully-rendered DOM before anything is saved.

The output is a self-contained static archive: HTML with all assets inlined, no scripts, no network calls, no tracking of any kind. A secondary output is a structured database entry recording exactly which tracking techniques were found, classified by tactic type using Gemini Nano (Chrome's built-in on-device AI), contributed anonymously to a public repository that the privacy research community can use to build better filters and countermeasures.

amber depends on [kage](https://github.com/spacedudem/kage) for archiving sites that do not use bot protection, delegating to amber only when a real browser session is required.

---

## User Requirements

### Functional Requirements

---

#### FR-001: Extension Installation and One-Time Configuration

**Title:** Install and configure amber from Chrome Web Store or unpacked source  
**Priority:** P0  
**Description:** A user must be able to install the amber extension in Chrome 127 or later, either from the Chrome Web Store (once listed) or by loading the unpacked source via `chrome://extensions`. On first run, the extension popup must prompt for the server base URL (e.g., `https://amber.korh.one`) and validate connectivity before allowing capture. The configuration is persisted in `chrome.storage.local` and survives browser restarts without re-prompting.

**Acceptance Criteria:**
- Extension installs without errors on Chrome 127+
- On first popup open, user is shown a server URL field with a "Test Connection" button
- Test Connection fires a GET /health request; success shows a green checkmark, failure shows a descriptive error
- Server URL is stored in chrome.storage.local after successful test
- Subsequent popup opens skip the configuration step and show the capture UI directly
- User can access settings from a gear icon in the popup to change the server URL at any time

---

#### FR-002: One-Click Capture Trigger

**Title:** User initiates capture from the extension popup  
**Priority:** P0  
**Description:** When the user clicks the amber toolbar icon, the popup displays the current page's title and URL, and a single "Capture" button. Clicking Capture begins the full capture pipeline. The button becomes a spinner with status text for the duration of the operation. No additional clicks or configuration should be required for a standard capture.

**Acceptance Criteria:**
- Popup opens within 200ms of clicking the toolbar icon
- Popup shows current tab's title and URL
- "Capture" button is prominently displayed and immediately clickable
- Clicking Capture disables the button and shows a progress indicator
- The popup can be closed and reopened mid-capture without interrupting the operation (capture continues in background.js service worker)
- If no server URL is configured, Capture is disabled with a tooltip directing the user to settings

---

#### FR-003: DOM Serialization After Full Render

**Title:** Serialize the fully-rendered DOM including dynamically inserted content  
**Priority:** P0  
**Description:** content.js, injected into the current page, must capture the DOM as it exists after all JavaScript has finished executing and building markup — not the raw HTML from the network response. This includes content inserted by React, Vue, Angular, or any other client-side framework, as well as dynamically loaded ad and tracking containers. Serialization uses `document.documentElement.outerHTML` on the live DOM.

**Acceptance Criteria:**
- Captured HTML reflects post-JavaScript state (shadow DOM excluded from v1 scope)
- Dynamically inserted elements (e.g., React-rendered content, lazily loaded sections) are present in serialized output
- Serialization captures `<head>` (with meta, link, style) and `<body>` in full
- Serialization completes within 2 seconds for pages under 5MB DOM
- If serialization fails (e.g., cross-origin frame blocks), content.js reports the error to background.js rather than silently producing incomplete output

---

#### FR-004: Script Tag Removal

**Title:** Remove all script elements from the serialized DOM  
**Priority:** P0  
**Description:** The cleaning pipeline must remove every `<script>` element from the serialized HTML, regardless of src attribute, type attribute, or content. This includes `<script type="application/ld+json">` (JSON-LD, which can contain tracking identifiers), `<script type="module">`, `<noscript>` tags (which exist to serve alternate tracking pixels when JS is off), and any `<script>` elements hidden inside `<template>` elements.

**Acceptance Criteria:**
- Zero `<script>` elements remain in cleaned output
- Zero `<noscript>` elements remain in cleaned output
- `<template>` elements are fully removed (their instantiated content may be preserved if already in DOM)
- Removal is verified by parsing final HTML with a DOM parser before transmitting to server
- Count of removed script tags is reported as part of the snippet metadata

---

#### FR-005: Inline Event Handler Removal

**Title:** Strip all inline JavaScript event handlers from HTML attributes  
**Priority:** P0  
**Description:** Every element attribute that begins with `on` and contains JavaScript (onclick, onload, onmouseover, onerror, onsubmit, etc.) must be removed. The attribute name match must be case-insensitive. The raw attribute value must be collected as a snippet for classification before removal.

**Acceptance Criteria:**
- All `on*` attributes removed from every element in the cleaned DOM
- Raw attribute values collected as snippets with `{type: "inline_handler", attribute: "onclick", element_tag: "div", ...}` metadata
- Match is case-insensitive (ONCLICK, OnClick, onclick all stripped)
- Collected snippets truncated to 2048 characters before storage to avoid bloat
- Elements themselves are preserved after handler removal (only the attribute is removed, not the element)

---

#### FR-006: Tracking Pixel and Beacon Removal

**Title:** Remove tracking pixels, web beacons, and server-sent beacon triggers  
**Priority:** P0  
**Description:** The cleaning pipeline must identify and remove known zero-size or near-zero-size images used as tracking pixels (`<img width="1" height="1">`, `<img width="0" height="0">`), `<img>` elements whose src matches known tracker domains, and `<link rel="preload">` or `<link rel="prefetch">` elements pointing to tracker URLs. It must also remove any `navigator.sendBeacon` call patterns found in inline scripts (which will also be removed by FR-004, but snippets must be collected first).

**Acceptance Criteria:**
- `<img>` elements with width and height both <= 1 are removed (as tracking pixels) regardless of domain
- `<img>` elements whose src domain appears in the bundled domain blocklist are removed
- Removed pixel URLs are collected as snippets with tactic_type "tracking_pixel"
- `<link rel="preload">` and `<link rel="prefetch">` pointing to tracker domains are removed
- Legitimate 1x1 spacer images (e.g., old table layouts) may be false-positively removed — this is acceptable in v1

---

#### FR-007: Data Tracking Attribute Removal

**Title:** Remove data-* attributes used for analytics and tracking  
**Priority:** P1  
**Description:** Many analytics libraries embed user identifiers, session identifiers, experiment assignments, and event names in HTML `data-*` attributes. The cleaning pipeline must remove `data-*` attributes matching a configurable blocklist of known tracking attribute names. The blocklist is bundled with the extension and covers common patterns from Google Analytics, Segment, Heap, Mixpanel, and similar tools.

**Acceptance Criteria:**
- Attributes matching blocklist patterns (e.g., `data-ga-*`, `data-segment-*`, `data-heap-*`, `data-analytics-*`, `data-track-*`, `data-gtm-*`) are removed
- Other `data-*` attributes used for UI behavior are preserved
- Removed attribute names and values are collected as snippets with tactic_type "data_attribute"
- Blocklist is a JSON file bundled with extension, updatable without extension code change
- At least 40 known tracking attribute patterns are included in the initial blocklist

---

#### FR-008: Ad Container Removal

**Title:** Remove advertisement containers and iframes  
**Priority:** P1  
**Description:** Elements matching known ad-serving patterns must be removed: `<iframe>` elements whose src matches ad-network domains, elements with class names or IDs matching ad-container patterns (e.g., `div.ad-slot`, `div#google_ads_iframe_*`), and `<ins class="adsbygoogle">` elements. A bundled CSS-selector blocklist governs ad container detection.

**Acceptance Criteria:**
- All `<iframe>` elements are removed in v1 (conservative approach — iframes are rarely needed for static content)
- Elements matching ad-container CSS selector list are removed
- `<ins class="adsbygoogle">` always removed
- Removed elements are counted and reported in capture metadata
- Count of removed ad containers exposed in popup summary after capture

---

#### FR-009: Fingerprinting Code Collection

**Title:** Collect fingerprinting code snippets before script removal  
**Priority:** P1  
**Description:** Before script tags are removed, the cleaning pipeline must scan inline script content for fingerprinting API calls: `navigator.userAgent`, `navigator.platform`, `screen.width`, `screen.height`, `screen.colorDepth`, `canvas.getContext`, `AudioContext`, `WebGLRenderingContext`, `navigator.hardwareConcurrency`, `navigator.deviceMemory`, `navigator.languages`, and battery/USB device enumeration APIs. Matching code fragments are extracted as snippets for Gemini Nano classification.

**Acceptance Criteria:**
- Regex pattern matching covers all listed API surfaces
- Matched fragments extracted with surrounding context (up to 512 chars before and after match)
- Extracted snippets tagged with tactic_type "fingerprinting" before Gemini Nano refinement
- Minified scripts that exceed 4096 chars are split into overlapping 3000-char windows for scanning
- Collection runs before FR-004 script removal so snippets are available

---

#### FR-010: Asset URL Discovery

**Title:** Discover all asset URLs referenced by the page  
**Priority:** P0  
**Description:** After the cleaning pass, content.js must collect all URLs for assets the archive needs to render correctly offline: `<img src>`, `<img srcset>`, `<source src>` and `<source srcset>`, `<link rel="stylesheet" href>`, `@import` URLs inside `<style>` blocks, `url()` references inside inline CSS, and `<link rel="icon">` and `<link rel="apple-touch-icon">`. External font URLs from Google Fonts and similar CDNs are included. URLs are normalized to absolute form before transmission.

**Acceptance Criteria:**
- All image, stylesheet, font, and favicon URLs discovered and deduplicated
- Relative URLs converted to absolute using `new URL(relative, document.baseURI)`
- srcset attributes parsed correctly (multiple URLs per attribute)
- CSS `url()` references inside inline `<style>` tags parsed
- Final list transmitted to background.js for fetching; content.js does not fetch directly

---

#### FR-011: Asset Fetching via Browser Session

**Title:** Fetch all discovered assets using the user's authenticated browser session  
**Priority:** P0  
**Description:** background.js (service worker) must fetch all URLs in the asset list using `fetch()` with `credentials: "include"`, ensuring cookies, auth tokens, and session state are sent. This allows access to paywalled images, CDN-protected resources, and other assets that a headless scraper could not retrieve. Each fetched asset is base64-encoded and included in the archive package sent to the server.

**Acceptance Criteria:**
- Assets fetched with `credentials: "include"` from background.js context
- Failed fetches (404, 403, network error) are logged and skipped — the archive is saved without that asset rather than failing entirely
- Assets larger than 10MB are skipped with a warning in capture metadata
- Total payload to server must not exceed 50MB; assets are dropped in reverse size order if limit approached
- Base64 encoding used for binary assets (images, fonts); UTF-8 text used for CSS

---

#### FR-012: URL Rewriting for Offline Operation

**Title:** Rewrite all asset references in cleaned HTML to local paths  
**Priority:** P0  
**Description:** After all assets are fetched, every reference in the cleaned HTML to an external URL for an asset in the fetched set must be rewritten to a relative local path (e.g., `assets/img_abc123.png`). CSS files must also have their internal `url()` references rewritten if those referenced assets were also fetched. The final HTML file must render identically offline as it did online.

**Acceptance Criteria:**
- All `src`, `href`, and `url()` references pointing to fetched assets rewritten to relative paths
- Path format: `assets/{sha256_first_8_chars}_{filename}` where filename is the last segment of the original URL
- CSS files have their `url()` references rewritten before saving
- Unfetched asset URLs (failed or skipped) are either left as absolute URLs or replaced with a placeholder `data:image/png;base64,...` 1x1 transparent PNG depending on type
- Round-trip test: open archive in browser with network disabled, no console errors about missing resources

---

#### FR-013: Gemini Nano Classification of Stripped Snippets

**Title:** Classify each collected snippet by tracking tactic using Gemini Nano  
**Priority:** P1  
**Description:** background.js must use the Chrome Prompt API (`window.ai.languageModel`, Gemini Nano) to classify each collected snippet into one or more tactic types. Classification runs locally on-device with no data leaving the browser. Each snippet is submitted to the model with a structured prompt asking for a JSON response containing tactic_type, confidence, and a plain-English description of what the code does.

**Acceptance Criteria:**
- Gemini Nano session created via `window.ai.languageModel.create()` before classification
- Each snippet submitted as a separate prompt call (no batching within a single prompt to avoid context overflow)
- Snippets over 3000 tokens are truncated before submission with a note in metadata
- Classification output parsed as JSON; if JSON parsing fails, snippet is stored with tactic_type "unclassified"
- Classification of all snippets must complete within 30 seconds total or be abandoned (remaining snippets stored as unclassified)
- Supported tactic_type values: `canvas_fingerprint`, `webgl_fingerprint`, `font_fingerprint`, `audio_fingerprint`, `battery_api`, `beacon`, `pixel`, `localStorage_abuse`, `sessionStorage_tracking`, `indexeddb_tracking`, `cookie_sync`, `cname_cloak`, `service_worker_tracking`, `inline_handler`, `data_attribute`, `css_tracking`, `third_party_loader`, `behavioral`, `network_timing`, `fetch_exfil`, `unknown`

---

#### FR-014: Gemini Nano Availability Fallback

**Title:** Continue capture without classification when Gemini Nano is unavailable  
**Priority:** P0  
**Description:** Gemini Nano is not available in all Chrome configurations. If `window.ai.languageModel` is undefined or returns an error during session creation, the extension must continue with capture and skip classification. Snippets are stored with tactic_type "unclassified" and a note that classification was skipped. The archive is still saved normally.

**Acceptance Criteria:**
- Absence of `window.ai` or `window.ai.languageModel` is caught without throwing
- `window.ai.languageModel.capabilities()` checked before attempting session creation; if `available` is `"no"`, skip classification
- Popup shows a non-blocking warning: "Gemini Nano unavailable — snippets stored unclassified"
- Full archive is saved regardless of classification availability
- Classification skip is recorded in archive metadata (`gemini_available: false`)

---

#### FR-015: Progress Feedback in Popup

**Title:** Show real-time capture progress in the popup  
**Priority:** P1  
**Description:** The popup must reflect the current stage of the capture pipeline via status messages and a progress indicator. Stages communicated: "Serializing DOM...", "Cleaning markup...", "Discovering assets...", "Fetching assets (N/M)...", "Classifying snippets (N/M)...", "Uploading to server...", "Done." Each message must update in real time via chrome.runtime.onMessage from background.js.

**Acceptance Criteria:**
- Popup shows progress messages matching each pipeline stage
- Asset fetch progress shown as numeric (e.g., "Fetching assets 14/37...")
- Snippet classification progress shown as numeric
- If popup is closed and reopened mid-capture, the current status is recovered from background.js via a status query message
- On completion, popup shows summary: page title, archive URL on server, number of scripts removed, number of assets fetched, number of snippets classified
- On error, popup shows a red error message with the failure stage and error text

---

#### FR-016: Transmit Archive to Server

**Title:** POST cleaned HTML and assets to the Go server  
**Priority:** P0  
**Description:** background.js POSTs the full archive package to `POST /amber/api/v1/archive` on the configured server. The package is a JSON body with fields: `url`, `captured_at`, `amber_version`, `page_title`, `html`, `assets` (array of `{original_url, content_type, data, size_bytes}`), and `snippet_count`. See API_SPEC.md for the authoritative schema. The server endpoint is authenticated with a pre-shared API key.

**Acceptance Criteria:**
- POST body is JSON conforming to the API_SPEC.md `POST /archive` schema
- API key sent as `X-Amber-Key: {key}` header
- API key configured in extension settings alongside server URL
- Total body size enforced under 50MB before POST; assets trimmed if needed
- Server response includes archive ID and public URL; extension shows this in popup on success
- On server error (non-2xx), extension shows error status code and response body (truncated to 500 chars) in popup

---

#### FR-017: Archive Storage on Server

**Title:** Server saves archive to disk with deterministic directory structure  
**Priority:** P0  
**Description:** The Go server receives the archive POST and writes the cleaned HTML and all assets to disk. Directory structure: `{archives_root}/{domain}/{url_path}/`. The index.html is the cleaned HTML. Assets go into an `_assets/` subdirectory (named by SHA-256 prefix of URL + extension). An `_amber-meta.json` sidecar records capture details. The archive is immediately accessible via a stable URL.

**Acceptance Criteria:**
- Archive directory created atomically (write to temp, rename)
- Domain extracted from source URL, sanitized for use as directory name (alphanumeric + hyphen + dot, max 100 chars)
- URL path segments are sanitized: non-alphanumeric characters except `.` replaced with `-`, `..` components rejected
- `_amber-meta.json` contains: `amber_version`, `captured_at`, `original_url`, `page_title`, `asset_count`, `snippet_count`, `tactics_found`, `archive_id`, `url_hash`, `status`
- Archive URL format: `https://{server}/amber/archives/{domain}/{url_path}/`
- Serving the URL delivers the index.html with `Content-Type: text/html; charset=utf-8`

---

#### FR-018: Tracker Snippet Storage in SQLite

**Title:** Store classified snippets in SQLite tracker database  
**Priority:** P0  
**Description:** The server's `POST /api/v1/archive` handler (or a secondary `POST /api/v1/snippets` route) writes each classified snippet to the SQLite tracker database. Before writing, snippets are anonymized: source URL is stripped of path and query parameters (only the domain is retained), and any user-identifiable strings (email patterns, UUIDs matching session patterns) are redacted.

**Acceptance Criteria:**
- Snippets written to `snippets` table with fields: id, archive_id (FK to archives), tactic_type (FK to tactic_types), raw_snippet, context, element_type, attribute_name, severity, classifier_confidence, submitted_to_public, submitted_at (see DATA_MODEL.md for the authoritative schema)
- Source URL anonymized to domain only before storage
- Email address patterns (`/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+/`) redacted in raw_snippet before storage
- UUID patterns that appear likely session identifiers (matched against URL query parameters) redacted
- Domain written to `domains` table with first_seen and last_seen updated on each submission
- Duplicate snippets (same domain + tactic_type + raw_snippet hash) are upserted not inserted

---

#### FR-019: Public GitHub Feed Generation

**Title:** Export tracker database to public GitHub repository formats  
**Priority:** P1  
**Description:** On every push of new snippet data to the tracker DB, a GitHub Actions workflow regenerates export files in the `spacedudem/amber-tracker-db` repository: `tracker_tactics.json` (full database dump), `exports/ublock_filters.txt` (uBlock Origin static filter syntax), `exports/hosts.txt` (hosts file format), `exports/disconnect.json` (Disconnect.me format), and `exports/pihole.txt`. GitHub Pages serves these files publicly.

**Acceptance Criteria:**
- GitHub Actions workflow triggers on push to main of the amber-tracker-db repo
- `tracker_tactics.json` is valid JSON array of all snippet records with anonymized fields
- `ublock_filters.txt` follows uBlock Origin filter syntax (e.g., `||example.com^$third-party`)
- `hosts.txt` follows standard hosts file format (`0.0.0.0 example.com`)
- `disconnect.json` follows Disconnect.me category/resource schema
- `pihole.txt` is a plain domain list compatible with Pi-hole gravity
- All files regenerated within 5 minutes of a push
- GitHub Pages serves files at `https://spacedudem.github.io/amber-tracker-db/`

---

#### FR-020: Archive Browsing API

**Title:** Server exposes API to list and browse saved archives  
**Priority:** P1  
**Description:** The Go server must expose `GET /api/v1/archives` returning a paginated JSON list of saved archives, sortable by date. Each entry includes: archive ID, source URL, captured_at timestamp, archive URL, and summary counts (asset_count, snippet_count). A `GET /api/v1/archives/{id}` endpoint returns full metadata for a single archive including a tactic breakdown. These endpoints are authenticated with the same API key as the POST endpoint.

**Acceptance Criteria:**
- `GET /api/v1/archives` returns JSON object with pagination (`?limit=20&offset=0`)
- Response includes `total`, `limit`, `offset`, and `archives` fields
- `GET /api/v1/archives/{id}` returns full archive metadata as JSON (sourced from the `archives` SQLite table)
- Both endpoints require `X-Amber-Key: {key}` header
- Unauthorized requests return HTTP 401
- Response time under 500ms for the first 20 results

---

### Non-Functional Requirements

---

#### NFR-001: Performance

Capture must complete within 10 seconds for a typical page (under 2MB DOM, under 30 assets, under 20 snippets). Asset fetching and Gemini Nano classification are the longest operations and must run concurrently where possible. Asset fetches must be parallelized (up to 6 concurrent fetches). Gemini Nano classification calls must be serialized (the Prompt API does not support concurrent sessions reliably) but must complete within 30 seconds before being abandoned.

The Go server must handle the archive POST and write to disk within 5 seconds for a 50MB payload. SQLite writes must be wrapped in a single transaction per capture to avoid per-insert overhead.

---

#### NFR-002: Privacy

No PII must be stored or transmitted by the extension to the server, and no PII must appear in the public tracker database. Specifically:

- The extension must never transmit the user's full browsing history, cookies, or authentication tokens to the server
- Only stripped code snippets and their classifications are transmitted
- Snippet content is anonymized before storage (email patterns and session UUID patterns redacted)
- The server must not log request bodies containing asset content
- The extension must not use any analytics SDK or remote logging service
- `chrome.storage.local` usage is limited to server URL and API key; no browsing history is persisted

---

#### NFR-003: Security

- All communication between extension and server must use HTTPS (TLS 1.2 minimum)
- The extension must not use `eval()`, `Function()`, or any dynamic code execution
- The extension manifest must declare a restrictive Content Security Policy
- API key must be stored in `chrome.storage.local` (not in `localStorage`, not in manifest)
- Server must validate the Content-Type of the POST body and reject non-JSON payloads
- Server must enforce a maximum request body size of 50MB and return HTTP 413 if exceeded
- Archive files served by nginx must include `X-Content-Type-Options: nosniff` and `Content-Security-Policy: default-src 'none'; img-src 'self'; style-src 'self'` headers, ensuring archived pages cannot execute scripts even if one slipped through

---

#### NFR-004: Reliability

When Gemini Nano is unavailable (model not downloaded, flag not enabled, API missing), the extension degrades gracefully: capture continues, snippets are stored as unclassified, and the user is notified with a non-blocking warning. When the server is unreachable, the archive package must be retained in `chrome.storage.local` (up to 100MB total) and retried automatically up to 3 times with exponential backoff. After 3 failed attempts, the user is notified and must manually retry from the popup.

---

#### NFR-005: Compatibility

amber targets Chrome 127 and later. It must not use Chrome APIs deprecated before Chrome 127. Manifest V3 is required. The extension must not use `chrome.webRequest` in blocking mode (not available in MV3); all cleanup occurs in content.js on the already-loaded DOM. The `window.ai.languageModel` API requires Chrome 127+ with the Gemini Nano model downloaded; the extension must check for availability rather than assuming it.

---

#### NFR-006: Offline Archive Quality

Saved archives must render completely in a browser with no network connectivity. No external URLs must be referenced in the final HTML for assets that were successfully fetched. CSS must render correctly (fonts inlined or absent, layout preserved). Images must display from inlined data URIs or local relative paths. No JavaScript must execute. The page must pass a manual visual inspection against the original page for layout fidelity.

---

## Feature Specifications

---

### Feature 1: Page Capture

**Trigger:** User clicks the "Capture" button in the extension popup.

**Flow:**

1. popup.js sends a `{action: "capture"}` message to background.js via `chrome.runtime.sendMessage`.
2. background.js queries the active tab and sends a `{action: "serialize"}` message to content.js in the active tab via `chrome.tabs.sendMessage`.
3. content.js executes `document.documentElement.outerHTML` to get the rendered DOM string, then runs the cleaning pipeline (see Feature 2) and asset URL collection (see Feature 3). It returns `{html, assetUrls, snippets}` to background.js.
4. background.js fetches all assets (see Feature 3), runs Gemini Nano classification (see Feature 4), then POSTs the complete package to the server.
5. background.js sends progress updates to popup.js throughout via `chrome.runtime.sendMessage({action: "progress", stage, detail})`.
6. On server success, background.js sends `{action: "done", archiveUrl, summary}` to popup.js.

**Edge Cases:**
- User navigates away mid-capture: content.js context is destroyed. background.js receives a connection error. The partial result is discarded and the user notified.
- Capture on a chrome:// or extension page: content.js cannot be injected. background.js detects the restricted URL and shows an error immediately without attempting injection.
- Page is still loading when capture is triggered: content.js checks `document.readyState`; if not "complete", it waits up to 5 seconds polling before proceeding.
- Multiple concurrent captures: background.js allows only one capture at a time. If a second Capture button press arrives while one is in progress, it is rejected with a status message.

**Error States:**
- content.js injection fails: popup shows "Cannot capture this page type"
- Serialization produces empty string: popup shows "DOM serialization returned empty result"
- All asset fetches fail: archive is still saved with only the HTML; popup shows warning
- Server POST fails: retry logic triggers; popup shows retry count

---

### Feature 2: DOM Cleaning Pipeline

The cleaning pipeline executes in content.js on the serialized HTML string, using a DOMParser to create a manipulable document tree. Operations execute in the following order to ensure snippets are collected before elements are removed.

**Stage 1 — Snippet Collection (before any removal):**
- Scan all `<script>` elements: extract content matching fingerprinting API patterns (FR-009)
- Scan all `on*` attributes across all elements: collect as inline_handler snippets (FR-005)
- Scan `<img>` elements: collect 1x1 pixel URLs as tracking_pixel snippets (FR-006)
- Collect `navigator.sendBeacon` call patterns from inline scripts as beacon snippets

**Stage 2 — Script Removal (FR-004):**
- Remove all `<script>` elements
- Remove all `<noscript>` elements
- Remove all `<template>` elements

**Stage 3 — Attribute Stripping (FR-005, FR-007):**
- Remove all `on*` attributes from all elements
- Remove `data-*` attributes matching blocklist
- Remove `integrity` attributes from link/script elements (no longer relevant, avoids SRI errors on modified content)

**Stage 4 — Tracking Pixel and Beacon Removal (FR-006):**
- Remove `<img>` with width <= 1 and height <= 1
- Remove `<img>`, `<iframe>` whose src domain is in blocklist
- Remove `<link rel="preload|prefetch">` to tracker domains

**Stage 5 — Ad Container Removal (FR-008):**
- Remove all `<iframe>` elements
- Remove elements matching ad CSS selector blocklist
- Remove `<ins class="adsbygoogle">`

**Stage 6 — Metadata Annotation:**
- Add `<meta name="amber:captured-at" content="{ISO8601}">` to `<head>`
- Add `<meta name="amber:source-url" content="{url}">` to `<head>`
- Add `<meta name="amber:version" content="{ext-version}">` to `<head>`
- Add `<meta http-equiv="Content-Security-Policy" content="default-src 'none'; img-src 'self'; style-src 'self'">` to lock down the archive

**What stays:** All text content, layout-bearing div/section/article/nav/header/footer elements (unless matched by ad selector), all `<img>` that are not tracking pixels, all `<style>` and `<link rel="stylesheet">` (URLs rewritten in Feature 3), all semantic HTML, all `<a>` links (preserved as absolute external URLs for reference).

**Edge Cases:**
- Obfuscated script: removed wholesale regardless of content (no attempt to decode)
- `<svg>` with inline `<script>`: SVG script removed (SVG script is still script)
- CSS `animation` referencing external URL: preserved but URL rewritten if asset is fetched
- `<meta http-equiv="refresh">`: removed (auto-redirect in a static archive is undesirable)

---

### Feature 3: Asset Localization

**Discovery (in content.js):**
URL discovery traverses the cleaned DOM tree (post-cleaning, so no ad iframes generate URLs to fetch). Collected URL types: `img[src]`, `img[srcset]` (parsed per W3C srcset spec), `source[src]`, `source[srcset]`, `link[rel=stylesheet][href]`, `link[rel=icon][href]`, `link[rel=apple-touch-icon][href]`, `style` element `url()` references (regex scan), inline `style` attribute `url()` references. All relative URLs resolved against `document.baseURI`.

**Fetching (in background.js):**
Up to 6 concurrent `fetch()` calls with `credentials: "include"`. Each fetch has a 15-second timeout. Responses are read as `ArrayBuffer` for binary types, `text` for CSS. Binary assets are base64-encoded. Failures are logged; the URL is marked as "fetch_failed" in metadata.

**URL Rewriting (in background.js or server-side):**
Before assembling the final HTML, every fetched URL in the cleaned HTML string is replaced with the relative asset path. Replacement is performed as a string operation on the serialized HTML after DOM cleaning, using a map of original URL to local path. CSS files have the same replacement applied to their content.

**Asset Naming:**
`{sha256_of_url[:8]}_{original_filename}` where original_filename is the last path segment of the URL, max 64 chars, sanitized to alphanumeric + hyphen + dot.

**Failure Handling:**
- 404: skip asset, leave original URL or remove attribute
- 403/401: skip asset, log as "auth_required"
- Timeout: skip asset, log as "timeout"
- Asset too large (>10MB): skip, log as "too_large"

---

### Feature 4: Gemini Nano Classifier

**Session Management:**
One `window.ai.languageModel` session is created per capture and reused for all snippets. Session creation uses `systemPrompt` to establish the classifier role. Session is destroyed after all snippets are processed.

**System Prompt:**
```
You are a tracking code classifier. Given a JavaScript code snippet or HTML attribute value extracted from a web page, respond with JSON only. No explanation outside the JSON. Schema:
{"tactic_type": "<one of: canvas_fingerprint|webgl_fingerprint|font_fingerprint|audio_fingerprint|battery_api|beacon|pixel|localStorage_abuse|sessionStorage_tracking|indexeddb_tracking|cookie_sync|cname_cloak|service_worker_tracking|inline_handler|data_attribute|css_tracking|third_party_loader|behavioral|network_timing|fetch_exfil|unknown>", "confidence": <0.0-1.0>, "description": "<one sentence describing what this code does>"}
```

**Per-Snippet Prompt:**
```
Classify this snippet extracted from {element_tag} element (attribute: {attribute_name if applicable}):
---
{snippet_content}
---
```

**Output Parsing:**
Response text is parsed with `JSON.parse()`. If parsing fails, the raw response is stored in a `parse_error` field and tactic_type is set to "unclassified". If confidence is below 0.5, tactic_type is set to "unclassified" regardless of the model's output.

**Chunking:**
Snippets over 3000 characters are split at the nearest statement boundary (`;` or `\n`) before the 3000-char mark. Each chunk is classified independently. If chunks of the same original snippet produce different tactic_types, the most severe tactic_type is used for the combined record.

**Timeout Handling:**
A total 30-second budget is allocated for all classification. If the budget is exhausted, remaining snippets are stored with tactic_type "unclassified" and a note `classification_skipped: "timeout"`.

**Fallback:**
If Gemini Nano is unavailable (FR-014), all snippets go to the server as "unclassified" without any classification attempt.

---

### Feature 5: Archive Storage

**Server-Side Directory Structure:**
```
/var/www/html/sharex/uploads/kage/
  {domain}/
    {YYYY}/
      {MM}/
        {DD}/
          {HH-MM-SS}_{slug}/
            index.html
            _amber-meta.json
            _assets/
              {sha256[:8]}_{filename}
              ...
```

**Write Protocol:**
1. Server receives POST /api/v1/archive JSON body
2. Validates body schema (required fields present, html non-empty)
3. Writes to a temp directory under `/tmp/amber-{uuid}/`
4. Writes index.html, _amber-meta.json, and all asset files
5. `os.Rename()` temp directory to final path (atomic on same filesystem)
6. Returns HTTP 201 with `{id, url}` JSON body

**Deduplication:**
If an archive already exists for the same source URL within the last 5 minutes (checked by scanning `_amber-meta.json` files under the domain directory), the server returns HTTP 409 with the URL of the existing archive rather than creating a duplicate.

**Serving:**
Archives are served by nginx as static files. The server does not serve archive content directly — it delegates to nginx via a configured `root` directive pointing to `/var/lib/amber/archives/`.

**Retention:**
No automatic deletion in v1. Manual cleanup by the operator. Storage usage is reported in the server's `/health` response.

---

### Feature 6: Tracker DB Contribution

**Anonymization (server-side, before SQLite write):**
1. Source URL reduced to domain only (e.g., `https://www.example.com/path?q=1` becomes `example.com`)
2. Raw snippet content scanned for email pattern (`[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`), replaced with `[EMAIL_REDACTED]`
3. UUID-like strings in positions that correspond to URL query parameters (suggesting session IDs) replaced with `[UUID_REDACTED]`
4. Hardcoded API keys (strings matching `[A-Za-z0-9_\-]{20,}` appearing in string literals assigned to variables named `key`, `token`, `secret`, `apiKey`) replaced with `[KEY_REDACTED]`

**Submission Format (SQLite schema):**

See `DATA_MODEL.md` for the authoritative SQLite schema. Key fields in the `snippets` table: `id`, `archive_id` (FK to archives), `tactic_type` (FK to tactic_types), `raw_snippet`, `context`, `element_type`, `attribute_name`, `severity`, `classifier_confidence`, `submitted_to_public`, `submitted_at`. The `archives` table links each snippet batch to the captured page via `archive_id`.

**Rate Limiting:**
The server accepts at most 500 snippets per POST /api/v1/archive request. If more are submitted, only the first 500 (by severity descending, then by snippet length descending as a proxy for complexity) are stored.

**GitHub Export Trigger:**
After each successful write to SQLite, the server calls a GitHub Actions workflow dispatch API (`POST /repos/spacedudem/amber-tracker-db/actions/workflows/export.yml/dispatches`) with a short payload summarizing the update. The workflow regenerates all export files and commits them to the repository.

---

## Out of Scope (v1)

The following features are acknowledged as valuable but explicitly deferred to v2 or later:

- **Full session recording:** Saving an entire browsing session as a timestamped folder with a local map of the user's internet path. Requires a persistent background process and storage design not compatible with MV3 service workers.
- **OSINT evidence package:** Cryptographically signed, timestamped captures with full network request logs suitable for legal proceedings. Requires a notarization service and legal review of evidence chain-of-custody requirements.
- **Shadow DOM capture:** The current serialization uses `outerHTML` which does not include shadow DOM content. A separate Shadow DOM traversal pass would be needed.
- **Firefox/Safari support:** The Chrome Prompt API (Gemini Nano) is Chrome-specific. A cross-browser port would need a different classification mechanism.
- **Extension-side archive browser:** A UI within the extension to browse previously saved archives. In v1, the user accesses archives via the server URL returned after capture.
- **Scheduled/automatic capture:** Capturing pages on a schedule or on URL match rules. In v1, capture is always user-initiated.
- **Collaborative tactic labeling:** Human review and correction of Gemini Nano classifications in the public DB. In v1, Gemini Nano output is final (modulo unclassified fallback).
- **uBlock Origin rule auto-subscription:** Serving the generated uBlock filter list at a URL that users can add directly as a subscription. In v1, files exist at GitHub Pages URLs but no subscription metadata is configured.

---

## Dependencies

| Dependency | Version | Role | Risk |
|---|---|---|---|
| Chrome | 127+ | Extension runtime, Gemini Nano host | Stable, long-term |
| Chrome Prompt API (`window.ai.languageModel`) | Origin Trial / flag | On-device AI classification | High — behind flag or Origin Trial token |
| Gemini Nano model | Auto-downloaded by Chrome | Classification model | Medium — user must have model downloaded |
| Go | 1.22+ | Server runtime | Low |
| mattn/go-sqlite3 | latest | SQLite driver for Go | Low (CGO dependency, requires C compiler) |
| SQLite | 3.x (via go-sqlite3) | Tracker database | Low |
| nginx | 1.24+ | Static file serving, reverse proxy | Low |
| GitHub Actions | N/A | Export workflow automation | Low |
| GitHub Pages | N/A | Public export hosting | Low |
| kage | latest | Non-bot-protected archiving (sibling tool) | Low — independent binary |
| Docker | 24+ | Server containerization on natsec host | Low |

**Critical path dependency:** The Chrome Prompt API Origin Trial token is required for Chrome Web Store distribution. Without it, the extension works only for developers who enable the `#optimization-guide-on-device-model` flag manually. This must be obtained from Google before any public release.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Chrome Prompt API removed or changed before stable release | Medium | High — classification feature broken | Abstract classifier behind an interface; fallback to unclassified already implemented (FR-014); monitor Chrome origin trial status |
| Gemini Nano not downloaded on user's Chrome | High | Low — graceful fallback exists | Popup surfaces clear guidance to enable model download in Chrome settings; classification degrades gracefully |
| Origin Trial token required for Web Store distribution not granted | Medium | High — blocks public distribution | Begin Origin Trial application process early; maintain flag-based developer distribution path |
| 50MB payload limit insufficient for asset-heavy pages | Medium | Medium | Client-side enforcement with asset dropping; future work on streaming upload or server-side asset fetching by URL rather than base64 |
| Manifest V3 service worker 30-second idle timeout interrupts long captures | Medium | High — large pages fail silently | Extend service worker lifetime via `chrome.alarms` keepalive pattern; break capture into checkpointed stages stored in chrome.storage |
| SQLite write contention under concurrent captures | Low (single-user tool) | Low | Single writer enforced by Go mutex; WAL mode enabled |
| Tracking code mutation (sites detect and alter code in response to extension) | Low in v1 | Medium | Capture runs from real user session with no injected identifiers detectable by page JS in MV3 |
| False-positive removal of legitimate content | Medium | Low — archive is still useful | Filtering is conservative (FR-006 notes 1x1 spacers may be false positives); user can optionally request less aggressive cleaning in v2 |
| GitHub Actions export workflow rate-limited | Low | Low | Debounce workflow dispatches to at most once per 5 minutes; batch multi-capture exports |
