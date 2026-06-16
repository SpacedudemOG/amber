# amber — UX Wireframes

**Version:** 1.0  
**Last Updated:** 2026-06-16  
**Scope:** Extension Popup, Options Page, Archive Metadata Panel, First-Time Setup Flow

---

## Design Principles

1. **One-click primary action.** Capture takes exactly one click from the popup. There is no confirmation dialog, no modal, no secondary prompt. The user is already on the page and has already decided to save it.
2. **Progressive disclosure.** Status detail expands as the capture progresses. The idle state shows minimal chrome; the in-progress state surfaces a live removal log so the user can see work happening; the completion state shows the full breakdown.
3. **Trust signals.** Every stripped element is named and counted. The user sees exactly what was removed and why it matters (tactic type and severity). Transparency is a core feature, not a detail buried in logs.
4. **Zero friction setup.** Sensible defaults are pre-filled. The setup flow is three discrete steps, each completable in under 30 seconds. Nothing requires the user to read documentation before capturing their first page.
5. **Offline-first resilience.** When the server is unavailable, amber saves the stripped HTML locally and queues the upload. The UI communicates this clearly so the user never loses a capture.

---

## Surface 1: Extension Popup (320 x 400 px)

The popup is the primary interaction surface. It opens when the user clicks the amber toolbar icon. It is 320 px wide and up to 400 px tall. All state transitions are animated with a 150 ms ease-in-out fade on the content area. The header row (logo + settings gear) is fixed and does not animate between states.

### Typography and Color

- Header label: 14 px semibold, `#F5C542` (amber yellow)
- Body text: 13 px regular, `#E8E8E8` on `#1A1A1A` background
- Status dot colors: idle = `#F5C542`, active = `#4A9EFF`, success = `#4CAF50`, error = `#E53935`
- Progress bar fill: `#4A9EFF`; track: `#2E2E2E`
- Dividers: 1 px `#333333`
- Capture button: `#F5C542` background, `#1A1A1A` text, 6 px border-radius, full-width minus 16 px padding each side

### State 1a: Idle (Ready to Capture)

```
┌─────────────────────────────────────┐
│  🟡 amber                        ⚙️  │
├─────────────────────────────────────┤
│                                     │
│  axios.com                          │
│  Google's AI trade worker...        │
│                                     │
│  ┌─────────────────────────────┐    │
│  │     📸  Capture Page        │    │
│  └─────────────────────────────┘    │
│                                     │
│  Last capture: 2 hours ago          │
│  paulgraham.com › essays            │
│                                     │
│  ─────────────────────────────────  │
│  📁 Browse Archives  🌐 Tracker DB  │
└─────────────────────────────────────┘
```

**Content details:**

- The current tab's hostname is displayed in 15 px semibold. The page title is truncated at 38 characters with an ellipsis if longer.
- If the current tab is a Chrome internal page (`chrome://`, `chrome-extension://`, `about:`, `file://`), the Capture button is replaced with a disabled state labeled "Not a web page" in gray, and a subtitle reads "Navigate to a website to capture."
- "Last capture" shows the hostname and truncated path of the most recent successful capture, with a human-readable relative timestamp. If no captures exist yet, this line reads "No captures yet."
- "Browse Archives" opens `korh.one/amber/archives/` in a new tab.
- "Tracker DB" opens `github.com/spacedudem/amber-tracker-db` in a new tab.
- The gear icon opens the Options page (`chrome.runtime.openOptionsPage()`).

**Keyboard interaction:** Tab order is Capture Page button → Browse Archives → Tracker DB → gear icon. The Capture Page button is focused by default when the popup opens. Pressing Enter or Space on it triggers capture. Pressing Escape closes the popup.

### State 1b: Capturing — Stripping DOM

This state activates immediately when the user clicks Capture Page. The content area transitions to a live status feed. The progress bar advances in discrete increments as stages complete.

