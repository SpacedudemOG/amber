# amber — User Stories & Use Cases

---

## Personas

### Alex — Privacy-Conscious Researcher

**Background:** Alex is a graduate student studying digital surveillance and platform accountability. They spend hours daily reading academic papers, policy documents, corporate filings, and investigative journalism hosted on sites that aggressively fingerprint browsers and resist archiving. Alex runs uBlock Origin and Privacy Badger but knows these tools only block network requests — the JavaScript that attempts fingerprinting still runs inside the page and leaves a behavioral trace before it is blocked.

**Goals:**
- Save clean, fully-rendered copies of web pages for offline reading and citation
- Understand concretely which tracking mechanisms were active on a given page, not just which domains were blocked
- Contribute findings to a public resource that helps others build better defenses
- Keep archived material indefinitely without it degrading or phoning home

**Frustrations:**
- SingleFile and other archive tools save JS intact; reopening an archive fires analytics events
- The Wayback Machine does not always archive paywalled or Cloudflare-gated pages
- Browser DevTools show network requests but do not classify inline tracking code or obfuscated scripts
- No tool gives a human-readable explanation of what a specific obfuscated snippet was trying to do

**Technical level:** High. Comfortable with browser DevTools, command-line tools, JSON, and reading source code. Runs a personal Linux server. Not a developer by trade but can configure technical tools.

---

### Jordan — Investigative Journalist

**Background:** Jordan works for a mid-size digital news outlet covering corporate malfeasance and government accountability. Their investigations involve archiving evidence from websites that frequently change or disappear — press releases, product pages, policy documents, investor disclosures. Jordan has been burned before by a company quietly editing a page after publication, with no archived copy to prove the original content.

**Goals:**
- Create timestamped, tamper-evident copies of web pages as evidence for published stories
- Archive pages behind Cloudflare or login walls that headless scrapers cannot reach
- Access saved archives weeks or months later without link rot or JavaScript errors
- Eventually produce legally defensible captures with cryptographic seals (v2 OSINT package)

**Frustrations:**
- Archive.org sometimes refuses to archive certain domains or is blocked by robots.txt
- Pages captured with conventional tools are still live documents — JavaScript loads, the page calls home, tracker code runs
- There is no easy way to prove a page has not been altered between capture and publication
- Setting up technical archiving tools requires sysadmin skills Jordan does not have

**Technical level:** Medium. Comfortable with web browsers, note-taking apps, and basic file management. Uses cloud tools extensively. Not comfortable with command line or server configuration — needs a polished one-click experience.

---

### Sam — Security Researcher / Filter List Maintainer

**Background:** Sam maintains a mid-size community filter list for uBlock Origin and contributes to the Disconnect.me tracker database. Their workflow involves manually reviewing page source, identifying new tracker domains and fingerprinting scripts, and writing regex patterns or domain rules. The process is tedious: obfuscated JavaScript requires manual deobfuscation, and the community has no systematic database of implementation-level tracking techniques.

**Goals:**
- Automatically extract and classify tracking snippets from any page in the browser session
- Feed classified findings into a community database that auto-exports to uBlock, hosts.txt, and Pi-hole formats
- Identify novel fingerprinting techniques (canvas, AudioContext, font enumeration) that domain-only lists miss
- Review the public tactic database to find patterns and write better filter rules

**Frustrations:**
- EasyList and Disconnect only track domains — inline scripts that do not make external requests are invisible to them
- Obfuscated JS bundled with legitimate code cannot be blocked at the network level without breaking pages
- No shared database exists for tracking implementation patterns — every researcher rediscovers the same techniques independently
- Gemini/GPT-based classification tools require sending code to external servers — a privacy violation when analyzing sensitive pages

**Technical level:** Expert. Writes regex, reads minified JS, understands browser APIs for fingerprinting, runs local tooling. Wants raw data exports and programmatic access.

---

### Morgan — Legal Professional

**Background:** Morgan is a litigation attorney at a firm that handles privacy law cases and GDPR enforcement actions. They frequently need to prove that a website tracked users without consent, collect evidence of deceptive design patterns, or show what a page said at a specific point in time. Web captures are regularly challenged in court because standard screenshots and even PDF prints can be altered and do not capture the underlying code behavior.

**Goals:**
- Produce captures that are admissible as evidence: timestamped, showing the rendered page as the user saw it, and including machine-readable proof of what tracking code was present
- Archive pages behind login or Cloudflare protection that automated tools cannot reach
- Maintain a chain of custody for archived files — know exactly when and how each capture was made
- Eventually use cryptographically sealed packages (v2 OSINT feature) for high-stakes proceedings

