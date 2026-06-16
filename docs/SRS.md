# amber — Software Requirements Specification

**Version:** 1.0
**Date:** 2026-06-16
**Status:** Draft
**Project:** amber — Static Browser Archive with Tracker Intelligence

---

## Table of Contents

1. Introduction
2. Overall Description
3. Specific Requirements
4. Appendices

---

## 1. Introduction

### 1.1 Purpose

This Software Requirements Specification (SRS) defines the complete functional and non-functional requirements for amber, a Chrome browser extension and companion Go server that produces clean, static, script-free archives of web pages captured from a live authenticated browser session. The document is authoritative for design, implementation, testing, and acceptance decisions.

This SRS is written for:
- Developers implementing the Chrome extension, Go server, and database schema
- Reviewers evaluating functional completeness and privacy compliance
- Future contributors extending the tracker-intelligence pipeline

### 1.2 Scope

amber consists of three tightly coupled subsystems:

1. **Chrome Extension** — A Manifest V3 extension that serializes the fully rendered DOM, strips all active tracking code, classifies stripped snippets with Gemini Nano, and transmits the resulting package to the Go server.
2. **Go Server** — A lightweight HTTP API server that receives archive packages, persists static HTML and binary assets to disk, and stores classified tracker snippets in a SQLite database.
3. **Public Tracker Feed** — A GitHub repository (spacedudem/amber-tracker-db) populated by the server with anonymized tracker intelligence data; GitHub Actions generate export files in multiple blocklist formats on every push.

amber does not replace existing scraping infrastructure (kage handles non-bot-protected targets). It specifically solves the case where the user is already authenticated in Chrome and the target page is protected by bot-detection systems.

In scope:
- One-click DOM capture and sanitization
- Gemini Nano on-device classification of tracker tactics
- Server-side archive storage with asset embedding
- SQLite-backed tracker intelligence database
- Automated export to uBlock Origin, Pi-hole, Disconnect.me, and hosts-file formats

Out of scope (v1):
- Full session recording (approved for v2)
- OSINT/evidence packages with cryptographic sealing (approved for v2)
- Firefox or Safari extension ports
- Cloud-hosted Gemini API fallback
- User accounts or multi-user deployment

### 1.3 Definitions, Acronyms, Abbreviations

**DOM (Document Object Model):** The in-memory tree representation of an HTML document as constructed and mutated by JavaScript at runtime. amber captures the DOM after JavaScript has finished executing, ensuring that dynamically injected markup is preserved in the archive.

**Gemini Nano:** A 1.8-billion-parameter language model embedded in Chrome 127 and later. It executes entirely on the user's device using the GPU/NPU and requires no network access. amber uses it for classifying stripped code snippets by tracker tactic type.

**Prompt API (Chrome Prompt API):** The JavaScript API exposed by Chrome under `window.ai.languageModel` that provides access to Gemini Nano from extension and web contexts. The API is governed by an Origin Trial in Chrome 127–132 and the `#optimization-guide-on-device-model` flag for local development.

**Origin Trial:** A Chrome mechanism that grants temporary access to experimental browser APIs by embedding a cryptographically signed token in an extension manifest or HTTP header. The token is tied to a specific origin, API name, and expiry date. Public Chrome Web Store distribution of amber requires an Origin Trial token issued by Google; personal use may rely on the `chrome://flags` override.

**Manifest V3 (MV3):** The third generation of the Chrome extension manifest format, mandatory for new Chrome Web Store submissions. MV3 replaces persistent background pages with ephemeral service workers, tightens content security policy enforcement, and restricts remotely hosted code.

**CSP (Content Security Policy):** An HTTP response header and meta tag mechanism that instructs the browser which resource origins are permitted. amber's static archives include a strict CSP meta tag that blocks all script execution and external resource loading.

**CNAME Cloak:** A DNS technique used by first-party tracker deployments to disguise third-party analytics endpoints as subdomains of the target site (e.g., `metrics.example.com CNAME data.thirdpartyanalytics.com`). amber detects CNAME-cloaked beacons by matching request destinations against a known cloak pattern list.

**uBlock Origin:** A widely-deployed content-blocking browser extension that consumes filter lists in Adblock Plus filter syntax. amber's public feed exports a compatible filter list.

**Pi-hole:** A network-level DNS sinkhole that blocks domains listed in its blocklist configuration. amber's public feed exports a compatible domain list.

**Tracker Tactic:** A specific implementation technique used to track users across web sessions or devices. amber classifies tactics at the code level, not merely by domain. Examples: fingerprint collection via Canvas API, session replay via keyboard event listeners, beacon firing via `navigator.sendBeacon`, CNAME-cloaked pixel requests, localStorage-backed cross-site ID sync.

**Static Archive:** An HTML file in which all JavaScript has been removed, all assets (images, CSS, fonts) have been inlined as data URIs or embedded alongside the HTML, all external links are preserved as plain `<a>` hrefs, and a strict CSP meta tag prevents any script execution. A static archive renders identically in any browser, offline, indefinitely, without making any network requests.

**Serialized DOM:** The HTML string produced by `document.documentElement.outerHTML` (or equivalent) after JavaScript has finished mutating the DOM. This captures dynamically injected elements, deferred content, and SPA-rendered markup that would be absent in a raw HTTP response.

**Service Worker:** In the MV3 context, the background.js file that runs as an event-driven ephemeral worker rather than a persistent page. It handles message routing, asset fetching, Gemini Nano calls, and server communication.

**CSP nonce:** A cryptographic random value embedded in a `<script nonce="...">` attribute and the page's CSP header to permit specific inline scripts. amber removes all nonces and their corresponding scripts during sanitization.

### 1.4 References

| ID | Document |
|----|----------|
| R1 | Chrome Extension Manifest V3 documentation — developer.chrome.com/docs/extensions/mv3 |
| R2 | Chrome Prompt API specification — developer.chrome.com/docs/ai/built-in |
| R3 | Origin Trial for Prompt API — developer.chrome.com/origintrials |
| R4 | mattn/go-sqlite3 Go SQLite driver — github.com/mattn/go-sqlite3 |
| R5 | kage archiver — github.com/spacedudem/kage |
| R6 | IEEE Std 830-1998, Recommended Practice for Software Requirements Specifications |
| R7 | Disconnect.me tracker list JSON schema — github.com/disconnectme/disconnect-tracking-protection |
| R8 | uBlock Origin static filter syntax — github.com/gorhill/uBlock/wiki/Static-filter-syntax |
| R9 | Pi-hole compatible blocklist format — docs.pi-hole.net/ftldns/blockingmode |
| R10 | EasyList filter list — easylist.to |