```
┌─────────────────────────────────────┐
│  🔵 amber                        ⚙️  │
├─────────────────────────────────────┤
│                                     │
│  Stripping page...                  │
│  ████████░░░░░░░░░░░░  35%          │
│                                     │
│  ✂️  Removed 4 script tags          │
│  ✂️  Removed 2 tracking pixels      │
│  ⏳  Fetching 23 assets...          │
│                                     │
│                                     │
│             [ Cancel ]              │
└─────────────────────────────────────┘
```

**Progress stages and percentages:**

| Stage | % | Label |
|---|---|---|
| Serializing DOM | 0–10 | "Reading page..." |
| Removing scripts | 10–25 | "Stripping scripts..." |
| Removing inline handlers | 25–35 | "Stripping page..." |
| Removing trackers/pixels | 35–50 | "Stripping page..." |
| Fetching assets | 50–75 | "Fetching N assets..." |
| Encoding assets | 75–80 | "Packaging..." |
| Uploading to server | 80–95 | "Saving archive..." |
| Done | 100 | transitions to State 1d |

The removal log shows lines as they are emitted by content.js. Lines appear with a 50 ms stagger. If more than 5 lines accumulate, the list scrolls within a 120 px fixed-height box (overflow: hidden, no scrollbar visible — the area auto-scrolls to the newest item). Each line uses a ✂️ icon for removals, ⏳ for pending, and ✅ for completed sub-stages.

Cancel sends a `cancelCapture` message to the background service worker, which aborts any in-flight fetch calls. Local state is cleared. The popup returns to State 1a.

### State 1c: Classifying with Gemini Nano

This state is entered when asset fetching is complete and Gemini Nano classification begins. It is separated from State 1b because classification is asynchronous and may take 3–10 seconds per batch depending on snippet length and device GPU availability.

```
┌─────────────────────────────────────┐
│  🔵 amber                        ⚙️  │
├─────────────────────────────────────┤
│                                     │
│  Classifying trackers...            │
│  ████████████████░░░░  80%          │
│                                     │
│  🧠 Gemini Nano analyzing           │
│     8 stripped snippets             │
│                                     │
│  Snippet 3 of 8...                  │
│                                     │
│             [ Cancel ]              │
└─────────────────────────────────────┘
```

If Gemini Nano is unavailable (either the flag is not set, or the model is not downloaded, or the device does not meet requirements), the classifier step is skipped silently. The progress bar jumps from 75% directly to 80% and the state skips to uploading. A note appears in State 1d indicating that classification was not available.

The "Snippet N of M..." line updates as each snippet is submitted to the Prompt API. Snippets longer than 3000 tokens are split before submission and counted as separate items in the progress counter.

### State 1d: Complete