**Frustrations:**
- Screenshot tools produce only images — no proof of what JavaScript was running
- Existing archive tools either cannot reach protected pages or produce archives that still execute tracking code, which weakens their evidential value
- No tool provides a machine-readable manifest of exactly what was stripped and why

**Technical level:** Low-to-medium. Highly proficient with office software and legal research tools. Comfortable installing browser extensions. Not comfortable with server setup — expects a hosted or easily installable server component.

---

## Epic 1: Page Capture & Archive

**Theme:** Save a clean, static copy of what I am viewing.

---

**US-101**
As Alex, I want to capture the current page with a single click so that I do not interrupt my reading flow or lose my place in a long document.

*Acceptance criteria:*
- Given I am on any HTTP/HTTPS page with the amber extension installed,
- When I click the amber toolbar icon,
- Then the extension begins capture immediately without requiring additional confirmation dialogs,
- And the popup shows a progress indicator (Serializing > Stripping > Fetching assets > Uploading),
- And I am returned to the page within 15 seconds for typical article-length pages.

---

**US-102**
As Jordan, I want the capture to work on Cloudflare-protected pages so that I can archive evidence that automated scrapers cannot reach.

*Acceptance criteria:*
- Given I have passed a Cloudflare challenge and am viewing the target page normally,
- When I trigger a capture,
- Then the extension serializes the fully-rendered DOM from within my live browser session,
- And asset fetches use my existing session cookies so CDN-gated images and stylesheets are retrieved successfully,
- And the resulting archive renders identically to the live page in an offline browser.

---

**US-103**
As Alex, I want the saved archive to contain zero JavaScript so that reopening it never fires analytics events or calls home.

*Acceptance criteria:*
- Given a captured archive is opened in any browser, including one with no network connection,
- Then no script tags are present in the HTML,
- And no inline event handlers (onclick, onload, onmouseover, etc.) are present on any element,
- And the browser Network panel shows zero outbound requests when the archive is opened,
- And the page renders its text, images, and CSS layout correctly.

---

**US-104**
As Jordan, I want every capture to include a machine-readable timestamp so that I can prove when the page was archived.

*Acceptance criteria:*
- Given a capture completes successfully,
- Then the server stores a `captured_at` field in ISO 8601 UTC format,
- And the archive HTML includes a `<meta name="amber:captured-at">` tag with that timestamp,
- And the archive listing API returns the timestamp for every archive,
- And the timestamp reflects the moment the DOM was serialized, not the moment assets finished uploading.

---

**US-105**
As Alex, I want images, stylesheets, and fonts to be embedded in the archive so that the page looks correct when opened offline without an internet connection.

*Acceptance criteria:*
- Given a page with external images, Google Fonts, and external CSS,
- When a capture is made,
- Then each image is fetched using the browser session and embedded as a base64 data URI in the HTML,
- And each external stylesheet is fetched and inlined as a `<style>` block,
- And each web font file is fetched and embedded as a base64 data URI in the `@font-face` declaration,
- And the archive renders without any broken image icons or missing font fallbacks.

---

**US-106**
As Sam, I want the extension to capture dynamically-inserted DOM elements — content added by JavaScript after initial load — so that the archive reflects the page as the user actually saw it, not as it was delivered from the server.

*Acceptance criteria:*
- Given a single-page application or infinite-scroll page where content is injected by React, Vue, or vanilla JS,
- When I trigger capture after the page has rendered,
- Then the serialized HTML includes all DOM nodes that were present at the time of capture, including those injected by JavaScript,
- And the capture does not re-fetch or re-execute the original server HTML.

---

**US-107**
As Jordan, I want to see a clear success or failure notification in the popup so that I know whether the capture completed before navigating away.

*Acceptance criteria:*
- Given a capture is in progress,
- When it completes successfully,
- Then the popup displays "Archive saved" with a direct link to the saved archive,
- And if any asset failed to fetch, a count of skipped assets is shown (e.g., "3 assets skipped"),
- And if the server is unreachable, the popup displays "Upload failed — check server connection" and no partial archive is created.

---

**US-108**
As Morgan, I want the archive to include a manifest listing every element that was stripped so that I can provide a machine-readable record of what tracking code was present on the page.

*Acceptance criteria:*
- Given a capture completes,
- Then a `amber-manifest.json` file is included alongside the archive HTML,
- And the manifest lists every removed element with: element type, a truncated preview of the content (first 120 chars), the tactic classification, and whether it was a script tag, inline handler, tracking pixel, or ad container,
- And the manifest includes the total counts by category (e.g., `{"script_tags": 14, "inline_handlers": 32, "tracking_pixels": 3}`).