### 1.5 Overview

Section 2 places amber in its broader system context, summarizes product functions, characterizes users, and identifies key constraints. Section 3 provides the complete specification: external interface requirements, detailed numbered functional requirements per component, performance requirements, design constraints, and software quality attributes.

---

## 2. Overall Description

### 2.1 Product Perspective

amber is a new addition to a personal archiving and intelligence ecosystem anchored on the natsec host (Debian Linux, korh.one domain). It complements kage, which handles non-bot-protected targets via headless Chromium; amber handles bot-protected targets by operating inside the user's live authenticated Chrome session.

The following ASCII context diagram shows the major system components and their communication paths:

```
  +-----------------------------------------------------------+
  |                  User's Chrome Browser                    |
  |                                                           |
  |  +----------------+   messages   +--------------------+  |
  |  |  content.js    | -----------> |  background.js     |  |
  |  | (content       | <----------- |  (service worker)  |  |
  |  |  script)       |              |                    |  |
  |  +----------------+              |  window.ai         |  |
  |  +----------------+              |  (Gemini Nano)     |  |
  |  |  popup.html    | -----------> |                    |  |
  |  |  popup.js      |              +----------+---------+  |
  |  +----------------+                         | HTTPS       |
  +-------------------------------------------+-+------------+
                                               |
                                  POST /api/archive
                                  POST /api/snippets
                                               |
  +--------------------------------------------v------------+
  |               Go Server (korh.one/amber)                |
  |                                                         |
  |   +-----------------+        +----------------------+   |
  |   |   HTTP API      |        |   Archive Store      |   |
  |   | /api/archive    | -----> | /var/amber/archives  |   |
  |   | /api/snippets   |        |   (HTML + assets)    |   |
  |   | /api/archives   |        +----------------------+   |
  |   | /health         |        +----------------------+   |
  |   +--------+--------+        |   SQLite DB          |   |
  |            |                 |  tracker_intel.db    |   |
  |            +---------------->|                      |   |
  |                              +-----------+----------+   |
  +------------------------------------------+--------------+
                                             | git push
                                             |
  +------------------------------------------v--------------+
  |        GitHub: spacedudem/amber-tracker-db              |
  |                                                         |
  |   tracker_tactics.json        (raw DB export)           |
  |   exports/ublock_filters.txt  (Adblock Plus syntax)     |
  |   exports/hosts.txt           (POSIX hosts format)      |
  |   exports/disconnect.json     (Disconnect.me schema)    |
  |   exports/pihole.txt          (Pi-hole domain list)     |
  |                                                         |
  |   +------------------------------------------------+    |
  |   |  GitHub Actions: regenerate exports on push    |    |
  |   +------------------------------------------------+    |
  +---------------------------------------------------------+
```

amber does not communicate with any external analytics, cloud AI, or telemetry endpoint. Gemini Nano executes locally; all data flows between the user's browser and the self-hosted Go server.

### 2.2 Product Functions

The primary functions amber provides, in user-visible terms:

1. **One-Click Page Capture** — The user clicks the amber toolbar icon on any page. A single button press initiates capture; no configuration is required for standard operation.
2. **Fully Rendered DOM Serialization** — Captures the page state after all JavaScript has executed, including dynamically injected content and single-page application rendering.
3. **Complete Tracker Sanitization** — Strips every category of active tracking from the serialized DOM before saving: script tags, inline event handlers, tracking pixels, beacon invocations, fingerprinting routines, ad containers, and data-tracking attributes.
4. **On-Device Tracker Classification** — Uses Gemini Nano (local, private, no network) to classify each stripped snippet by tracker tactic type, producing structured intelligence data.
5. **Asset Embedding** — Fetches all referenced images, CSS, and font files through the user's existing browser session (with full cookie access) and embeds them as data URIs so the archive is fully self-contained.
6. **Static Archive Delivery** — Saves a complete, offline-functional HTML file to the Go server's archive store, accessible via the listing API.
7. **Tracker Intelligence Storage** — Persists classified snippets (anonymized, no PII) to SQLite for analysis and export.
8. **Public Blocklist Feed** — Automatically publishes tracker findings to GitHub in four blocklist-compatible formats that the privacy community can consume directly.

### 2.3 User Characteristics

**Primary user: solo technical operator (the repository owner).** This user:
- Has administrator access to the natsec host and the Go server
- Is comfortable operating Chrome developer flags and loading unpacked extensions
- Has reviewed the Origin Trial constraints and accepts personal-use limitations
- Does not require UI onboarding flows or error-message hand-holding

**Secondary users: privacy community consumers of the public tracker feed.** These users:
- Subscribe to the GitHub repository for blocklist updates
- May import the generated filter lists into uBlock Origin, Pi-hole, or other tools
- Never interact with the Chrome extension or Go server directly

No accessibility, localization, or end-user support requirements apply to v1.

### 2.4 Constraints

**C-01 Chrome Version:** amber requires Chrome 127 or later. The Prompt API (`window.ai.languageModel`) is only available from Chrome 127 onward. No fallback to earlier versions is provided.

**C-02 Origin Trial Scope:** During Chrome 127–132, the Prompt API requires either the `#optimization-guide-on-device-model` flag (personal use) or a valid Origin Trial token embedded in `manifest.json` (Chrome Web Store distribution). The token is tied to the declared extension ID and cannot be transferred between builds.

**C-03 Gemini Nano Context Limit:** The Gemini Nano model loaded via the Prompt API has a context window of approximately 4096 tokens. JavaScript chunks submitted for classification must be truncated or split at logical boundaries before submission. No single classification request may exceed 3500 tokens (reserving space for the system prompt).

**C-04 No PII Storage:** amber must not store, log, or transmit any personally identifiable information. This includes but is not limited to: Chrome user profile identifiers, session cookies, authentication tokens visible in captured URLs, full page URLs in the public GitHub feed (domain only is permitted), and user agent strings with hardware identifiers.

**C-05 No Remotely Hosted Code:** Manifest V3 prohibits remotely hosted code in extension context. All extension logic must be bundled at install time. The Gemini Nano model is provided by Chrome; no other remote model endpoint is used.

