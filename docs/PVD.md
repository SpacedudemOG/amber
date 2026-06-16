# amber — Product Vision Document

---

## Executive Summary

The web is hostile to archivists. The two dominant forces working against preservation are bot detection systems that block automated scrapers from reaching content at all, and the JavaScript ecosystem that ensures any page you do manage to save continues calling home every time you open it. A saved page that re-fires analytics, loads ad networks, and phones tracking servers is not an archive — it is a liability. The person who saved it gets re-profiled on every reopening, and the content itself may silently change if any of those remote calls succeed.

amber solves both problems by changing the capture model entirely. Instead of scraping from outside the browser, amber operates from inside a real Chrome session — meaning bot detection is never triggered because the user is already authenticated, already past Cloudflare, already holding valid session cookies. Capture happens after JavaScript has fully executed and built the final DOM. amber then strips every script, every inline event handler, every tracking pixel, every beacon call, and every fingerprinting hook before saving. What remains is static HTML plus locally-embedded assets: images, CSS, and fonts fetched through the live session and inlined. The result opens anywhere, works forever offline, and makes zero outbound requests.

Beyond preservation, amber pursues a second mission that no existing tool attempts: systematic documentation of how tracking code actually works at the implementation level. Every stripped snippet is classified by tactic type using Gemini Nano — Chrome's built-in on-device AI — and the resulting database is published as a live, structured feed. Filter list maintainers, privacy researchers, and tool builders have always been able to see which domains serve trackers. amber gives them something they have never had: a public, machine-readable catalog of how those trackers are implemented, snippet by snippet. The timing is right because two things have converged: Chrome's built-in AI (Gemini Nano via the Prompt API) has matured to the point where local, private classification of tracking code is possible without sending data to an external service, and bot detection has reached a sophistication level where the only reliable way to archive challenge-protected content is from inside a real browser session. amber sits at this exact intersection.

---

## Vision Statement

For privacy-conscious users and researchers who need to preserve web content permanently without leaving a trail or feeding surveillance infrastructure, amber is a Chrome extension and archiving system that captures fully-rendered, script-free pages from within the user's live browser session and catalogs every tracking tactic it strips. Unlike SingleFile, HTTrack, or headless scrapers, amber produces archives that are truly static and phonehome-free while simultaneously contributing implementation-level tracking intelligence to a public, community-maintained database.

---

## Target Users

### Primary: Privacy-Conscious Power Users

These are technically literate individuals who regularly save web content for personal reference, research, or evidentiary purposes. They understand that "saving" a page in a conventional browser gives them a snapshot that can rot (link rot, JS-dependent rendering) and that tools like SingleFile preserve surveillance code alongside content. They want permanence and cleanliness. They are comfortable installing a browser extension and running a local server.

**Persona: Maya, 34, independent security journalist**
Maya covers surveillance capitalism and data broker ecosystems. She needs to preserve evidence of specific tracking behaviors before companies quietly update their pages. She currently uses SingleFile, but has noticed that reopening saved pages sometimes triggers analytics events — she can see the network requests in DevTools. She has tried headless scrapers to avoid this, but Cloudflare blocks her on most targets. What she needs is a way to capture a fully-rendered page, strip all tracking code before it can execute on reopen, and keep a record of what was stripped so she can cite it in her reporting. amber is built for Maya.

**Persona: Dmitri, 41, privacy advocate and filter list contributor**
Dmitri maintains a regional language block list for uBlock Origin. He regularly encounters new tracking snippets embedded in inline scripts on news sites and shopping portals — snippets that are not covered by EasyList or Disconnect because they are implemented inline rather than loaded from a known tracker domain. He needs a systematic way to catalog these patterns and export them in a format compatible with uBlock, Pi-hole, and hosts files. The amber tracker database is his contribution pipeline.

### Secondary: Security Researchers

Malware analysts, threat intelligence researchers, and red teamers who need to preserve phishing pages, malvertising payloads, or suspicious sites for offline analysis. They need the JavaScript preserved separately (for forensic review) while the rendered artifact is clean enough to open safely. amber's classification output gives them an immediate triage of what categories of code were present.

**Persona: Yuki, 28, threat intelligence analyst**
Yuki investigates phishing campaigns. She needs to snapshot landing pages before they are taken down. Headless browsers are fingerprinted and often served decoy pages. Wayback Machine crawls are too slow and unreliable. She visits the target in her real browser (which attackers expect), captures it with amber, and gets a clean archive plus a classified list of every suspicious code pattern amber found. She uses the classification output as a starting point for deeper forensic analysis.