---

## Epic 2: Tracking Code Classification

**Theme:** Understand and expose what was tracking me.

---

**US-201**
As Alex, I want each stripped tracking snippet to be classified by tactic type so that I can understand not just that tracking occurred but specifically how.

*Acceptance criteria:*
- Given a snippet has been extracted from the page during capture,
- When Gemini Nano processes it,
- Then it is assigned one of the defined tactic types: `fingerprinting`, `behavioral_analytics`, `session_recording`, `ad_targeting`, `beacon`, `pixel_tracking`, `identity_resolution`, or `unknown`,
- And the classification includes a one-sentence plain-English description of what the snippet was doing.

---

**US-202**
As Sam, I want Gemini Nano classification to run locally without sending code to external servers so that I can safely analyze captures from sensitive pages.

*Acceptance criteria:*
- Given the Chrome Prompt API is enabled (either via Origin Trial flag or Web Store token),
- When classification runs during capture,
- Then all inference is performed by the Gemini Nano model running in Chrome's on-device AI subsystem,
- And no snippet content is transmitted to Google or any external API endpoint,
- And the extension manifest does not declare any `externally_connectable` permissions pointing to AI cloud endpoints.

---

**US-203**
As Alex, I want minified and obfuscated JavaScript snippets to be classified correctly so that common evasion techniques do not produce `unknown` results.

*Acceptance criteria:*
- Given a snippet consists of hex-encoded strings or eval-based obfuscation typical of fingerprinting libraries,
- When Gemini Nano classifies it,
- Then the tactic type is correctly identified as `fingerprinting` or the appropriate tactic with confidence above the defined threshold,
- And if the snippet exceeds the 4096-token context window, it is split into chunks and each chunk is classified independently before results are merged.

---

**US-204**
As Sam, I want to see the severity rating for each classified snippet so that I can prioritize which findings to turn into filter rules first.

*Acceptance criteria:*
- Given a snippet is classified,
- Then it receives a severity rating of `low`, `medium`, or `high`,
- Where `high` covers identity resolution and cross-site tracking, `medium` covers behavioral analytics and session recording, and `low` covers first-party analytics and simple pixel fires,
- And severity is stored in the database and exposed in the API and public JSON export.

---

**US-205**
As Alex, I want to review the classifications for a specific capture before they are submitted to the public database so that I can choose not to submit captures from sensitive personal pages.

*Acceptance criteria:*
- Given a capture has completed and snippets have been classified,
- Then the popup (or a linked review page) shows a list of classified snippets before submission,
- And I can choose "Submit to public DB" or "Keep local only",
- And if I choose local only, the snippets are stored in the server's SQLite DB but not pushed to the GitHub public feed,
- And this preference can be set as a global default in extension settings.

---

## Epic 3: Public Tactic Database Contribution

**Theme:** Help the community build better defenses.

---

**US-301**
As Sam, I want classified snippets to be automatically exported to uBlock Origin filter format so that my capture findings can be immediately usable by the broader community.

*Acceptance criteria:*
- Given new snippets have been pushed to the `spacedudem/amber-tracker-db` GitHub repository,
- When the GitHub Actions workflow runs,
- Then `exports/ublock_filters.txt` is regenerated and contains syntactically valid uBlock Origin static filter rules for all domains identified in high-severity findings,
- And the file passes uBlock Origin's filter linter with zero errors.

---

**US-302**
As Sam, I want the public database to include raw code snippets (anonymized) so that I can study implementation patterns and write more precise filter rules.

*Acceptance criteria:*
- Given a snippet is marked for public submission,
- Then before it is written to `tracker_tactics.json`, all referring URLs, user identifiers, and session tokens are stripped from the snippet context,
- And the snippet entry includes: `domain`, `tactic_type`, `severity`, `raw_snippet` (truncated to 500 chars), `context` (surrounding code, 200 chars either side), and `captured_at` (date only, no time),
- And no entry contains `user_id`, `session_id`, or any unique identifier linkable to the submitting user.

---

**US-303**
As Alex, I want to see which domains appear most frequently in the public tactic database so that I can identify the most pervasive trackers across sites.

*Acceptance criteria:*
- Given the public GitHub Pages site for amber-tracker-db,
- Then a `summary.json` file is generated by the export workflow,
- And it lists the top 50 domains by finding count, with per-domain breakdowns by tactic type,
- And this file is regenerated on every push to the public repository.

---