```
┌─────────────────────────────────────┐
│  🟢 amber                        ⚙️  │
├─────────────────────────────────────┤
│                                     │
│  ✅ Captured!                       │
│                                     │
│  Stripped:                          │
│  • 4 scripts (2 fingerprinting,     │
│    1 beacon, 1 analytics)           │
│  • 3 tracking pixels                │
│  • 7 inline event handlers          │
│                                     │
│  📦 23 assets saved locally         │
│  🌐 8 tactics contributed to DB     │
│                                     │
│  ┌─────────────────────────────┐    │
│  │     📂  View Archive        │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

The stripped breakdown shows tactic subtypes in parentheses only when Gemini Nano classified them. If classification was skipped, the line reads "• 4 scripts (unclassified)" in a muted gray. Tactic subtypes are drawn from the `tactic_types` table: fingerprinting, beacon, analytics, session-replay, ad-pixel, a/b-testing, consent-gate.

"📦 N assets saved locally" reflects the count of binary assets embedded in the archive as base64 data URIs. "🌐 N tactics contributed to DB" reflects the count of snippets successfully posted to `/api/snippets`. If the server was unreachable, this line reads "⏳ N tactics queued for DB — will retry" in amber yellow.

"View Archive" opens the saved archive URL (`korh.one/amber/archives/<domain>/<slug>/`) in a new tab.

After 30 seconds of inactivity on the complete screen, the popup returns to State 1a automatically, resetting to show the current tab info. This prevents users from reopening the popup and seeing stale capture results from a previous session.

### State 1e: Error

```
┌─────────────────────────────────────┐
│  🔴 amber                        ⚙️  │
├─────────────────────────────────────┤
│                                     │
│  ⚠️  Capture failed                  │
│                                     │
│  Could not reach amber server.      │
│  Check server URL in settings.      │
│                                     │
│  The stripped HTML has been         │
│  saved locally and will retry       │
│  when server is available.          │
│                                     │
│  [ Check Settings ]  [ Retry ]      │
└─────────────────────────────────────┘
```

The error message adapts based on failure type:

- **Server unreachable:** "Could not reach amber server. Check server URL in settings."
- **Auth failure (401/403):** "Authentication failed. Check your API key in settings."
- **Server error (5xx):** "Server returned an error (503). The archive has been queued locally."
- **DOM serialization failure:** "Could not read this page. It may be a PDF or special browser page."
- **Asset fetch partial failure:** "Captured with N missing assets. Some images or fonts may not display." (non-fatal, transitions to State 1d with a warning note instead)

"Check Settings" and "Retry" are equal-width buttons split 50/50 across the popup width minus 32 px total padding.

---

## Surface 2: Options / Settings Page

Opened via `chrome://extensions/` or the gear icon in the popup. Rendered as a full browser tab using `options_page` in manifest.json, not an inline popup panel. This allows sufficient width for form controls and avoids the 400 px popup constraint.

The page uses a two-column layout at viewport widths above 900 px (settings sections left, contextual help text right). Below 900 px it collapses to a single column. Maximum content width is 720 px, centered.

```
amber — Settings
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SERVER CONFIGURATION
  Server URL   [ https://korh.one/amber/api/v1              ]
  API Key      [ ••••••••••••••••••••                       ] 👁
               [ Test Connection ]   ✅ Connected (42 ms)

CAPTURE BEHAVIOR
  [✓] Strip all <script> tags
  [✓] Strip inline event handlers  (onclick, onmouseover, etc.)
  [✓] Strip tracking pixels  (1x1 img, beacon fetch calls)
  [✓] Strip <iframe> elements
  [✓] Strip data-tracking-* and data-analytics-* attributes
  [ ] Strip <link rel="preconnect">  (may break web fonts)
  [ ] Strip <noscript> fallback blocks

GEMINI NANO CLASSIFIER
  Status       ✅ Available — Gemini Nano loaded (Chrome 130+)
  [✓] Enable AI classification of stripped snippets
  [✓] Contribute anonymized findings to public tracker DB
  [ ] Show snippets for review before submitting  (advanced)

  Classifier prompt language:   [English ▾]

ARCHIVE STORAGE
  Archives are saved to the server path configured above.
  Local queue (pending upload): 0 items
  [ Clear Queue ]

PRIVACY
  [✓] Hash domain names in public submissions
  [✓] Strip referring URLs from public submissions
  [ ] Include classifier confidence scores in public submissions
  [ ] Log capture activity to browser console  (debug)

  [ Export My Submissions (JSON) ]    [ Delete All Local Data ]

ABOUT
  amber extension    v1.0.0
  server             kage v0.4.0
  [ View on GitHub ]    [ View Public Tracker DB ]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[ Save Settings ]
```

### Field-Level Behavior

**Server URL:** Validated on blur. Must be a valid HTTPS URL. HTTP is rejected with the inline error "amber requires HTTPS." Trailing slashes are stripped automatically.

**API Key:** Stored in `chrome.storage.local` (not `sync`, to avoid syncing credentials across devices). The 👁 icon toggles between masked (`••••`) and plaintext display. The key is never logged to the console.