### Tertiary: Journalists, Legal Professionals, and OSINT Investigators

People who need to preserve digital evidence for legal proceedings, editorial fact-checking, or investigative dossiers. Their primary need is permanence and trustworthiness of the archive. amber's static output (no JS, no network calls on reopen) ensures the preserved document does not change behavior between capture and review. The v2 OSINT evidence package (cryptographically sealed, timestamped) directly serves this audience.

**Persona: Sebastien, 52, civil litigation attorney**
Sebastien needs to preserve screenshots and page archives as exhibits. He currently uses a process server to take screenshots, which courts have increasingly questioned as not reliably representing what was shown at a specific time. He needs a tool that produces a verifiable, static artifact of what the page looked like and did at capture time, with a tamper-evident audit trail. amber's v2 evidence package is his solution.

### Quaternary: Filter List Maintainers

EasyList, uBlock Origin, Disconnect.me, and similar projects maintain blocklists based on domain-level intelligence. amber's public tracker database exposes implementation-level patterns that could power the next generation of behavioral blocking rules — blocking by code signature rather than domain, enabling detection of first-party tracking that domain-based lists miss entirely.

---

## Problem Space

### Problem 1: Bot Detection Blocks Headless Scrapers

Cloudflare, Akamai, DataDome, and similar bot management systems have become sophisticated enough to reliably identify and block headless browsers even when they impersonate real Chrome. Techniques used include: TLS fingerprint analysis (headless browsers have distinct TLS handshake signatures), JavaScript challenge execution timing analysis, absence of expected browser APIs, WebGL and Canvas fingerprinting, behavior heuristics (mouse movement patterns, timing), and IP reputation.

A real-world example: attempting to archive a news article protected by Cloudflare's Managed Challenge using Puppeteer or Playwright results in a 403 or an infinite challenge loop. This is the problem that kage (amber's sibling project) runs into on a significant fraction of targets. amber sidesteps this entirely because it runs inside the user's real Chrome session. The page has already loaded. Bot detection has already passed. amber captures the rendered result. This is not a marginal improvement over headless approaches — it is a categorical shift. It means amber can capture content that no headless scraper can reach, including pages behind login, pages that require interaction to load content, and pages served by aggressive bot mitigation that defeats even the best stealth tooling.

### Problem 2: Existing Archivers Save JavaScript Intact

SingleFile is the gold standard for single-file page archiving in Chrome. It does excellent work inlining resources. But it preserves script tags and inline event handlers because its goal is to reproduce the page exactly. This means:

- Reopening a SingleFile archive fires analytics events (Google Analytics, Plausible, Segment, etc.)
- Ad containers re-request ads from networks
- Tracking pixels (1x1 images with query string parameters) fire on load
- Fingerprinting code (canvas, WebGL, AudioContext, font enumeration) runs on reopen
- Beacon calls (navigator.sendBeacon) transmit data on page unload

A concrete example: a SingleFile archive of a major news site will, on reopen, send requests to over 40 distinct domains including advertising networks, analytics providers, and data brokers. The user's IP address and browser fingerprint are transmitted to all of them. The "archived" page is not static — it is an active surveillance artifact.

HTTrack downloads entire site trees but with the same fundamental problem: it preserves JavaScript. The Wayback Machine's saved pages partially neutralize JavaScript in some cases, but not consistently, and Wayback requires sending the URL to the Internet Archive's servers.

amber's approach is render-then-strip: let the page fully render (so the user sees exactly what they expect), then serialize the DOM in its rendered state and strip all executable content before saving. The archive is purely structural (HTML) plus inert assets (images, CSS, fonts). Nothing executes on reopen because there is nothing left to execute.

### Problem 3: Tracking Tactics Are Cataloged at the Domain Level, Not the Implementation Level

EasyList, uBlock Origin's filter lists, Disconnect.me's tracker database, and DuckDuckGo's Tracker Radar are all built around domain-level blocking. They know that `analytics.google.com` is a tracking domain and block requests to it. This is effective for most cases.

But a growing fraction of tracking code is implemented inline — embedded directly in the page's HTML or in first-party JavaScript bundles — specifically to defeat domain-based blocking. These include:

- Inline fingerprinting: canvas element manipulation, WebGL renderer enumeration, AudioContext latency measurement — all implemented in inline `<script>` blocks with no external domain request
- First-party proxied beacons: analytics data sent to the site's own domain and proxied server-side to Google Analytics, invisible to any domain-based filter
- Data attribute tracking: `data-track-click`, `data-impression-id`, `data-analytics-event` attributes on DOM elements, processed by inline JavaScript
- Obfuscated identifier injection: user identifiers packed into first-party cookies and localStorage, set by inline code that is never flagged because it makes no requests to known tracker domains

No current tool systematically documents these patterns. amber captures them, classifies them using Gemini Nano (which can understand obfuscated code semantically rather than by pattern matching), and contributes them to a public database. Over time, this database becomes a primary source for researchers building next-generation countermeasures that can handle inline and first-party tracking. There is no public database of how tracking code is implemented — only of which domains serve it. amber fills that gap as a direct product of its core archiving function.

---

## Market Landscape

### SingleFile (Chrome Extension)
**What it does:** Serializes the fully-rendered page into a single self-contained HTML file, inlining all CSS, images, and fonts as data URIs. Excellent fidelity. Widely trusted.
**What it misses:** Preserves all JavaScript, event handlers, and tracking infrastructure intact. Archives phone home on reopen. No classification of stripped content because nothing is stripped.
**Why amber is different:** amber explicitly renders-then-strips. The goal is a clean, inert artifact, not a perfect reproduction. amber trades JS fidelity for privacy and permanence.

### HTTrack (Desktop Application)
**What it does:** Recursively downloads entire websites for offline browsing. Has been the standard offline archiving tool for decades.
**What it misses:** Makes automated HTTP requests that are easily blocked by bot detection. Downloads JavaScript intact. Not designed for single-page capture or for operating within an existing browser session.
**Why amber is different:** amber operates from inside a live browser session, bypassing bot detection. It captures single pages on demand rather than crawling site trees.

### Wayback Machine / Internet Archive (Web Service)
**What it does:** Archives the public web at scale. Maintains historical snapshots accessible via URL. Extremely valuable as a public resource.
**What it misses:** Requires submitting URLs to the Internet Archive — a privacy concern for sensitive targets. Crawls are scheduled and may not capture a specific page state at the moment the user needs it. JavaScript in saved pages is inconsistently neutralized. Cannot capture pages behind authentication.
**Why amber is different:** amber is local, on-demand, and private. No URL is submitted to any external service. The user controls when and what is captured. amber is complementary to the Wayback Machine, not competitive: for public pages both are useful, for authenticated or bot-protected content amber fills a gap the Wayback Machine cannot.

### ArchiveBox (Self-Hosted)
**What it does:** A self-hosted archiving platform that saves pages using multiple methods (wget, Chrome headless, SingleFile, screenshot) and stores them in a local archive. Strong tool for power users.
**What it misses:** Headless Chrome is blocked by bot detection on many targets. Does not strip JavaScript before saving. No tracker classification pipeline.
**Why amber is different:** amber runs in the user's real Chrome session (not headless). It strips rather than preserving executable content. It contributes to a community tracker intelligence database as a side effect of normal use.

### kage (github.com/spacedudem/kage)
**What it does:** amber's sibling project. A Go-based archiver for non-bot-protected sites. Fast, efficient, good for bulk archiving of sites that do not use bot detection.
**What it misses:** Cannot handle Cloudflare-protected or challenge-gated content. Does not strip JavaScript from saved pages. No tracker classification.
**Why amber is different:** amber handles exactly what kage cannot — bot-detection-protected sites, pages behind authentication, and single targeted captures where the user is already present. They are complementary: kage for bulk unprotected archiving, amber for targeted capture of protected or sensitive pages.

---

## Product Goals (v1)

1. **One-click capture:** A user can archive any page they are currently viewing with a single click in the browser toolbar, with status feedback visible in the extension popup within 5 seconds of click.

2. **Zero JavaScript in output:** Every saved archive must contain zero executable script tags, zero inline event handlers, and zero tracking pixel `<img>` tags. Verified by automated test over a corpus of 50 known-tracking-heavy pages.

3. **Asset completeness:** Saved archives must render visually correctly (correct layout, images, fonts) for at least 90% of tested pages without any external network requests.

4. **Gemini Nano classification:** Every captured page must produce at least one classification record in the tracker database, even if the page has no detectable tracking code (classification: "none detected"). Classification must run locally with no external API calls.

5. **Go server stability:** The server must handle concurrent capture requests without data loss or corruption. Target: 10 simultaneous captures with no errors.

6. **SQLite tracker database:** All classified snippets must be stored in the structured SQLite schema with correct tactic_type, domain, severity, and anonymized context fields.