**US-304**
As Sam, I want the public database to export in Pi-hole and hosts.txt formats so that users with local DNS blockers can benefit from amber findings.

*Acceptance criteria:*
- Given the export workflow runs,
- Then `exports/hosts.txt` contains one `0.0.0.0 <domain>` line per unique tracker domain identified with severity `medium` or `high`,
- And `exports/pihole.txt` contains one domain per line in Pi-hole-compatible format,
- And both files include a header comment with the generation timestamp and total entry count.

---

**US-305**
As Jordan, I want captures I mark as evidence to be excluded from the public database entirely so that sensitive investigation targets are not disclosed.

*Acceptance criteria:*
- Given I flag a capture as "evidence" in the extension popup,
- Then no snippet data from that capture is written to the public GitHub repository,
- And the capture is stored only in the local server SQLite database,
- And the archive HTML is stored only on my server with no public URL,
- And this exclusion flag is stored permanently and cannot be accidentally reversed by a batch export.

---

## Epic 4: Archive Management

**Theme:** Find and use my saved archives.

---

**US-401**
As Alex, I want to list all my saved archives sorted by capture date so that I can quickly find a page I archived last week.

*Acceptance criteria:*
- Given archives have been saved to the server,
- When I call `GET /api/archives`,
- Then the response is a JSON array sorted by `captured_at` descending,
- And each entry includes: `id`, `url`, `title`, `captured_at`, `asset_count`, `snippet_count`, and a direct link to the archive HTML,
- And the response returns within 500ms for up to 10,000 archived pages.

---

**US-402**
As Jordan, I want to search my archives by original URL or page title so that I can retrieve a specific capture without scrolling through hundreds of entries.

*Acceptance criteria:*
- Given I have more than 50 saved archives,
- When I pass a `?q=` query parameter to `GET /api/archives`,
- Then results are filtered to archives where the original URL or title contains the query string (case-insensitive),
- And results return within 300ms,
- And if no matches are found, an empty array is returned with HTTP 200 (not 404).

---

**US-403**
As Alex, I want to open a saved archive directly in a new browser tab so that I can read it as I would the original page.

*Acceptance criteria:*
- Given an archive exists at a stable URL on the server,
- When I open that URL in Chrome,
- Then the page renders with correct layout, images, and fonts,
- And the browser Network panel shows zero outbound requests,
- And the page title in the browser tab matches the original page title.

---

**US-404**
As Morgan, I want to download a saved archive as a single self-contained HTML file so that I can attach it to a legal filing or share it with co-counsel.

*Acceptance criteria:*
- Given an archive is stored on the server,
- When I request `GET /api/archives/{id}/download`,
- Then I receive a single HTML file with all assets embedded as data URIs,
- And the file is served with `Content-Disposition: attachment` and a filename of `amber-{domain}-{date}.html`,
- And the file opens correctly when double-clicked in Windows Explorer or macOS Finder with no internet connection.

---

**US-405**
As Sam, I want to filter my archives by tactic type so that I can review all pages where canvas fingerprinting was detected.

*Acceptance criteria:*
- Given I call `GET /api/archives?tactic=fingerprinting`,
- Then the response includes only archives that have at least one snippet classified with tactic_type `fingerprinting`,
- And each result includes the count of fingerprinting snippets found on that page,
- And I can combine filters (e.g., `?tactic=fingerprinting&severity=high`).

---

## Epic 5: Configuration & Setup

**Theme:** Get amber working my way.

---

**US-501**
As Jordan, I want a guided first-run setup flow in the extension popup so that I can configure the server URL and verify the connection without reading documentation.

*Acceptance criteria:*
- Given I have installed the extension and have not yet configured a server URL,
- When I click the amber toolbar icon for the first time,
- Then the popup displays a setup screen with a labeled text field for Server URL,
- And a "Test connection" button that calls `GET /health` on the entered URL and displays success or error,
- And on first successful connection, the setup screen does not appear again.

---

**US-502**
As Alex, I want to enable the Gemini Nano Chrome flag from within the extension setup so that I do not have to remember the exact chrome://flags path.

*Acceptance criteria:*
- Given I am on the setup screen and Gemini Nano is not yet available (window.ai is undefined),
- Then the setup screen displays a direct deep-link button labeled "Enable Gemini Nano in Chrome flags",
- And clicking it opens `chrome://flags/#optimization-guide-on-device-model` in a new tab,
- And after I enable the flag and relaunch Chrome, the setup screen detects window.ai availability and shows a green checkmark.

---

**US-503**
As Jordan, I want to configure whether captures are automatically submitted to the public database or default to local-only so that I do not accidentally publish sensitive captures.