**Test Connection:** Fires a `GET /health` request to the configured URL with the API key in an `Authorization: Bearer` header. Displays latency in milliseconds on success. On failure, shows the HTTP status code and a short human-readable reason.

**Strip `<link rel="preconnect">`:** Disabled by default because stripping these tags causes Google Fonts and similar CDN-loaded fonts to fail silently when the archive is reopened. The checkbox label includes an inline caveat in muted text.

**Show snippets for review before submitting:** When enabled, after classification the popup enters an additional state (not shown in the popup wireframes above because it is an advanced opt-in) where a scrollable list of classified snippets appears with individual checkboxes. The user can deselect any snippet before submission. This is intended for privacy researchers who want to manually vet each contribution.

**Classifier prompt language:** Allows the user to select the language in which tactic descriptions are written by Gemini Nano. Default is English. Other options are French, German, Spanish, Japanese. This affects only the human-readable description field in the DB, not the tactic_type classification.

**Export My Submissions:** Generates a JSON file from `chrome.storage.local` containing all snippets the extension has submitted. Download is triggered via `URL.createObjectURL`. The file name is `amber-submissions-YYYYMMDD.json`.

**Delete All Local Data:** Shows a confirmation dialog: "This will clear all local queues, submission history, and settings. Captures already saved to the server are not affected. Continue?" Two buttons: "Delete Everything" (destructive, red) and "Cancel."

**Save Settings:** Settings are written to `chrome.storage.local`. A brief confirmation toast ("Settings saved") appears at the bottom of the page for 2 seconds, then fades out. If Test Connection has not been run since the URL or key was changed, a yellow advisory note appears next to Save: "You have unsaved server changes. Consider testing the connection first."

---

## Surface 3: Archive Metadata Panel (injected into korh.one/amber/archives/)

The archive browser is served by the existing nextexplorer Docker container. amber does not replace or rebuild this UI. Instead, the extension injects a metadata panel at the top of each archive page when the user views a saved capture. The panel is injected via a content script that runs on `korh.one/amber/archives/*` URLs.

The panel is collapsible. Default state is expanded. Collapsed, it shows a single line: "📦 amber archive — 8 tactics stripped — click to expand." A chevron icon on the right toggles expand/collapse. The collapsed/expanded state is persisted in `sessionStorage` so it survives page navigations within the same tab session.

```
┌─────────────────────────────────────────────────────────────┐
│  📦 amber archive                              2026-06-15   │
│  Original: https://axios.com/2026/06/11/google-ai-trade/    │
├─────────────────────────────────────────────────────────────┤
│  REMOVED FROM THIS PAGE:                                    │
│  ┌────────────────────────────────────────────────────┐    │
│  │ ⚠️  HIGH   canvas fingerprinting    (2 instances)  │    │
│  │ ⚠️  HIGH   behavioral tracking      (mouse+scroll) │    │
│  │ 🟡 MED    analytics beacon          (3 instances)  │    │
│  │ 🟡 MED    tracking pixel            (CloudFront)   │    │
│  │ 🟢 LOW    localStorage session ID   (1 instance)   │    │
│  └────────────────────────────────────────────────────┘    │
│  8 tactics stripped • 23 assets localized                   │
│  ✅ This page works fully offline — no external requests    │
│                                 [ ∧ Collapse panel ]        │
└─────────────────────────────────────────────────────────────┘
```

**Severity legend:**

- ⚠️ HIGH — tactics that actively identify or persistently track the user across sessions or devices (canvas fingerprinting, font fingerprinting, cross-site tracking pixels, session replay scripts).
- 🟡 MED — tactics that collect behavioral or contextual data within a session (analytics beacons, A/B test assignment scripts, referrer collection).
- 🟢 LOW — tactics that are privacy-relevant but limited in scope or easily cleared (localStorage identifiers, first-party cookies referenced via JS, consent gate scripts).