7. **Public GitHub export:** The public `spacedudem/amber-tracker-db` repository must be automatically updated within 5 minutes of new snippets being stored, with valid exports in uBlock, hosts, Disconnect, and Pi-hole formats.

8. **HTTPS end-to-end:** All communication between extension and server must be over HTTPS with a valid certificate. No plaintext transmission of captured content.

9. **No PII in public data:** Automated validation must confirm that no usernames, email addresses, session tokens, or referring URLs appear in any snippet submitted to the public database.

10. **Offline archive permanence:** A saved archive opened 1 year later with no internet connection must render correctly and make zero outbound network requests.

---

## Success Metrics

**Adoption (3 months post-launch)**
- 50+ distinct users have installed the extension and completed at least one successful capture
- 500+ archives created across all users
- 1,000+ unique tracking snippets in the public database

**Quality**
- Capture success rate: 95%+ of attempted captures complete without error (measured over any 30-day usage period)
- JavaScript elimination rate: 100% of output archives contain zero script tags and zero inline event handlers (hard requirement, automated regression test)
- Classification coverage: Gemini Nano successfully classifies snippets in 90%+ of captures (failures allowed only due to context window overflow, with graceful fallback to unclassified record)
- Archive render fidelity: 90%+ of captured pages render without broken layout when opened offline

**Community Value**
- Export validity: 100% of generated filter exports (uBlock, hosts, Pi-hole, Disconnect) pass format validation with zero syntax errors
- At least one external citation, fork, or integration of the tracker database within 90 days of going live
- At least one external filter list maintainer acknowledges the database as a reference

**Technical Health**
- Server uptime: 99.5%+ over any 30-day period
- No SQLite corruption events
- Capture latency P95 under 15 seconds for pages under 5MB total assets

---

## Non-Goals (v1)

- **Firefox support:** amber is Chrome-only in v1. Firefox does not support the Chrome Prompt API (window.ai / Gemini Nano). Adding a fallback classification path would double scope.
- **Bulk/crawl archiving:** amber captures one page at a time. Crawling linked pages is out of scope. kage handles that use case.
- **Full session recording:** Recording entire browsing sessions as timestamped folder archives is a v2 feature. v1 captures individual pages on demand only.
- **OSINT evidence packages:** Cryptographically sealed, legally-suitable evidence packages with chain-of-custody documentation are a v2 feature.
- **Chrome Web Store distribution:** Distributing amber via the Web Store requires an Origin Trial token from Google for the Prompt API. v1 targets personal use with the `#optimization-guide-on-device-model` Chrome flag. Web Store distribution is a v2 consideration.
- **Mobile browser support:** Chrome on Android does not support Gemini Nano via the Prompt API in the same form. Mobile is out of scope for v1.
- **Video and audio archiving:** Embedded media (YouTube players, audio streams) is not captured. The archive will contain poster images but not playable media files.
- **Login automation or session sharing:** amber uses the existing browser session. It does not automate login, manage credentials, or share sessions between users.
- **Real-time collaboration or sharing UI:** There is no social or sharing layer in v1. Archives are private by default; tracker database contributions are public, but individual archives are not exposed through any sharing mechanism.
- **User account system:** The server has no user management. It assumes a single trusted user operating it locally or on a private server. Multi-user support is a future concern.

---

## Strategic Bets

### Bet 1: Gemini Nano and the Chrome Prompt API Will Stabilize

The Chrome Prompt API (window.ai.languageModel) is behind an Origin Trial, which means Google has not yet committed it to a stable API surface. amber bets that Google will continue to invest in on-device AI in Chrome and that the API will stabilize toward a permanent feature. Evidence supporting this bet: Google has publicly committed to on-device AI as a Chrome differentiator, the API has been available in Origin Trial since Chrome 127, and the underlying Gemini Nano model ships with Chrome on most desktop platforms. Risk: if Google pivots or the API surface changes significantly, amber's classification pipeline must be rewritten. Mitigation: the classification layer is designed as a pluggable module. If Gemini Nano becomes unavailable, a regex-based fallback classifier can replace it without changing the rest of the pipeline. The public tracker database degrades gracefully to unclassified snippets rather than failing entirely.

### Bet 2: Chrome Market Share Keeps amber Relevant