*Acceptance criteria:*
- Given I am in extension settings,
- Then I can set a global default: "Always submit to public DB", "Always keep local", or "Ask me each time",
- And the default is "Ask me each time" on first install,
- And this preference is stored in Chrome's `sync` storage so it persists across Chrome profiles on the same account.

---

**US-504**
As Sam, I want to configure the minimum severity threshold for public submission so that only high-value findings reach the community database.

*Acceptance criteria:*
- Given I am in extension settings,
- Then I can set a minimum severity for auto-submission: `low`, `medium`, or `high`,
- And snippets classified below that threshold are stored locally but never pushed to the GitHub feed,
- And the default threshold is `medium`.

---

## Use Cases (Formal)

---

### UC-001: Capture a Cloudflare-Protected Article

**Use Case ID:** UC-001
**Name:** Capture Cloudflare-Protected Article
**Primary Actor:** Jordan (Investigative Journalist)
**Supporting Actors:** Chrome Extension (amber), Go Server, Chrome Networking Stack
**Preconditions:**
1. The amber extension is installed and configured with a valid server URL.
2. The Go server is running and reachable at the configured URL over HTTPS.
3. Jordan has manually navigated to the target page in Chrome and passed any Cloudflare challenge; the page is fully rendered.
4. Gemini Nano is available (window.ai is defined) or the extension is configured to skip classification.

**Trigger:** Jordan clicks the amber extension toolbar icon.

**Main Flow:**
1. The popup opens and displays a "Capture this page" button. Jordan clicks it.
2. `popup.js` sends a `START_CAPTURE` message to `background.js`.
3. `background.js` injects `content.js` into the active tab.
4. `content.js` serializes the current DOM using `document.documentElement.outerHTML`, capturing the fully-rendered state including all JS-injected nodes.
5. `content.js` walks the DOM and removes: all `<script>` elements, all inline event handler attributes (onclick, onload, onmouseover, etc.), `<img>` and `<iframe>` elements with dimensions of 1x1 (tracking pixels), elements with data attributes matching the tracker attribute pattern list, and `<ins>` ad container elements.
6. For each removed element or attribute, `content.js` records a snippet entry: `{type, preview, outer_html_truncated}`.
7. `content.js` collects all unique asset URLs (img src, CSS href, font src) referenced in the cleaned HTML.
8. `content.js` sends `{cleaned_html, snippet_list, asset_urls, page_title, page_url}` to `background.js`.
9. `background.js` fetches each asset URL using `fetch()` with `credentials: 'include'`, carrying Jordan's session cookies. Cloudflare-gated CDN assets are retrieved successfully.
10. Each fetched asset is base64-encoded and its reference in the HTML is replaced with a data URI.
11. `background.js` calls `window.ai.languageModel.create()` and passes each snippet for tactic classification. Snippets exceeding 4096 tokens are chunked before classification.
12. Each snippet receives a `tactic_type` and `severity` from Gemini Nano.
13. `background.js` assembles the final payload: `{html_with_embedded_assets, snippets_with_classifications, metadata}` and POSTs it to `/api/archive` on the configured server.
14. The Go server writes the HTML to disk, stores snippet data in SQLite, and returns `{archive_id, archive_url}`.
15. `background.js` sends the result to `popup.js`.
16. The popup displays "Archive saved" with the archive URL and snippet count.

**Alternative Flow A — Asset Fetch Failure:**
- At step 9, if a CDN asset returns 403 or times out after 5 seconds, it is skipped.
- The asset URL is retained as a broken reference in the HTML (img src preserved but not embedded).
- After all assets are attempted, the flow continues at step 11.
- At step 16, the popup notes: "14 assets embedded, 2 assets skipped."

**Alternative Flow B — Server Unreachable:**
- At step 13, if the POST to `/api/archive` fails with a network error or non-2xx response,
- `background.js` stores the payload in Chrome's `localStorage` as a pending upload,
- The popup displays: "Upload failed. Archive queued for retry."
- The extension retries every 60 seconds until the server responds successfully.

**Alternative Flow C — Gemini Nano Unavailable:**
- At step 11, if `window.ai` is undefined,
- Snippets are stored with `tactic_type: "unclassified"` and `severity: null`,
- The flow continues at step 13 without classification,
- The popup notes: "Gemini Nano unavailable — snippets stored unclassified."

**Postconditions:**
1. A static, script-free HTML file is stored on the Go server with a stable URL.
2. All stripped snippets are recorded in SQLite with tactic classifications (or marked unclassified).
3. A `amber-manifest.json` accompanies the archive listing all stripped elements.
4. The archive renders identically to the original page in an offline browser.
5. The archive URL is shown in the popup and accessible via `GET /api/archives`.