Each row in the tactics list is clickable. Clicking a row expands an inline detail view showing the raw stripped snippet (truncated to 300 characters, with a "Show full snippet" toggle), the classifier confidence score if available, and a link to the corresponding entry in the public tracker DB on GitHub.

The "Original:" URL is rendered as a non-clickable plain-text string (not an `<a>` tag) to prevent accidental navigation to the original, potentially tracking-heavy page. If the user wants to return to the original, they must manually copy and paste the URL.

---

## Surface 4: First-Time Setup Flow

Shown on first install via the `chrome.runtime.onInstalled` event. Opens as a full-tab page (`chrome-extension://<id>/setup.html`). Three discrete steps, no back button required from step 3 (steps 1 and 2 have a Back button). Progress is shown with a simple "Step N of 3" indicator at the top right.

### Step 1 of 3: Enable Gemini Nano

```
┌─────────────────────────────────────────────────────────────┐
│  amber setup                                    Step 1 of 3 │
│                                                             │
│  Enable Gemini Nano (optional but recommended)              │
│  ─────────────────────────────────────────────              │
│  amber uses Chrome's built-in AI to classify tracking       │
│  code into categories like fingerprinting, analytics,       │
│  and session replay. Classification runs 100% locally       │
│  — no data leaves your device during this step.             │
│                                                             │
│  To enable:                                                 │
│                                                             │
│  1. Open this URL in a new tab:                             │
│     chrome://flags/#optimization-guide-on-device-model      │
│     [ Copy URL ]                                            │
│                                                             │
│  2. Set the flag to: "Enabled BypassPerfRequirement"        │
│                                                             │
│  3. Click "Relaunch" at the bottom of the flags page.       │
│                                                             │
│  4. Return here. Chrome will reopen this setup page.        │
│                                                             │
│  Current status:  ❌ Not detected                           │
│  (After relaunching Chrome, this will update automatically) │
│                                                             │
│  [ Skip — disable classification ]   [ I've done this → ]  │
└─────────────────────────────────────────────────────────────┘
```

The "Copy URL" button copies `chrome://flags/#optimization-guide-on-device-model` to the clipboard and changes its label to "Copied ✓" for 2 seconds. The `chrome://flags` URL cannot be opened programmatically via `chrome.tabs.create()` (Chrome blocks this), so the copy-and-paste approach is the correct UX pattern.

After the user relaunches Chrome and this page reopens (the setup state is persisted in `chrome.storage.local`), the extension checks for Gemini Nano availability using `window.ai.languageModel.capabilities()`. If available, "Current status" updates to ✅ and the "I've done this" button becomes the sole primary action.

"Skip" saves `geminiEnabled: false` to storage and advances to Step 2. The user can re-enable classification later from the Settings page.

### Step 2 of 3: Connect to Your Server

```
┌─────────────────────────────────────────────────────────────┐
│  amber setup                                    Step 2 of 3 │
│                                                             │
│  Connect to your amber server                               │
│  ─────────────────────────────────────────────              │
│  amber saves clean archives to a Go server you host.        │
│  If you are using the default korh.one deployment,          │
│  the URL and key are pre-filled below.                      │
│                                                             │
│  Server URL   [ https://korh.one/amber/api/v1              ]│
│  API Key      [                                            ]│
│                                                             │
│  [ Test Connection ]                                        │
│                                                             │
│  (Connection result appears here after test)                │
│                                                             │
│  [ ← Back ]                      [ Save & Continue → ]     │
└─────────────────────────────────────────────────────────────┘
```

The Server URL is pre-filled with `https://korh.one/amber/api/v1` because this is the expected deployment for the primary user. The API Key field is empty; the user must paste it.