amber is Chrome-only. Chrome's desktop market share is consistently above 65% across measured platforms. The vast majority of privacy-conscious power users who run extensions are on Chrome or Chromium-based browsers (Edge, Brave, Arc). This bet assumes that Chrome's dominance is durable enough over the v1 lifecycle that single-browser targeting does not significantly limit amber's user base. Brave users — a meaningful share of the privacy-conscious cohort — can use amber because Brave is Chromium-based and supports the same extension API, though Gemini Nano availability on Brave is not guaranteed without additional configuration.

### Bet 3: The Community Will Adopt and Contribute to the Tracker Tactic Database

amber's long-term value compounds as the tracker database grows. The bet is that privacy researchers, filter list maintainers, and security analysts will find the implementation-level tactic classification useful enough that they adopt the database as a reference and contribute improvements. Evidence: EasyList and uBlock Origin filter lists are maintained by small communities of volunteers who clearly find this work worthwhile; there is no comparable resource at the implementation-tactic level, which suggests unmet demand. The exports deliberately target existing toolchain formats (uBlock, hosts, Disconnect, Pi-hole) to minimize adoption friction. Risk: if the schema or export format is not compatible with existing toolchains, adoption will be slow. Mitigation is the multi-format export design.

### Bet 4: Inline and First-Party Tracking Will Continue to Grow

The entire premise of the tracker database's value depends on inline and first-party tracking growing in importance relative to third-party tracking. This bet is well-supported by recent history: following Chrome's deprecation of third-party cookies and the widespread adoption of ad blockers, publishers and advertisers have systematically migrated toward first-party data collection, server-side tag managers, and inline fingerprinting. The trajectory is clear and accelerating. amber's classification capability becomes more valuable the more this migration continues.

---

## Future Vision (v2+)

### Feature: Full Session Recording

In v2, amber will optionally record entire browsing sessions as structured, timestamped archives. When session recording is enabled, every page visited is captured automatically (without requiring a manual click), saved to a local folder hierarchy organized by date and domain, and indexed for search. The local archive becomes a private, searchable map of the user's internet activity — a personal internet history that does not depend on browser history sync, cannot be subpoenaed from a cloud provider, and is stored entirely on the user's own hardware.

Session recording will be opt-in and local-only by default. No session data will be submitted to the tracker database. The user can export selected sessions in specific formats or review them through a local web UI served by the Go server. Sessions are organized as timestamped folders: each folder contains the clean static archive of the page visited, a metadata JSON record (URL, capture time, page title), and a thumbnail screenshot. This creates a faithful, browsable record of a research session or investigative thread that survives link rot, page changes, and account deletions.

### Feature: OSINT Evidence Package

For journalists, legal professionals, and investigators, v2 will introduce a dedicated capture mode that produces a cryptographically sealed evidence package. When capturing in evidence mode, amber will:

1. Record the full page archive (clean HTML + assets) as in standard mode
2. Capture a timestamped screenshot of the live page
3. Record the full HTTP request and response log for the page load (using Chrome's debugger API), showing exactly which requests were made and what responded
4. Generate a SHA-256 hash of the complete package contents
5. Produce a signed manifest containing: capture timestamp synced against an NTP reference, package hash, browser user agent, extension version, and operator-configurable identity field
6. Package everything as a single ZIP archive with the signed manifest as the root document

The resulting artifact provides the technical foundation for digital evidence submission: the hash proves the content has not been altered since capture, the signed manifest binds the hash to a moment in time, and the request log demonstrates what the page did when it loaded. This is specifically designed for journalists documenting illegal or false content before takedown, litigants preserving evidence of defamation or fraud, OSINT investigators building timestamped dossiers, and compliance teams documenting regulatory violations by third parties.

### Infrastructure: Distributed Tracker Database Federation

As the tracker database grows, v2 will introduce federation: other amber users running their own instances will be able to submit anonymized snippet classifications to a shared network. A simple conflict-resolution protocol will merge submissions from multiple sources, with severity scores weighted by submission count and source reputation. The public GitHub repository will continue to serve as the canonical export target, but the underlying data will be aggregated from the community rather than from a single server. This transforms amber from a single-user contribution tool into community infrastructure for the privacy ecosystem.

### API: Programmatic Access for Third-Party Tools

A documented REST API on the Go server will allow third-party tools — browser extensions, research scripts, CI pipelines, security scanners — to query the tracker database, submit new snippets for classification, and retrieve filter list exports programmatically. This opens amber's classification capability as a service: a tool or researcher can submit a JavaScript snippet and receive a tactic classification without running amber's full capture pipeline. Over time, this positions the tracker database as a reference service for the broader privacy tooling ecosystem.

---

*Document version: 1.0. Prepared for internal use. Last updated: June 2026.*