**Exceptions:**
- If the target page navigates away during capture (redirect, JS navigation), `content.js` serializes whatever DOM state exists at the moment of injection and notes `capture_interrupted: true` in the manifest.
- If `document.documentElement.outerHTML` returns an empty string (rare with SPAs in loading states), the extension reports "Page not ready — scroll to bottom and retry."

---

### UC-002: Classify and Submit Tracking Snippet

**Use Case ID:** UC-002
**Name:** Classify and Submit Tracking Snippet
**Primary Actor:** Sam (Security Researcher / Filter List Maintainer)
**Supporting Actors:** Gemini Nano (Chrome AI), Go Server, GitHub Actions, Public GitHub Repository
**Preconditions:**
1. A capture has completed and at least one snippet has been extracted.
2. Gemini Nano is available and the Chrome AI flag is enabled.
3. The extension's submission preference is set to "Ask me each time" or "Always submit."
4. The Go server has write access to push to `spacedudem/amber-tracker-db`.

**Trigger:** Capture completes; `background.js` begins the classification pipeline.

**Main Flow:**
1. `background.js` iterates over the `snippet_list` produced by `content.js`.
2. For each snippet, if `outer_html.length` exceeds 3800 characters (~4096 tokens estimated), it is split on function boundaries or semicolons into chunks of <= 3800 characters.
3. A classification prompt is constructed: `"Classify this web tracking code by tactic type. Types: fingerprinting, behavioral_analytics, session_recording, ad_targeting, beacon, pixel_tracking, identity_resolution, unknown. Respond with JSON: {tactic_type, severity: low|medium|high, description}. Code:\n\n{chunk}"`.
4. `window.ai.languageModel` processes each chunk. Results for split snippets are merged: tactic_type is the highest-confidence result, severity is the maximum across chunks.
5. Each classified snippet is stored in the server's `snippets` table via `POST /api/snippets` with fields: `domain`, `url`, `tactic_type`, `severity`, `raw_snippet`, `context`, `captured_at`.
6. If the submission preference is "Ask me each time," the popup presents a review list and waits for Sam to choose "Submit to public DB" or "Keep local."
7. Sam clicks "Submit to public DB."
8. The server anonymizes the snippets: strips the referring page URL (stores domain only), removes any string literals matching session token patterns (32+ hex chars, JWT format, UUID v4), and truncates `raw_snippet` to 500 characters.
9. The server commits the anonymized entries to the local SQLite `exports` staging table.
10. A scheduled job (or immediate trigger) pushes the updated `tracker_tactics.json` to `spacedudem/amber-tracker-db` via the GitHub API.
11. GitHub Actions detects the push and runs the export workflow: regenerates `ublock_filters.txt`, `hosts.txt`, `pihole.txt`, `disconnect.json`, and `summary.json`.
12. The workflow commits the regenerated exports to the repository.
13. The popup (or a notification) confirms: "3 snippets submitted to public database."

**Alternative Flow A — Low Confidence Classification:**
- At step 4, if Gemini Nano returns a `tactic_type` of `unknown` for a snippet,
- The snippet is stored locally with `tactic_type: "unknown"` and is not submitted to the public database automatically,
- Sam sees it flagged in the review list as "Unclassified — review manually,"
- Sam may override the classification manually before submission.

**Alternative Flow B — Snippet Below Severity Threshold:**
- At step 9, if the snippet severity is below Sam's configured minimum threshold (e.g., threshold is `medium` and snippet is `low`),
- The snippet is stored in local SQLite only,
- It is not pushed to the GitHub repository,
- The popup notes: "2 snippets kept local (below severity threshold)."

**Postconditions:**
1. Classified snippets are stored in the server's SQLite database with full metadata.
2. Snippets approved for public submission are anonymized and present in `tracker_tactics.json` in the public repository.
3. uBlock, Pi-hole, hosts.txt, and Disconnect exports are regenerated and up to date.
4. No user identifiers or full page URLs are present in any public data.

**Exceptions:**
- If the GitHub push fails (rate limit, network error), the server queues the export for retry with exponential backoff up to 1 hour.
- If Gemini Nano is unresponsive for a snippet after 10 seconds, that snippet is marked `tactic_type: "unclassified"` and processing continues.

---

### UC-003: Browse Saved Archives