**C-06 HTTPS Only:** All communication between the extension and the Go server must use TLS. The server must reject plain HTTP connections. The server certificate must be a valid TLS certificate issued by a trusted CA (Let's Encrypt acceptable).

**C-07 Chrome-Only:** v1 targets Chromium-family browsers only (Chrome, Chromium, Edge with Chromium). The Prompt API is not available on Firefox or Safari; no polyfill or fallback is in scope.

**C-08 SQLite Single-File Database:** The tracker intelligence database must be a single SQLite file at a configurable path. No external database server (PostgreSQL, MySQL) is required or supported in v1.

**C-09 Archive Immutability:** Once written to disk, an archive file must not be modified. Updates to a URL's archive must create a new versioned archive entry.

### 2.5 Assumptions and Dependencies

**A-01** The user has a working Chrome installation at 127 or later with Gemini Nano available (verifiable via `chrome://components` showing "Optimization Guide On Device Model" version >= 2024.5).

**A-02** The natsec Go server is reachable from the user's Chrome instance at a known HTTPS endpoint. The server base URL is configurable in the extension's options page and defaults to `https://korh.one/amber`.

**A-03** The GitHub repository `spacedudem/amber-tracker-db` exists, and the Go server has write access via a Personal Access Token stored in the server's environment (`AMBER_GITHUB_TOKEN`).

**A-04** The Go server's archive directory (`AMBER_ARCHIVE_DIR`, default `/var/amber/archives`) has sufficient disk space and write permissions for the server process.

**A-05** nginx on the natsec host proxies `korh.one/amber` to the Go server's local listener (`127.0.0.1:8742`). TLS termination occurs at nginx. The Go server listens only on the loopback interface.

**A-06** kage remains the preferred tool for non-bot-protected targets. amber does not replace kage; the two tools operate independently and may share the same server host.

**A-07** Gemini Nano model weights are already downloaded in Chrome (triggered by the optimization guide component). amber checks model availability before capture and displays an error if the model is not ready.

---

## 3. Specific Requirements

### 3.1 External Interface Requirements

#### 3.1.1 User Interfaces

**Popup (popup.html / popup.js)**

The popup opens when the user clicks the amber toolbar icon. It displays:
- The current tab's title and abbreviated URL (truncated at 60 characters with ellipsis)
- A single "Capture" button (primary action, full-width)
- A status area showing the current phase (Idle, Serializing, Stripping, Classifying, Fetching assets, Uploading, Done, Error)
- A progress indicator (text percentage) during asset fetch
- On completion: a hyperlink to the saved archive on the Go server
- On error: a brief error message and a "Retry" button

The popup must not close during an in-progress capture. If the user closes and reopens the popup during capture, it must display the current progress state, not reset.

**Options Page (options.html / options.js)**

Accessible via right-click on the toolbar icon > "Options". Provides:
- Server base URL field (default `https://korh.one/amber`), validated as a well-formed HTTPS URL before saving
- API key field (if server authentication is enabled), stored in `chrome.storage.local` (not `sync`)
- "Test connection" button that calls `GET /health` and displays the server response
- Gemini Nano availability check button that calls `window.ai.languageModel.availability()` and displays the result
- Classification toggle: enable/disable Gemini Nano classification (default: enabled)

#### 3.1.2 Hardware Interfaces

amber has no direct hardware interface requirements. Gemini Nano uses the GPU or NPU as available through Chrome's internal inference engine; this is transparent to the extension.

#### 3.1.3 Software Interfaces

**Chrome Extension APIs:**
- `chrome.tabs` — query the active tab URL and title
- `chrome.scripting` — inject content.js into the active tab's frame
- `chrome.storage.local` — persist server URL, API key, and in-progress capture state
- `chrome.downloads` — optional: offer local download of the archive HTML as fallback
- Host permissions — `<all_urls>` required to fetch assets from arbitrary origins using the browser session

**Chrome Prompt API:**
- `window.ai.languageModel.availability()` — check model readiness before capture
- `window.ai.languageModel.create({ systemPrompt })` — create a session with the classification system prompt
- `session.prompt(snippet)` — submit a single code snippet and receive a tactic classification
- `session.destroy()` — release model session after all snippets are classified

**Go Server HTTP API:**
- `POST /api/archive` — submit archive package (JSON body)
- `POST /api/snippets` — submit batch of classified snippets (JSON body)
- `GET /api/archives` — list archived pages
- `GET /health` — liveness check

**GitHub API:**
- `PUT /repos/spacedudem/amber-tracker-db/contents/{path}` — upsert files in the tracker DB repository. Used by the Go server's post-insert export routine via the GitHub REST API v3 with `Authorization: Bearer {token}`.

#### 3.1.4 Communication Interfaces

All extension-to-server communication uses HTTPS/TLS 1.2 or later over port 443. Payloads are UTF-8 encoded JSON. Binary assets (images, fonts) within the archive payload are base64-encoded strings within the JSON body.

The maximum single request body size for `POST /api/archive` is 50 MB. Pages with total asset sizes exceeding this limit must have their assets fetched and stored server-side via URL rather than inlined in the request body (fallback mode, see SRS-SV-007).

Content-Type for all POST requests: `application/json`.
Accept header for all GET requests: `application/json`.

Server responses use standard HTTP status codes: 200 OK, 201 Created, 400 Bad Request, 401 Unauthorized, 413 Payload Too Large, 500 Internal Server Error.

---

### 3.2 Functional Requirements

#### 3.2.1 Content Script Requirements

**SRS-CS-001 — DOM Serialization**
The content script shall capture the fully rendered DOM by reading `document.documentElement.outerHTML` after the `DOMContentLoaded` and `load` events have both fired. If the page uses a JavaScript framework with deferred rendering, the content script shall additionally wait 1500 ms after `load` before serializing to allow hydration to complete. The captured string shall be the authoritative source HTML for all subsequent processing.

**SRS-CS-002 — Script Tag Removal**
The content script shall remove all `<script>` elements from the serialized DOM regardless of their `src` attribute, `type` attribute, or content. This includes `type="module"`, `type="text/javascript"`, `type="application/ld+json"` (structured data that can include tracking identifiers), and untyped script tags. Removed script content shall be collected as raw snippets for classification.

**SRS-CS-003 — Inline Event Handler Removal**
The content script shall remove all HTML attribute-based event handlers from every element in the serialized DOM. Targeted attributes include but are not limited to: `onclick`, `onload`, `onerror`, `onmouseover`, `onmouseout`, `onsubmit`, `onchange`, `oninput`, `onfocus`, `onblur`, `onkeydown`, `onkeyup`, `onkeypress`, `onscroll`, `onresize`, `ontouchstart`, `ontouchend`, `ondragstart`. The attribute shall be removed entirely; the element shall be retained. Removed handler values shall be collected as raw snippets.

**SRS-CS-004 — Tracking Pixel and Beacon Removal**
The content script shall identify and remove elements and attributes matching tracking pixel patterns:
- `<img>` elements with zero or one pixel dimensions (`width="0"`, `height="0"`, `width="1"`, `height="1"`) that reference third-party origins
- `<img>` elements whose `src` attribute URL path contains query parameters named `t`, `ts`, `uid`, `cid`, `sid`, `evid`, `event`, `pixel`, or `beacon`
- `<iframe>` elements with zero dimensions referencing third-party origins
- `<link rel="preconnect">` and `<link rel="dns-prefetch">` elements referencing domains matching the tracker domain list

Removed elements shall be collected for classification.

**SRS-CS-005 — Fingerprinting Code Detection**
The content script shall scan all collected script snippets for fingerprinting API usage patterns before classification. Patterns include calls to: `canvas.getContext`, `HTMLCanvasElement.prototype.toDataURL`, `WebGLRenderingContext`, `AudioContext`, `navigator.plugins`, `navigator.mimeTypes`, `screen.colorDepth`, `window.devicePixelRatio`, `navigator.hardwareConcurrency`, `navigator.deviceMemory`, `navigator.languages`. Snippets containing three or more of these patterns shall be pre-tagged with tactic hint `fingerprint` before submission to Gemini Nano.

**SRS-CS-006 — Ad Container Removal**
The content script shall remove elements matching ad container patterns. An element qualifies if its `id` or `class` attribute contains any of: `ad`, `ads`, `adslot`, `advertisement`, `banner`, `dfp`, `gpt`, `prebid`, `adsense`, `doubleclick`. Removal is case-insensitive and matches substrings (e.g., `class="top-ad-container"` qualifies). The element and all descendants shall be removed. Removed outer HTML shall be collected for classification.

**SRS-CS-007 — Data Tracking Attribute Removal**
The content script shall remove the following attribute types from all elements in the serialized DOM:
- All attributes beginning with `data-track`, `data-analytics`, `data-gtm`, `data-ga`, `data-fbq`, `data-pixel`
- `data-src` attributes on `<img>` elements that reference third-party origins (lazy-load tracker images)
- `itemprop` and `itemscope` attributes (Schema.org markup that can expose PII)

The element shall be retained; only the matching attributes shall be removed.

**SRS-CS-008 — Asset URL Collection**
The content script shall collect all asset URLs referenced in the sanitized DOM for subsequent fetching by the background service worker. Collected URL types:
- `<img src>`, `<img srcset>` (all resolution variants)
- `<link rel="stylesheet" href>`
- `@import` URLs found in inline `<style>` blocks (via regex scan of style content)
- `url()` references in inline style attributes
- `<source src>` and `<source srcset>` within `<picture>` and `<video>` elements
- `<link rel="icon" href>` and `<link rel="apple-touch-icon" href>`

All collected URLs shall be resolved to absolute URLs relative to the page's base URL before transmission to the background service worker.

**SRS-CS-009 — CSP Meta Tag Injection**
The content script shall inject a Content Security Policy meta tag into the `<head>` of the sanitized DOM as the first child element. The CSP value shall be:

```
default-src 'none'; style-src 'unsafe-inline'; img-src data: blob:; font-src data:; connect-src 'none'; script-src 'none'; object-src 'none'; frame-src 'none'; base-uri 'none';
```

This policy prohibits all script execution, all external network requests, and all frame embedding in the saved archive.

**SRS-CS-010 — Message Protocol to Background**
Upon completion of sanitization and asset URL collection, the content script shall send a single structured message to the background service worker via `chrome.runtime.sendMessage`. The message shall contain:
- `type: "CAPTURE_PACKAGE"`
- `cleanHtml`: the sanitized DOM string
- `snippets`: array of objects, each with `raw` (string), `hint` (string | null), `elementType` (string), `context` (string, up to 200 characters of surrounding HTML)
- `assetUrls`: array of absolute URL strings
- `pageTitle`: string from `document.title`
- `pageUrl`: string from `window.location.href`
- `capturedAt`: ISO 8601 timestamp string

#### 3.2.2 Background Service Worker Requirements

**SRS-BG-001 — Message Listener Registration**
The background service worker shall register a `chrome.runtime.onMessage` listener on activation. The listener shall handle `CAPTURE_PACKAGE` messages from content.js and `STATUS_QUERY` messages from popup.js. For `STATUS_QUERY`, it shall return the current capture state object without interrupting an in-progress capture.

**SRS-BG-002 — Gemini Nano Availability Check**
Before initiating any Gemini Nano classification, the background service worker shall call `window.ai.languageModel.availability()`. If the result is `"no"`, classification shall be skipped entirely and the snippets shall be submitted to the server with `classifiedBy: "none"`. If the result is `"after-download"`, the background service worker shall wait up to 120 seconds for the model to become available before proceeding; if the model is not ready after 120 seconds, it shall skip classification.

**SRS-BG-003 — Snippet Classification via Gemini Nano**
The background service worker shall create a single Gemini Nano session per capture using the following system prompt:

```
You are a tracker tactic classifier. Given a JavaScript snippet, respond with exactly one JSON object: {"tactic": "<type>", "confidence": <0.0-1.0>, "explanation": "<one sentence>"}. Tactic types: fingerprint, session_replay, beacon, cname_cloak, id_sync, cookie_sync, retargeting_pixel, analytics, ad_auction, unknown.
```

For each snippet in the `snippets` array, the background service worker shall:
1. Truncate the snippet `raw` field to 3500 tokens (approximated as 14000 characters)
2. Submit via `session.prompt(raw)`
3. Parse the JSON response
4. Attach the parsed classification to the snippet object
5. If parsing fails, attach `{"tactic": "unknown", "confidence": 0, "explanation": "Classification parse error"}`

The session shall be destroyed via `session.destroy()` after all snippets in the batch have been classified.

**SRS-BG-004 — Asset Fetching**
The background service worker shall fetch each URL in the `assetUrls` array using `fetch()` within the service worker context (which carries the browser session cookies for the tab's origin). For each URL:
- Set `credentials: "include"` and `mode: "no-cors"` to allow cross-origin asset fetching using session credentials
- If the fetch succeeds (response status 200-299), read the response as `ArrayBuffer` and base64-encode it
- If the fetch fails (network error, 4xx, 5xx), record the URL in a `failedAssets` list and continue
- Respect a per-asset timeout of 10 seconds; abort and record as failed if exceeded

The background service worker shall not fetch URLs whose scheme is `data:` or `blob:` (already inline).

**SRS-BG-005 — Asset Inlining**
After fetching, the background service worker shall replace URL references in the clean HTML with `data:` URIs:
- `<img src="...">` replaced with `<img src="data:{mimeType};base64,{encoded}">`
- `<link href="..." rel="stylesheet">` replaced with `<style>{decoded CSS content}</style>` (CSS files are decoded to text and injected as inline style blocks; `url()` references within CSS that were also fetched are recursively replaced)
- Font `url()` references within CSS replaced with `data:{mimeType};base64,{encoded}`
- `<link rel="icon" href="...">` replaced with `<link rel="icon" href="data:{mimeType};base64,{encoded}">`

Assets that failed to fetch shall retain their original URLs with a `data-amber-fetch-failed="true"` attribute added to the referencing element.

**SRS-BG-006 — Archive Package Submission**
The background service worker shall POST the complete archive package to `{serverBaseUrl}/api/archive` as a JSON body with the following top-level fields:
- `pageTitle`: string
- `pageUrl`: string (full URL, stored server-side only; not published to public feed)
- `capturedAt`: ISO 8601 timestamp
- `cleanHtml`: string (the inlined, sanitized HTML)
- `failedAssets`: array of URL strings
- `snippetCount`: integer
- `clientVersion`: string (extension version from `chrome.runtime.getManifest().version`)

The request shall include an `Authorization: Bearer {apiKey}` header if an API key is configured in the options page. The background service worker shall retry the POST up to 2 times with 3-second delays on network error or 5xx response before reporting failure.

**SRS-BG-007 — Snippet Batch Submission**
After a successful archive submission (SRS-BG-006), the background service worker shall POST classified snippets to `{serverBaseUrl}/api/snippets` as a JSON array. Each element shall contain:
- `archiveId`: string (returned by the server in the archive POST response)
- `domain`: string (eTLD+1 of the page URL, extracted via the Public Suffix List)
- `tactic`: string (from Gemini Nano classification)
- `confidence`: float
- `explanation`: string
- `elementType`: string
- `context`: string (truncated to 200 characters)
- `capturedAt`: ISO 8601 timestamp

The `raw` snippet text shall NOT be included in the batch submission to avoid transmitting potentially PII-containing code fragments to the server's public feed pipeline. The server stores tactic metadata only.

**SRS-BG-008 — Capture State Machine**
The background service worker shall maintain a capture state object in `chrome.storage.local` with the field `captureState` containing one of: `idle`, `serializing`, `stripping`, `classifying`, `fetching_assets`, `uploading`, `done`, `error`. Transitions:
- `idle` to `serializing` on receiving `CAPTURE_PACKAGE`
- `serializing` to `stripping` after content script message received
- `stripping` to `classifying` after sanitization complete
- `classifying` to `fetching_assets` after all snippets classified
- `fetching_assets` to `uploading` after all asset fetches attempted
- `uploading` to `done` on successful server POST
- Any state to `error` on unrecoverable failure
- `done` or `error` to `idle` after 30 seconds or on next capture initiation

**SRS-BG-009 — Popup Status Push**
The background service worker shall send `chrome.runtime.sendMessage` status updates to the popup whenever the capture state transitions. Each update message shall include the current `captureState`, an optional `progress` integer (0-100), and an optional `error` string.

**SRS-BG-010 — Single Concurrent Capture**
The background service worker shall allow only one capture to run at a time. If a new `CAPTURE_PACKAGE` message arrives while `captureState` is not `idle` or `done` or `error`, the background service worker shall respond with an error message `{"error": "Capture already in progress"}` and ignore the new request.

#### 3.2.3 Popup Requirements

**SRS-PU-001 — Current Tab Display**
On open, the popup shall display the active tab's title (truncated to 60 characters) and domain (eTLD+1 only) within 200 ms. It shall not display the full URL to avoid surfacing sensitive path parameters in the UI.

**SRS-PU-002 — Capture Initiation**
The "Capture" button, when clicked, shall:
1. Set the button to a disabled state with label "Capturing..."
2. Send a `chrome.scripting.executeScript` call to inject content.js into the active tab's main frame
3. Transition to displaying the status area

If content.js is already injected (from a prior capture), the popup shall trigger capture via a `chrome.tabs.sendMessage` instead of re-injecting.

**SRS-PU-003 — Live Status Display**
The popup shall listen for status update messages from the background service worker and update the status area text accordingly:
- `serializing` displays "Serializing DOM..."
- `stripping` displays "Stripping trackers..."
- `classifying` displays "Classifying with Gemini Nano..."
- `fetching_assets` displays "Fetching assets ({progress}%)..."
- `uploading` displays "Uploading archive..."
- `done` displays "Saved. [View archive]" as a hyperlink
- `error` displays "Error: {message}" with a "Retry" button

**SRS-PU-004 — Archive Link**
On successful capture (state `done`), the popup shall display a clickable link that opens the saved archive's URL (returned by the server) in a new tab via `chrome.tabs.create`.

**SRS-PU-005 — Model Unavailability Warning**
If the background service worker reports that Gemini Nano was unavailable for the capture (classification skipped), the popup shall display a non-blocking warning: "Tracker classification skipped — Gemini Nano not available." This warning shall not prevent the archive from being saved.

#### 3.2.4 Go Server Requirements

**SRS-SV-001 — Archive POST Endpoint**
The server shall expose `POST /api/archive` that:
1. Validates the request body is valid JSON with required fields: `pageTitle`, `pageUrl`, `capturedAt`, `cleanHtml`
2. Generates a UUID v4 as the archive ID
3. Writes the `cleanHtml` field to disk as `{AMBER_ARCHIVE_DIR}/{archiveId}/index.html`
4. Persists archive metadata to the `archives` SQLite table
5. Returns `201 Created` with body `{"archiveId": "{uuid}", "url": "/archives/{uuid}/index.html"}`

**SRS-SV-002 — Snippet POST Endpoint**
The server shall expose `POST /api/snippets` that:
1. Validates the request body is a JSON array with at least one element
2. Validates each element contains `archiveId`, `domain`, `tactic`, `capturedAt`
3. Verifies the `archiveId` references an existing archive in the `archives` table
4. Inserts each snippet into the `snippets` SQLite table
5. Returns `201 Created` with body `{"inserted": {count}}`
6. After successful insert, triggers the export job (SRS-SV-008) asynchronously

**SRS-SV-003 — Archive List Endpoint**
The server shall expose `GET /api/archives` that returns a JSON array of archive metadata objects sorted by `captured_at` descending. Each object shall include: `archiveId`, `pageTitle`, `domain`, `capturedAt`, `snippetCount`. Full page URLs shall not be included in this response to prevent URL-based PII exposure via the listing endpoint.

**SRS-SV-004 — Health Endpoint**
The server shall expose `GET /health` that returns `200 OK` with body `{"status": "ok", "db": "ok", "archiveDir": "ok"}`. If the SQLite database is unreachable, `"db"` shall be `"error"`. If the archive directory is unwritable, `"archiveDir"` shall be `"error"`. The overall HTTP status shall be `503 Service Unavailable` if either check fails.

**SRS-SV-005 — Static Archive Serving**
The server shall serve the archive directory at `GET /archives/{archiveId}/index.html` as `Content-Type: text/html; charset=utf-8`. It shall set the following response headers on all archive responses:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Content-Security-Policy: default-src 'none'; style-src 'unsafe-inline'; img-src data: blob:; font-src data:;`
- `Cache-Control: public, max-age=31536000, immutable`

**SRS-SV-006 — Request Body Size Limit**
The server shall enforce a 50 MB maximum request body size on `POST /api/archive`. Requests exceeding this limit shall be rejected with `413 Payload Too Large` and body `{"error": "Archive payload exceeds 50 MB limit. Use asset URL mode."}`.

**SRS-SV-007 — Asset URL Fallback Mode**
If the extension detects that the total encoded asset payload will exceed 45 MB before POSTing, it shall transmit only the `cleanHtml` with original URLs retained (not replaced with data URIs) and set `assetMode: "urls"` in the request body. In this mode, the server shall fetch the listed asset URLs server-side (without session cookies, best-effort), inline what it can retrieve, and log failures. This mode produces archives that may have missing assets but avoids 413 errors on heavy pages.

**SRS-SV-008 — Export Job**
After each snippet batch insert, the server shall run an export job that:
1. Queries the `snippets` table for all rows grouped by domain and tactic
2. Generates `tracker_tactics.json` (full database dump, no raw snippets, no page URLs)
3. Generates `exports/ublock_filters.txt` in Adblock Plus static filter syntax from the domain list
4. Generates `exports/hosts.txt` as a POSIX hosts-file format domain blocklist
5. Generates `exports/disconnect.json` in the Disconnect.me JSON schema
6. Generates `exports/pihole.txt` as a newline-separated domain list
7. Commits and pushes updated files to `spacedudem/amber-tracker-db` via GitHub API

The export job shall run in a goroutine and shall not block the HTTP response for `POST /api/snippets`. Export failures shall be logged but shall not affect the server's availability.

**SRS-SV-009 — API Authentication**
The server shall support optional Bearer token authentication. If the environment variable `AMBER_API_KEY` is set and non-empty, the server shall require an `Authorization: Bearer {key}` header on all POST endpoints. GET endpoints (`/api/archives`, `/health`, `/archives/...`) shall remain unauthenticated. Requests with a missing or incorrect API key on protected endpoints shall receive `401 Unauthorized`.

**SRS-SV-010 — Server Configuration via Environment Variables**
The server shall be configured exclusively via environment variables with the following defaults:

| Variable | Default | Description |
|----------|---------|-------------|
| `AMBER_LISTEN` | `127.0.0.1:8742` | Listen address (must be loopback or specific interface) |
| `AMBER_ARCHIVE_DIR` | `/var/amber/archives` | Archive storage root |
| `AMBER_DB_PATH` | `/var/amber/tracker_intel.db` | SQLite database file path |
| `AMBER_API_KEY` | (empty, auth disabled) | Bearer token for POST endpoints |
| `AMBER_GITHUB_TOKEN` | (required for export) | GitHub Personal Access Token |
| `AMBER_GITHUB_REPO` | `spacedudem/amber-tracker-db` | Target GitHub repository |
| `AMBER_MAX_BODY_MB` | `50` | Maximum request body size in MB |

#### 3.2.5 Tracker Database Requirements

**SRS-DB-001 — Schema: archives table**
The `archives` table shall contain the following columns:

```sql
CREATE TABLE archives (
    id          TEXT PRIMARY KEY,
    page_title  TEXT NOT NULL,
    domain      TEXT NOT NULL,
    page_url    TEXT NOT NULL,
    captured_at TEXT NOT NULL,
    snippet_count INTEGER DEFAULT 0,
    client_version TEXT,
    created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
```

The `id` column holds a UUID v4. The `page_url` column stores the full URL server-side only and shall never appear in any exported file.

**SRS-DB-002 — Schema: snippets table**
The `snippets` table shall contain the following columns:

```sql
CREATE TABLE snippets (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    archive_id  TEXT NOT NULL REFERENCES archives(id),
    domain      TEXT NOT NULL,
    tactic      TEXT NOT NULL,
    confidence  REAL NOT NULL DEFAULT 0,
    explanation TEXT,
    element_type TEXT,
    context     TEXT,
    captured_at TEXT NOT NULL,
    created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_snippets_domain ON snippets(domain);
CREATE INDEX idx_snippets_tactic ON snippets(tactic);
CREATE INDEX idx_snippets_captured_at ON snippets(captured_at);
```

The `context` field is limited to 200 characters of surrounding HTML markup. No raw JavaScript snippet text is stored in this table.

**SRS-DB-003 — Schema: tactic_types table**
The `tactic_types` table shall provide a reference set of permitted tactic type identifiers:

```sql
CREATE TABLE tactic_types (
    tactic      TEXT PRIMARY KEY,
    description TEXT NOT NULL,
    severity    INTEGER NOT NULL
);
```

The table shall be seeded with at minimum:

| tactic | description | severity |
|--------|-------------|----------|
| fingerprint | Device fingerprinting via Canvas, WebGL, AudioContext, or navigator APIs | 5 |
| session_replay | Keystroke/mouse event recording for session replay (FullStory, Hotjar patterns) | 5 |
| beacon | Data exfiltration via navigator.sendBeacon or XMLHttpRequest on page unload | 4 |
| cname_cloak | First-party CNAME pointing to third-party tracker endpoint | 4 |
| id_sync | Cross-site user ID synchronization via pixel or redirect chain | 4 |
| cookie_sync | Cookie-based cross-domain identity matching | 4 |
| retargeting_pixel | Retargeting/conversion tracking pixel (Meta, Google, TikTok patterns) | 3 |
| analytics | Standard page analytics (Google Analytics, Segment, Mixpanel patterns) | 2 |
| ad_auction | Programmatic ad auction participation (Prebid, header bidding) | 3 |
| unknown | Classified as tracking but tactic not identified | 1 |

**SRS-DB-004 — Schema: exports table**
The `exports` table shall record every export run:

```sql
CREATE TABLE exports (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    exported_at TEXT NOT NULL DEFAULT (datetime('now')),
    row_count   INTEGER NOT NULL,
    github_sha  TEXT,
    status      TEXT NOT NULL,
    error       TEXT
);
```

The `status` column holds one of: `success`, `partial`, `error`.

**SRS-DB-005 — Data Retention and PII Exclusion**
The database shall not contain: raw JavaScript snippets, full page URLs in any exported file, cookie values, session tokens, user-agent strings with device identifiers, or IP addresses. The `page_url` column in the `archives` table is stored for server-side audit purposes only and shall not appear in any export. Export files shall contain domain names and tactic classifications only.

#### 3.2.6 Public Feed Requirements

**SRS-PF-001 — tracker_tactics.json Format**
The `tracker_tactics.json` file in the public GitHub repository shall be a JSON object with the following structure:

```json
{
  "generated_at": "ISO 8601 timestamp",
  "total_domains": 0,
  "total_snippets": 0,
  "tactics": {
    "{tactic_type}": {
      "count": 0,
      "domains": ["example.com"]
    }
  }
}
```

No raw snippets, no page URLs, and no user identifiers shall appear in this file.

**SRS-PF-002 — uBlock Origin Filter Format**
The `exports/ublock_filters.txt` file shall be a valid Adblock Plus static filter list. It shall include:
- A header comment block with generation timestamp and domain count
- One `||domain^` entry per domain observed in the snippets table
- Tactic-specific cosmetic filters where applicable (e.g., `##[data-track]` to hide tracking-attributed elements)

**SRS-PF-003 — Hosts File Format**
The `exports/hosts.txt` file shall be a valid POSIX hosts-file format blocklist:
- Header lines beginning with `#` giving generation timestamp and count
- One `0.0.0.0 domain` entry per domain
- Sorted alphabetically

**SRS-PF-004 — Disconnect.me JSON Format**
The `exports/disconnect.json` file shall conform to the Disconnect.me tracking protection JSON schema: a `categories` object keyed by category name (Advertising, Analytics, Fingerprinting, Social), each containing a list of organization objects mapping display name to domains object.

**SRS-PF-005 — GitHub Actions Export Regeneration**
The `.github/workflows/export.yml` workflow in the amber-tracker-db repository shall trigger on every push to `main`. The workflow shall:
1. Check out the repository
2. Run a shell script (`scripts/regenerate_exports.sh`) that reads `tracker_tactics.json` and regenerates the four export files
3. Commit and push any changes to the export files back to main
4. Publish the repository via GitHub Pages for direct blocklist URL access

---

### 3.3 Performance Requirements

**PR-01 — Capture Latency:** The total time from popup button click to `done` state (excluding network upload time) shall be under 30 seconds for pages with fewer than 500 script tags and fewer than 200 assets. Classification of up to 50 snippets with Gemini Nano shall complete within 20 seconds.

**PR-02 — Asset Fetch Throughput:** The background service worker shall fetch assets with up to 6 concurrent `fetch()` calls (matching Chrome's per-origin connection limit). Sequential fetching is prohibited.

**PR-03 — Server Archive Write Latency:** The server shall write the archive HTML to disk and return the `201 Created` response within 2 seconds of receiving the complete request body for archives up to 10 MB.

**PR-04 — SQLite Insert Throughput:** The server shall insert a batch of up to 500 snippets in a single SQLite transaction and complete the insert within 500 ms.

**PR-05 — Export Job Duration:** The export job (SRS-SV-008) shall complete — including GitHub API push — within 60 seconds for databases containing up to 100,000 snippet rows.

**PR-06 — Snippet Context Window Compliance:** No single Gemini Nano prompt shall exceed 3500 characters of snippet text plus system prompt overhead. The truncation logic shall operate in O(1) time relative to snippet count.

---

### 3.4 Design Constraints

**DC-01 — No Persistent Background Page:** The extension must comply with Manifest V3 requirements. The background service worker may be terminated by Chrome between captures. State persisted to `chrome.storage.local` must be sufficient to display correct status if the popup is opened after a service worker restart.

**DC-02 — No Remote Code Execution:** The extension must not load any JavaScript from remote URLs at runtime. All classification logic, sanitization routines, and UI code must be bundled at install time. This is both a Manifest V3 requirement and a security requirement.

**DC-03 — Go Standard Library Preference:** The Go server shall use the standard library (`net/http`, `encoding/json`, `database/sql`) wherever feasible. The only permitted third-party dependency is `mattn/go-sqlite3` for CGo-based SQLite access. No web frameworks (Gin, Echo, Fiber) are permitted in v1.

**DC-04 — Loopback Binding:** Per global security defaults, the Go server must bind to `127.0.0.1` or a specific internal interface only. Binding to `0.0.0.0` is prohibited.

**DC-05 — Archive Immutability:** Archive files written to disk must not be overwritten or modified after creation. If the same URL is captured again, a new archive with a new UUID is created. The old archive is retained indefinitely.

**DC-06 — SQLite WAL Mode:** The SQLite database must be opened with WAL (Write-Ahead Logging) journal mode enabled (`PRAGMA journal_mode=WAL`) to support concurrent reads during export job execution without blocking HTTP request handling.

---

### 3.5 Software System Attributes

**Reliability**

The Go server shall handle panics in HTTP handler goroutines with a recovery middleware that logs the error and returns `500 Internal Server Error` without crashing the process. The SQLite database shall be opened with `_foreign_keys=on` and `_journal_mode=WAL`. Archive writes shall be atomic: the HTML file is written to a temporary path and renamed to the final path only on successful write completion.

**Availability**

The server shall be managed by Docker with `restart: unless-stopped` policy on the natsec host. nginx shall be configured with health-check-based upstream failover (single upstream, failover to a static maintenance page). The system has no SLA commitment; best-effort availability for personal use is acceptable.

**Security**

- All communication between extension and server uses TLS (enforced by nginx)
- API key authentication on POST endpoints when `AMBER_API_KEY` is set
- Archive HTML files are served with a strict CSP header (SRS-SV-005) preventing XSS exploitation of archived content
- The SQLite file must be owned by the server process user with mode `0600`
- GitHub Personal Access Token must be scoped to `contents: write` on the target repository only
- The extension requests only the minimum necessary Chrome permissions: `tabs`, `scripting`, `storage`, `activeTab`, and `<all_urls>` host permission (required for asset fetching)
- No user PII is transmitted to GitHub or stored in the public feed

**Maintainability**

- Extension source shall be organized as: `extension/content.js`, `extension/background.js`, `extension/popup.html`, `extension/popup.js`, `extension/popup.css`, `extension/options.html`, `extension/options.js`, `extension/manifest.json`
- Server source shall be organized as: `server/main.go`, `server/handlers.go`, `server/db.go`, `server/export.go`
- SQLite schema migrations shall be managed via sequential numbered SQL files in `tracker-db/migrations/`
- The Gemini Nano system prompt shall be a named constant in `background.js`, not an inline string literal, to simplify future prompt iteration

**Portability**

The Go server binary shall be built with `CGO_ENABLED=1` (required for mattn/go-sqlite3) for the target platform (`linux/amd64`). A `Dockerfile` in `server/` shall provide a reproducible build environment. The Chrome extension is portable across all Chromium-family browsers that support the Prompt API; no browser-specific APIs beyond the Prompt API and standard Chrome extension APIs are used.

---

### 3.6 Other Requirements

**Legal and Ethical**

amber is designed for archiving publicly accessible web pages that the user visits in their own browser session, for personal research and privacy analysis purposes. Users are responsible for ensuring their use of amber complies with applicable terms of service, copyright law, and the Computer Fraud and Abuse Act. The tool shall not be used to capture pages the user is not authorized to access.

**Privacy by Design**

The public tracker feed shall contain only domain names and tactic classifications derived from stripped tracking code. It shall never contain: content from the archived pages, user-identifying information, session data, captured URL paths or query parameters, or timing data that could be correlated to specific users. The server's access logs (managed by nginx) shall be subject to the same log rotation and retention policies as other services on the natsec host.

**Documentation**

The `docs/` directory shall contain, at minimum:
- `SRS.md` (this document)
- `SETUP.md` — step-by-step installation guide covering Chrome flag setup, Origin Trial token, Go server deployment, nginx configuration, and GitHub repository setup
- `ARCHITECTURE.md` — component diagram and data flow description
- `API.md` — full Go server API reference with request/response examples

**Versioning**

The extension version in `manifest.json` and the server version (returned in `GET /health`) shall both follow semantic versioning (`MAJOR.MINOR.PATCH`). v1.0.0 is the target for initial deployment satisfying all requirements in this SRS.

---

## 4. Appendices

### Appendix A: Tactic Classification Prompt Engineering Notes

The Gemini Nano classification prompt (SRS-BG-003) is designed to return structured JSON rather than prose to enable reliable parsing without secondary NLP. The confidence score (0.0-1.0) reflects the model's certainty; scores below 0.5 shall be stored with tactic `unknown` in the public feed regardless of the returned tactic field, to reduce false-positive entries in blocklists.

The system prompt places the output schema first (before the task description) because Gemini Nano at 1.8B parameters responds more reliably to instruction-then-schema ordering than schema-then-instruction. This ordering should be preserved in future prompt iterations.

### Appendix B: Sanitization Completeness Rationale

amber's sanitization approach (strip then classify) differs from allow-listing approaches (permit only known-safe elements). This is intentional: allow-listing is more brittle for long-term archival because new HTML elements and attributes are introduced in web standards periodically. The strip-then-classify model errs toward over-removal (may remove legitimate dynamic content) rather than under-removal (may leave active tracking code). For the use case of creating static archives for evidence and research, over-removal is the correct tradeoff.

### Appendix C: Relationship to kage

kage (github.com/spacedudem/kage) handles non-bot-protected archiving via headless Chromium. amber handles bot-protected pages via the live browser session. The two tools share the same natsec host and may share the same Go server process (different endpoint prefixes) in a future integration milestone. v1 of amber operates independently with no code-level dependency on kage.

### Appendix D: Complete SQLite Schema DDL

```sql
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;

CREATE TABLE IF NOT EXISTS tactic_types (
    tactic      TEXT PRIMARY KEY,
    description TEXT NOT NULL,
    severity    INTEGER NOT NULL
);

INSERT OR IGNORE INTO tactic_types (tactic, description, severity) VALUES
    ('fingerprint',       'Device fingerprinting via Canvas, WebGL, AudioContext, or navigator APIs', 5),
    ('session_replay',    'Keystroke/mouse event recording for session replay (FullStory, Hotjar patterns)', 5),
    ('beacon',            'Data exfiltration via navigator.sendBeacon or XMLHttpRequest on page unload', 4),
    ('cname_cloak',       'First-party CNAME pointing to third-party tracker endpoint', 4),
    ('id_sync',           'Cross-site user ID synchronization via pixel or redirect chain', 4),
    ('cookie_sync',       'Cookie-based cross-domain identity matching', 4),
    ('retargeting_pixel', 'Retargeting/conversion tracking pixel (Meta, Google, TikTok patterns)', 3),
    ('analytics',         'Standard page analytics (Google Analytics, Segment, Mixpanel patterns)', 2),
    ('ad_auction',        'Programmatic ad auction participation (Prebid, header bidding)', 3),
    ('unknown',           'Classified as tracking but tactic not identified', 1);

CREATE TABLE IF NOT EXISTS archives (
    id            TEXT PRIMARY KEY,
    page_title    TEXT NOT NULL,
    domain        TEXT NOT NULL,
    page_url      TEXT NOT NULL,
    captured_at   TEXT NOT NULL,
    snippet_count INTEGER DEFAULT 0,
    client_version TEXT,
    created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS snippets (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    archive_id   TEXT NOT NULL REFERENCES archives(id) ON DELETE CASCADE,
    domain       TEXT NOT NULL,
    tactic       TEXT NOT NULL REFERENCES tactic_types(tactic),
    confidence   REAL NOT NULL DEFAULT 0,
    explanation  TEXT,
    element_type TEXT,
    context      TEXT,
    captured_at  TEXT NOT NULL,
    created_at   TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS exports (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    exported_at TEXT NOT NULL DEFAULT (datetime('now')),
    row_count   INTEGER NOT NULL,
    github_sha  TEXT,
    status      TEXT NOT NULL,
    error       TEXT
);

CREATE INDEX IF NOT EXISTS idx_snippets_domain      ON snippets(domain);
CREATE INDEX IF NOT EXISTS idx_snippets_tactic      ON snippets(tactic);
CREATE INDEX IF NOT EXISTS idx_snippets_captured_at ON snippets(captured_at);
CREATE INDEX IF NOT EXISTS idx_archives_captured_at ON archives(captured_at);
CREATE INDEX IF NOT EXISTS idx_archives_domain      ON archives(domain);
```

### Appendix E: Extension manifest.json Structure

```json
{
  "manifest_version": 3,
  "name": "amber",
  "version": "1.0.0",
  "description": "Save static, script-free page archives and catalog tracker tactics.",
  "permissions": ["tabs", "scripting", "storage", "downloads", "activeTab"],
  "host_permissions": ["<all_urls>"],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },
  "options_page": "options.html",
  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'none';"
  },
  "trial_tokens": ["<ORIGIN_TRIAL_TOKEN_PLACEHOLDER>"]
}
```

---

*End of Software Requirements Specification — amber v1.0*