"Save & Continue" is disabled until a successful Test Connection result has been received in the current page load. This prevents advancing with a misconfigured server URL. If the user came from Step 1 via "Skip", a note reads: "Gemini Nano classification is disabled. You can enable it later in Settings."

### Step 3 of 3: Ready

```
┌─────────────────────────────────────────────────────────────┐
│  amber setup                                    Step 3 of 3 │
│                                                             │
│  You're ready to capture                                    │
│  ─────────────────────────────────────────────              │
│                                                             │
│  ✅  Gemini Nano       Ready — local AI classification on   │
│  ✅  Server            Connected — korh.one/amber/api/v1    │
│  ✅  Archive storage   /uploads/kage/ (configured on server)│
│                                                             │
│  How it works:                                              │
│  1. Navigate to any web page in Chrome                      │
│  2. Click the amber icon in your toolbar                    │
│  3. Click "Capture Page" — one click, that's it             │
│  4. amber strips all tracking and saves a clean copy        │
│                                                             │
│  Your captures contribute anonymized tactic data to a       │
│  public tracker database. You can opt out in Settings.      │
│                                                             │
│                      [ Start Capturing ]                    │
└─────────────────────────────────────────────────────────────┘
```

"Start Capturing" closes the setup tab and opens the current active tab (the one the user was on before installing). The amber icon in the toolbar is briefly highlighted with a pulsing ring animation (via the `chrome.action.setBadgeText` API) for 5 seconds to draw attention to its location.

If Gemini Nano was skipped in Step 1, the first line reads "⚠️ Gemini Nano — Disabled (pages will be captured without tactic classification)" in amber yellow rather than green.

---

## Interaction Design Notes

### Tab Order and Keyboard Navigation

All interactive elements in the popup and options page follow logical document order for tab navigation. No custom tab index values are used. The popup opens with focus on the Capture Page button so a user can trigger capture by pressing Tab once (to reach the extension icon) and Enter (to open popup) and Enter again (to capture) — three keystrokes total from keyboard.

In the options page, the Save Settings button has `accesskey="s"` so Alt+S (Windows/Linux) or Ctrl+Option+S (macOS) triggers save from anywhere on the page.

### Focus Management

When the popup transitions between states (e.g., from Idle to Capturing), focus moves to the Cancel button to ensure keyboard users can interrupt an in-progress capture. When the transition completes to the Done state, focus moves to the View Archive button.

When the error state appears, focus moves to the "Check Settings" button (first interactive element in error state).

### Screen Reader Support

All icon glyphs (📸, ✂️, 🧠) are supplemented with visually hidden `aria-label` text. The progress bar is a native `<progress>` element with `aria-valuenow`, `aria-valuemin`, and `aria-valuemax` attributes updated live. The live removal log in State 1b uses `aria-live="polite"` so screen readers announce new lines without interrupting speech.

Status dot colors alone are not used to convey meaning — each state also has a distinct text label in the header (the dot is supplementary).

### Animations and Motion

The progress bar fill uses a CSS transition of 300 ms ease-out. State transitions use 150 ms opacity fade. No looping animations are used anywhere — only discrete progress updates. Users with `prefers-reduced-motion: reduce` set in their OS receive no transitions; all state changes are instant.

### Error Recovery

All API call failures are retried once automatically after 3 seconds before displaying an error. Network timeouts are set at 15 seconds for the archive POST (which may include large asset payloads) and 5 seconds for the health check and snippets POST. These values are configurable in the manifest's default storage object but not exposed in the Settings UI to avoid overloading it.

### Mobile and Other Browsers

Chrome on Android is not supported in v1. The manifest does not declare compatibility with Firefox or Safari. If the extension is somehow loaded in a non-Chrome browser, `window.ai.languageModel` will be undefined and the extension will operate in classification-disabled mode without erroring. The core capture, strip, and archive upload functionality relies only on standard Manifest V3 APIs available across browsers, but is only tested and supported on Chrome desktop (Windows, macOS, Linux).