**Use Case ID:** UC-003
**Name:** Browse Saved Archives
**Primary Actor:** Alex (Privacy-Conscious Researcher)
**Supporting Actors:** Go Server, SQLite Database
**Preconditions:**
1. The Go server is running.
2. At least one archive has been previously captured and saved.
3. Alex is accessing the server over HTTPS (either locally or via nginx reverse proxy).

**Trigger:** Alex wants to retrieve an article they archived three days ago.

**Main Flow:**
1. Alex opens the amber popup and clicks "Browse archives," which opens the archive listing page served by the Go server.
2. The page loads and calls `GET /api/archives?sort=captured_at_desc&limit=50`.
3. The server queries SQLite for archives ordered by `captured_at DESC` and returns the first 50 results as JSON.
4. The page renders a list: each entry shows the page title, original URL (truncated), capture date/time, asset count, and snippet count.
5. Alex sees the article they are looking for listed third in the results.
6. Alex clicks the archive title. The server serves the static archive HTML from disk.
7. The page opens in a new tab, fully rendered, with no outbound network requests.
8. Alex clicks "View manifest" — a JSON viewer in the popup shows the `amber-manifest.json` for this capture.

**Alternative Flow A — Searching by Keyword:**
- At step 2, Alex types "EFF surveillance" into the search box.
- The page calls `GET /api/archives?q=EFF+surveillance`.
- The server queries SQLite: `WHERE title LIKE '%EFF surveillance%' OR original_url LIKE '%EFF surveillance%'`.
- Matching archives are returned and rendered in the list.
- If no results match, the page shows "No archives found for 'EFF surveillance'."

**Alternative Flow B — Filtering by Tactic:**
- Alex clicks the "Fingerprinting" filter chip.
- The page calls `GET /api/archives?tactic=fingerprinting`.
- The server joins the archives and snippets tables to return only archives with at least one fingerprinting snippet.
- Each result in the filtered list shows the fingerprinting snippet count in a badge.

**Alternative Flow C — Downloading for Offline Use:**
- Alex clicks "Download" on a listed archive.
- The server responds to `GET /api/archives/{id}/download` with the HTML file served as `Content-Disposition: attachment`.
- Alex saves the file locally and can open it in any browser without a server.

**Postconditions:**
1. Alex has accessed the desired archive.
2. The archive was read from disk — no re-fetching or JS execution occurred.
3. The server access log records the request.

**Exceptions:**
- If the archive HTML file has been deleted from disk but the database record remains, the server returns HTTP 410 Gone with a message indicating the file is missing.
- If the SQLite database is locked (during a write), read queries are retried up to 3 times with 100ms delays before returning HTTP 503.

---

### UC-004: First-Time Setup

**Use Case ID:** UC-004
**Name:** First-Time Setup
**Primary Actor:** Jordan (Investigative Journalist)
**Supporting Actors:** Go Server (Jordan's IT contact sets it up), Chrome Browser
**Preconditions:**
1. Jordan has Chrome 120 or later installed.
2. The amber extension `.crx` file or unpacked extension directory has been provided to Jordan.
3. The Go server has been installed and started by Jordan's IT contact at a known HTTPS URL.

**Trigger:** Jordan installs the amber extension for the first time.

**Main Flow:**
1. Jordan opens `chrome://extensions`, enables Developer mode, and loads the unpacked extension directory (or installs the `.crx`).
2. The amber icon appears in the Chrome toolbar.
3. Jordan clicks the amber icon. Because no server URL is configured, the setup wizard appears.
4. The wizard shows three steps with status indicators: (1) Server URL, (2) Connection test, (3) Gemini Nano.
5. Jordan enters the server URL provided by IT (e.g., `https://archive.example.com`) into the Server URL field.
6. Jordan clicks "Test connection." The extension calls `GET https://archive.example.com/health`.
7. The server responds with HTTP 200 and `{"status": "ok", "version": "1.0.0"}`.
8. The wizard shows a green checkmark next to "Server URL" and "Connection test."
9. The wizard checks for `window.ai` availability. It is undefined (flag not yet enabled).
10. The wizard shows a yellow warning next to "Gemini Nano": "Gemini Nano is not available. Captures will work but tracking code will not be classified."
11. A button labeled "Enable Gemini Nano (requires Chrome flag)" is shown. Jordan clicks it.
12. Chrome opens `chrome://flags/#optimization-guide-on-device-model` in a new tab.
13. Jordan sets the flag to "Enabled" and clicks "Relaunch."
14. After relaunch, Jordan clicks the amber icon again. The wizard re-checks `window.ai` and finds it defined.
15. All three wizard steps show green checkmarks.
16. Jordan clicks "Finish Setup." The extension stores the server URL in `chrome.storage.sync` and dismisses the wizard.
17. The extension popup now shows the standard "Capture this page" interface.

**Alternative Flow A — Jordan Skips Gemini Nano:**
- At step 10, Jordan clicks "Skip — I'll enable this later."
- The wizard marks the Gemini Nano step as "Skipped" and Jordan proceeds to step 16.
- The popup shows a persistent yellow dot on the Gemini Nano status indicator until the flag is enabled.
- Captures work fully; snippets are stored as unclassified.

**Alternative Flow B — Server Connection Fails:**
- At step 7, the server returns a network error or non-200 response.
- The wizard shows a red error: "Could not connect. Check the URL and ensure the server is running."
- The Server URL field is highlighted and the test button becomes "Retry."
- Jordan corrects the URL and retries. The flow resumes at step 6.

**Postconditions:**
1. The extension is configured with a valid server URL stored in `chrome.storage.sync`.
2. The `GET /health` endpoint has been successfully called at least once, confirming reachability.
3. Gemini Nano availability has been detected and its status is persisted in extension settings.
4. Jordan can immediately capture any page by clicking the amber icon.

**Exceptions:**
- If Chrome's `storage.sync` is unavailable (e.g., not signed into Chrome), the extension falls back to `storage.local` and notes this in a tooltip: "Settings are stored locally only — not synced across devices."
- If the extension is installed in Chrome's Guest mode, setup completes but a warning notes: "Guest mode — archives will not persist between sessions."

---

## Story Map

```
PERSONAS ->     Alex                 Jordan               Sam                  Morgan
                (Researcher)         (Journalist)         (Security/Filters)   (Legal)

EPICS v

EPIC 1:         US-106               US-102               US-106               US-108
PAGE            Capture SPA          Capture CF page      Capture rendered     Capture manifest
CAPTURE &       DOM after JS         with session         DOM                  of stripped items
ARCHIVE
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                US-103               US-107               US-103               US-104
                Zero JS in           Success/failure      Zero JS              Timestamp for
                archive              popup notice         in archive           evidence
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                US-105               US-104               US-105               US-404
                Embed all            Timestamp on         Embed assets         Download single
                assets offline       capture              offline              HTML for filing
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                US-101               US-108               US-101               US-101
                One-click            Manifest for         One-click            One-click
                capture              legal record         capture              capture

=====================================================================================
                                    MVP LINE
=====================================================================================

EPIC 2:         US-201               --                   US-203               US-201
TRACKING        Classify by          --                   Classify obf.        Classification
CODE            tactic type          --                   JS correctly         in manifest
CLASSIFICATION
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                --                   --                   US-202               --
                --                   --                   Local inference      --
                --                   --                   only (no cloud)      --
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                US-205               US-205               US-204               --
                Review before        Review before        Severity             --
                submitting           submitting           ratings              --

EPIC 3:         US-303               US-305               US-301               --
PUBLIC          Domain               Exclude evidence     uBlock export        --
TACTIC DB       frequency            from public DB       generation           --
CONTRIBUTION
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                --                   --                   US-302               --
                --                   --                   Raw snippets         --
                --                   --                   anonymized           --
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                --                   US-305               US-304               --
                --                   Evidence flag        Pi-hole and          --
                --                   local-only           hosts.txt export     --

EPIC 4:         US-401               US-402               US-405               US-404
ARCHIVE         List by              Search by URL        Filter by            Download
MANAGEMENT      capture date         or title             tactic type          for filing
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                US-403               US-403               US-403               US-403
                Open archive         Open archive         Open archive         Open archive
                in tab               in tab               in tab               in tab

EPIC 5:         US-502               US-501               US-504               US-501
CONFIG &        Enable Gemini        Guided setup         Severity             Guided setup
SETUP           Nano flag            flow                 threshold            flow
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
                US-503               US-503               US-503               --
                Default submit       Default local        Default submit       --
                preference           preference           preference           --

=====================================================================================
                              POST-MVP (v2+)
=====================================================================================

FUTURE:         Full session         Cryptographic        Batch domain         OSINT/evidence
v2+ FEATURES    recording            seal + chain         analysis API         sealed package
                (pre-approved)       of custody           (export access)      (pre-approved)
```

---

*Document version 1.0 — generated 2026-06-16*
*Minimum viable product delivers Epic 1 (full), Epic 2 (classification pipeline), Epic 4 (basic archive listing and open), and Epic 5 (setup wizard). Epics 3 (public DB contribution) and advanced archive management are post-MVP but pre-designed.*
