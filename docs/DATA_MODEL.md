# amber — Data Model & ERD

## Overview

amber stores data across three distinct layers, each with a different owner, retention policy, and privacy posture.

**Layer 1 — SQLite (server-side, private):** The Go server maintains a local SQLite database at `/opt/amber/amber.db` (configurable via the `AMBER_DB_PATH` environment variable; in Docker Compose this is bind-mounted from the host). This is the authoritative operational store. It holds every archive record, every stripped snippet with its Gemini Nano classification, and every fetched asset's metadata. This data never leaves the server unless the user explicitly triggers a public submission. It contains non-anonymized internal fields (full URLs, file paths, archive IDs) that exist purely to let the server manage its own state.

**Layer 2 — Disk archive (server-side, private):** Static HTML files and fetched assets live under `/var/www/html/sharex/uploads/kage/`. Each archived page writes one directory tree. This is what the user actually reads offline. The database row for an archive points to a `file_path` in this tree; the tree carries a `_amber-meta.json` sidecar with enough metadata to reconstruct context without querying the database.

**Layer 3 — Public GitHub repository (anonymized, permanent):** A subset of snippet data — stripped of all user-identifying fields — is periodically exported to `spacedudem/amber-tracker-db`. This is the community resource. Once a snippet is pushed to the public repository it is considered immutable; it can be appended to but never modified or deleted (the git history is permanent). The public schema is strictly additive: it gains new tactic entries over time but existing entries are never edited.

The separation is deliberate. The user's browsing context, the original full URLs of pages they visited, and the file paths where archives live are private by design and never appear in Layer 3.

---

## SQLite Schema (Server-Side, Private)

All tables live in a single SQLite file. Foreign keys are enforced (`PRAGMA foreign_keys = ON` is set at connection open). WAL mode is enabled for concurrent reads during export jobs.

### Table: archives

The root entity. One row per captured page.

```sql
CREATE TABLE archives (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  domain           TEXT    NOT NULL,
  url              TEXT    NOT NULL,
  url_hash         TEXT    NOT NULL UNIQUE,   -- sha256(url), used for dedup
  captured_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  file_path        TEXT    NOT NULL,          -- absolute path to index.html on disk
  asset_count      INTEGER NOT NULL DEFAULT 0,
  snippet_count    INTEGER NOT NULL DEFAULT 0,
  page_title       TEXT,
  status           TEXT    NOT NULL DEFAULT 'complete'
                   CHECK (status IN ('complete', 'partial', 'failed'))
);

CREATE INDEX idx_archives_domain      ON archives(domain);
CREATE INDEX idx_archives_captured_at ON archives(captured_at);
CREATE INDEX idx_archives_status      ON archives(status);
```

**Field notes:**

- `url_hash` is a hex-encoded SHA-256 of the normalized URL (scheme + host + path, query string stripped). It enforces uniqueness at the database level and is used as a fast deduplication key when the extension submits a URL the server has already archived.
- `file_path` stores the absolute path to the root `index.html` of the archive. For multi-page captures this points to the directory root; the convention is that the server always writes an `index.html` at the top level. Asset paths are stored in the `assets` table relative to this root.
- `status` transitions: an archive begins as `partial` when the POST /api/v1/archive request arrives; it moves to `complete` after all assets are written and the asset/snippet counts are finalized; it moves to `failed` if a critical write error occurs after the row is inserted. This means a row always exists even for failed captures, which is important for the deduplication check.
- `asset_count` and `snippet_count` are denormalized counters updated transactionally with the child rows. They are redundant with `COUNT(*)` queries on child tables but exist to make the archive listing endpoint (`GET /api/v1/archives`) fast without joins.

### Table: snippets

One row per stripped element. A single page capture can produce dozens to hundreds of snippet rows.

```sql
CREATE TABLE snippets (
  id                    INTEGER PRIMARY KEY AUTOINCREMENT,
  archive_id            INTEGER NOT NULL REFERENCES archives(id) ON DELETE CASCADE,
  tactic_type           TEXT    NOT NULL REFERENCES tactic_types(id),
  raw_snippet           TEXT    NOT NULL,
  context               TEXT,          -- up to 500 chars of surrounding HTML
  element_type          TEXT    NOT NULL
                        CHECK (element_type IN ('script', 'attr', 'pixel', 'iframe', 'style', 'other')),
  attribute_name        TEXT,          -- populated when element_type = 'attr'
  severity              TEXT    NOT NULL DEFAULT 'medium'
                        CHECK (severity IN ('low', 'medium', 'high', 'critical')),
  classifier_confidence REAL
                        CHECK (classifier_confidence IS NULL
                               OR (classifier_confidence >= 0.0 AND classifier_confidence <= 1.0)),
  submitted_to_public   INTEGER NOT NULL DEFAULT 0
                        CHECK (submitted_to_public IN (0, 1)),
  submitted_at          DATETIME,
  FOREIGN KEY (archive_id)  REFERENCES archives(id) ON DELETE CASCADE,
  FOREIGN KEY (tactic_type) REFERENCES tactic_types(id)
);

CREATE INDEX idx_snippets_tactic_type    ON snippets(tactic_type);
CREATE INDEX idx_snippets_archive_id     ON snippets(archive_id);
CREATE INDEX idx_snippets_severity       ON snippets(severity);
CREATE INDEX idx_snippets_not_submitted  ON snippets(submitted_to_public)
  WHERE submitted_to_public = 0;
```

**Field notes:**

- `raw_snippet` holds the verbatim code as stripped from the DOM. For `script` elements this is the full text content of the `<script>` tag (or the `src` URL if it was an external script that was not fetched). For `attr` elements this is the attribute value (e.g., the full `onclick="..."` handler string). The Gemini Nano context window cap of ~4096 tokens means that minified JS blocks longer than approximately 3000 characters are split into overlapping 2048-token chunks before classification; each chunk produces a separate snippet row with the same `archive_id` and a `context` field indicating the chunk index.
- `context` stores up to 500 characters of surrounding HTML (200 chars before, 300 chars after the stripped element). This helps human reviewers understand where the tracker appeared in page structure without storing the full page HTML redundantly.
- `element_type` is an enum over the five categories the extension recognizes: `script` (inline or external JS), `attr` (inline event handler like `onclick`, `onload`, `data-tracking-*`), `pixel` (1x1 img with tracking URL), `iframe` (third-party iframe ad or widget), `style` (CSS-based tracking or exfiltration), and `other` for edge cases.
- `attribute_name` is populated when `element_type = 'attr'`. It records which HTML attribute was stripped (e.g., `onclick`, `onmouseover`, `data-ga-label`). This enables aggregation queries like "which attributes are most commonly used for inline tracking?"
- `classifier_confidence` is the probability score returned by Gemini Nano (0.0 to 1.0). A NULL value means the snippet was classified by regex heuristics in the extension's content.js fallback path rather than by the AI model (this happens when Gemini Nano is unavailable or the chunk exceeds the context window).
- `submitted_to_public` and `submitted_at` track whether this snippet has been pushed to the public GitHub repository. The partial index `idx_snippets_not_submitted` makes the nightly export job fast — it only scans unsubmitted rows.
- `ON DELETE CASCADE` means deleting an archive row also purges its child snippets and assets. The user can delete an archive from the UI without leaving orphaned rows.

### Table: tactic_types (reference / seed table)

A controlled vocabulary of tracking tactics. This table is populated at server startup from the seed file `tracker-db/tactic_types.json` and is not modified at runtime.

```sql
CREATE TABLE tactic_types (
  id               TEXT PRIMARY KEY,    -- snake_case, e.g. 'canvas_fingerprint'
  display_name     TEXT NOT NULL,       -- 'Canvas Fingerprinting'
  description      TEXT NOT NULL,
  severity_default TEXT NOT NULL
                   CHECK (severity_default IN ('low', 'medium', 'high', 'critical')),
  owasp_category   TEXT,               -- e.g. 'A05:2021'
  cwe_id           TEXT                -- e.g. 'CWE-359'
);
```

**Seed values (non-exhaustive):**

| id | display_name | severity_default | cwe_id |
|----|-------------|-----------------|--------|
| `canvas_fingerprint` | Canvas Fingerprinting | high | CWE-359 |
| `webgl_fingerprint` | WebGL Fingerprinting | high | CWE-359 |
| `audio_fingerprint` | AudioContext Fingerprinting | high | CWE-359 |
| `font_fingerprint` | Font Enumeration Fingerprinting | medium | CWE-359 |
| `beacon` | Navigator.sendBeacon Exfiltration | high | CWE-201 |
| `pixel` | Tracking Pixel | medium | CWE-201 |
| `session_replay` | Session Replay (keystroke/mouse) | critical | CWE-312 |
| `cname_cloak` | CNAME-Cloaked Tracker | high | CWE-441 |
| `evercookie` | Evercookie / Respawning Storage | critical | CWE-539 |
| `localstorage_id` | LocalStorage Persistent Identifier | medium | CWE-539 |
| `third_party_iframe` | Third-Party iframe Ad/Widget | low | CWE-1021 |
| `inline_handler` | Inline Event Handler (generic) | low | — |
| `data_attr_tracking` | Data Attribute Tracking | low | — |
| `fetch_exfil` | Fetch/XHR Exfiltration | high | CWE-201 |
| `timing_attack` | High-Resolution Timing Attack | medium | CWE-208 |

The `owasp_category` and `cwe_id` fields allow the public dataset to be cross-referenced with existing security taxonomies. They are informational — amber does not enforce or validate against OWASP/CWE schemas at runtime.

### Table: assets

One row per fetched asset for an archive. CSS, fonts, and images fetched through the user's session are recorded here.

```sql
CREATE TABLE assets (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  archive_id    INTEGER NOT NULL REFERENCES archives(id) ON DELETE CASCADE,
  original_url  TEXT    NOT NULL,
  local_path    TEXT    NOT NULL,   -- relative to archive root, e.g. '_assets/a1b2c3.png'
  content_hash  TEXT    NOT NULL,   -- sha256 of file bytes, hex-encoded
  asset_type    TEXT    NOT NULL
                CHECK (asset_type IN ('image', 'stylesheet', 'font', 'other')),
  size_bytes    INTEGER NOT NULL,
  fetch_status  TEXT    NOT NULL
                CHECK (fetch_status IN ('success', 'failed', 'skipped')),
  FOREIGN KEY (archive_id) REFERENCES archives(id) ON DELETE CASCADE
);

CREATE INDEX idx_assets_archive_id    ON assets(archive_id);
CREATE INDEX idx_assets_content_hash  ON assets(content_hash);
CREATE UNIQUE INDEX idx_assets_archive_url ON assets(archive_id, original_url);
```

**Field notes:**

- `local_path` is relative to the archive root directory. The server resolves it to an absolute path by joining the parent archive's `file_path` directory with this value.
- `content_hash` enables deduplication across archives. If two different pages on the same domain reference the same CDN-hosted font, the second archive can copy the file from the first archive's `_assets/` directory instead of refetching it.
- `fetch_status = 'skipped'` occurs when the extension marks an asset URL as a known tracker domain (e.g., `doubleclick.net` resources). These are recorded in the database so the archive metadata accurately reflects what was omitted, but no file is written to disk for them.
- The unique index on `(archive_id, original_url)` prevents duplicate asset rows if the extension sends the same URL twice in a single archive payload (which can happen when a page references the same CSS from multiple `<link>` tags).

---

## ASCII ERD Diagram

```
 +--------------------------------------------------+
 |                  tactic_types                    |
 +--------------------------------------------------+
 | PK id TEXT                                       |
 |    display_name TEXT                             |
 |    description TEXT                              |
 |    severity_default TEXT                         |
 |    owasp_category TEXT                           |
 |    cwe_id TEXT                                   |
 +-------------------------+------------------------+
                           | 1
                           | is referenced by
                           | N
 +-------------------------+------------------------+
 |                    snippets                      |
 +--------------------------------------------------+
 | PK id INTEGER                                    |
 | FK archive_id INTEGER ---------------------------+---+
 | FK tactic_type TEXT                              |   |
 |    raw_snippet TEXT                              |   |
 |    context TEXT                                  |   |
 |    element_type TEXT                             |   |
 |    attribute_name TEXT                           |   |
 |    severity TEXT                                 |   |
 |    classifier_confidence REAL                    |   |
 |    submitted_to_public INTEGER                   |   |
 |    submitted_at DATETIME                         |   |
 +--------------------------------------------------+   |
                                                        | N
 +------------------------------------------------------+--+
 |                    archives                             |
 +--------------------------------------------------+------+
 | PK id INTEGER                                    |
 |    domain TEXT                                   |
 |    url TEXT                                      |
 |    url_hash TEXT UNIQUE                          |
 |    captured_at DATETIME                          |
 |    file_path TEXT                                |
 |    asset_count INTEGER                           |
 |    snippet_count INTEGER                         |
 |    page_title TEXT                               |
 |    status TEXT                                   |
 +--------------------------------------------------+
                           | 1
                           | has
                           | N
 +-------------------------+------------------------+
 |                     assets                       |
 +--------------------------------------------------+
 | PK id INTEGER                                    |
 | FK archive_id INTEGER                            |
 |    original_url TEXT                             |
 |    local_path TEXT                               |
 |    content_hash TEXT                             |
 |    asset_type TEXT                               |
 |    size_bytes INTEGER                            |
 |    fetch_status TEXT                             |
 +--------------------------------------------------+

Cardinality:
  archives    1 ---------- N  snippets
  archives    1 ---------- N  assets
  tactic_types 1 --------- N  snippets
```

---

## Public Database Schema (GitHub, Anonymized)

The public repository at `spacedudem/amber-tracker-db` contains only data that has been explicitly anonymized. The nightly export job (a Go binary invoked by a systemd timer) selects all snippets where `submitted_to_public = 0`, applies the anonymization transforms described in the Data Privacy Model section, writes the output files, and then updates `submitted_to_public = 1` and `submitted_at = NOW()` for those rows in a single transaction.

### tracker_tactics.json

The canonical public data file. Regenerated in full on every export run (not appended to) so consumers can treat it as a complete snapshot.

```json
{
  "schema_version": "1.0.0",
  "generated_at": "2026-06-16T02:00:00Z",
  "total_entries": 12543,
  "tactics": [
    {
      "id": "01939f4a-6c2d-7e8b-9d0e-1f2a3b4c5d6e",
      "domain_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "tactic_type": "canvas_fingerprint",
      "raw_snippet": "var c=document.createElement('canvas');var ctx=c.getContext('2d');ctx.fillText('Cwm fjordbank',2,15);fp=c.toDataURL();",
      "context": "<script type=\"text/javascript\">// tracking init",
      "element_type": "script",
      "attribute_name": null,
      "severity": "high",
      "classifier_confidence": 0.94,
      "submitted_at": "2026-06-16T02:00:00Z"
    }
  ]
}
```

**Anonymization transforms applied before export:**

- `id`: A newly generated UUIDv4 per snippet, not the internal SQLite `id`. This prevents cross-referencing between the public dataset and any future leaked internal database.
- `domain_hash`: SHA-256 hex of the lowercase domain (e.g., `sha256("tracking.example.com")`). Not reversible for unknown domains, but known domains can be looked up. This is intentional — the goal is to allow researchers to ask "does domain X appear in the dataset?" without exposing the full URL or page path.
- `raw_snippet`: Exported verbatim. The content.js stripping pass has already removed user data (session tokens, auth cookies in attribute values, email addresses in form field defaults). The server performs a secondary scan before export, redacting any string matching email, UUID, or numeric-ID patterns longer than 8 digits using a conservative regex.
- Fields omitted entirely from public export: `archive_id`, `file_path`, `url`, `original_url`, and `context` fragments that contain user-identifying URL segments.

### Export formats

The `exports/` directory in the public repository contains four derived files regenerated by GitHub Actions on every push to `main`. The Actions workflow (`regenerate-exports.yml`) runs a Go binary from `tracker-db/cmd/export/` that reads `tracker_tactics.json` and produces all four formats.

**ublock_filters.txt**

uBlock Origin extended static filter syntax. The file contains one filter rule per unique domain where a high or critical severity snippet was seen, drawn from the known-domains seed in `tracker-db/known_domains.json`. Inline-code filters use the `##` cosmetic syntax to target elements by attribute pattern.

```
! amber tracker database -- uBlock Origin filters
! Generated: 2026-06-16T02:00:00Z
! Entries: 847
!
||analytics.example.com^$third-party
||pixel.tracker.net^$image,third-party
##[data-ga-label]
##[onclick*="beacon"]
@@||cdn.trusted.com^$script
```

**hosts.txt**

Standard hosts file format mapping tracking domains to the null address. Only domains resolvable from the known-domains seed appear here. Comments at the top of the file document the source and generation timestamp.

```
# amber tracker database -- hosts format
# Generated: 2026-06-16T02:00:00Z
# Entries: 612
#
0.0.0.0 analytics.example.com
0.0.0.0 pixel.tracker.net
0.0.0.0 beacon.adtech.io
```

**disconnect.json**

Disconnect.me JSON format, compatible with the Firefox Disconnect extension and similar consumers. The categories map to amber's severity levels: `Advertising` for low/medium pixel/iframe tactics, `Analytics` for fingerprinting/beacon tactics, `Social` for social widget iframes, and `Fingerprinting` for high/critical device fingerprinting tactics.

```json
{
  "version": 1,
  "updated": "2026-06-16T02:00:00Z",
  "categories": {
    "Fingerprinting": {
      "amber-canvas-fp": {
        "canvas_fingerprint": ["example.com", "tracker.net"]
      }
    },
    "Analytics": {
      "amber-beacon": {
        "beacon": ["beacon.adtech.io"]
      }
    },
    "Advertising": {
      "amber-pixels": {
        "pixel": ["pixel.tracker.net"]
      }
    }
  }
}
```

**pihole.txt**

The simplest format. One domain per line, no comments, no headers. Pi-hole reads this directly as a blocklist URL. Same domain set as `hosts.txt`.

```
analytics.example.com
pixel.tracker.net
beacon.adtech.io
```

---

## Archive File Structure (Disk)

Archives are written to `/var/www/html/sharex/uploads/kage/` by the Go server after it receives a POST /api/v1/archive request. The directory structure mirrors the URL path of the archived page with the domain as the top-level directory. All asset filenames are the first 8 hex characters of their SHA-256 content hash plus the original file extension, which makes them content-addressable and avoids filename collisions between archives on the same domain.

```
/var/www/html/sharex/uploads/kage/
+-- paulgraham.com/
|   +-- _amber-meta.json
|   +-- _assets/
|   |   +-- a1b2c3d4.png
|   |   +-- e5f6g7h8.css
|   |   +-- f9e8d7c6.woff2
|   +-- index.html
|   +-- essays/
|       +-- index.html
|       +-- programming.html
+-- www.axios.com/
    +-- _amber-meta.json
    +-- _assets/
    |   +-- 3a4b5c6d.png
    |   +-- 7e8f9a0b.css
    +-- 2026/
        +-- 06/
            +-- 11/
                +-- google-trade-worker-initiative-ai.html
```

All HTML files in the archive have the following properties enforced by the server-side postprocessing step before writing to disk:

1. No `<script>` tags of any kind (inline or `src=`).
2. No `<link rel="preconnect">`, `<link rel="dns-prefetch">`, or `<link rel="preload">` tags (these trigger network requests on reopen).
3. No inline event handler attributes (`on*`).
4. All `<img>`, `<source>`, `<link rel="stylesheet">`, and `@font-face src` URLs rewritten to point to `_assets/` relative paths.
5. A `<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'none';">` tag injected into `<head>` to enforce script-free behavior at the browser level even if a script tag were somehow missed.
6. No `data-*` attributes that amber's content.js flagged as tracking attributes (e.g., `data-ga-label`, `data-fbp`, `data-track-click`).

### _amber-meta.json structure

```json
{
  "amber_version": "1.0.0",
  "captured_at": "2026-06-16T14:32:11Z",
  "original_url": "https://paulgraham.com/",
  "page_title": "Paul Graham",
  "asset_count": 47,
  "snippet_count": 12,
  "tactics_found": ["canvas_fingerprint", "beacon", "pixel"],
  "archive_id": 42,
  "url_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "status": "complete"
}
```

This file is the only metadata that travels with the archive on disk. If the SQLite database is lost or rebuilt, the server's recovery tool (`tracker-db/cmd/recover/`) can walk the archive tree, read every `_amber-meta.json`, and reconstruct the `archives` table rows. The `url_hash` and `archive_id` fields are the join keys between the sidecar file and the database.

---

## Data Privacy Model

amber's threat model distinguishes three principals: the user, the tracked domain, and potential observers of the public dataset.

**What is stored locally (private, SQLite + disk):**

- Full original URLs of archived pages including query strings (stored in `archives.url`).
- Full asset URLs including any session-specific parameters (stored in `assets.original_url`). Note: the content of fetched assets may contain session-specific data if it is baked into a CSS or image file; the server does not parse asset contents for embedded PII.
- Raw tracking snippets exactly as they appeared in the page.
- Page titles (`archives.page_title`).
- Capture timestamps.

None of this is transmitted anywhere unless the user explicitly enables the public submission feature. The default configuration has `AMBER_PUBLIC_SUBMIT=false`.

**What is anonymized before public submission:**

- Domain: hashed with SHA-256. The original domain string is never written to the public repository.
- Archive ID and URL: omitted entirely. There is no way to determine which page a snippet came from using only the public dataset.
- Snippet content: run through a secondary redaction pass that strips email addresses, UUID-like strings, and long numeric identifiers. This handles the case where a tracking pixel URL was embedded with a user-specific ID (e.g., `https://pixel.example.com/?uid=1234567890`).
- Timestamps: `submitted_at` is the export batch timestamp, not the original capture timestamp. This prevents timing correlation attacks that might link a submitted snippet to a specific user session.

**What is never stored, anywhere:**

- Cookies or session tokens from the user's browser. The extension uses `credentials: 'include'` to fetch assets but never reads or stores the cookie values themselves — they are used transparently by the fetch call and are not accessible from the extension context due to the browser's httpOnly and SameSite constraints.
- Browser history beyond the currently captured page. The extension does not have the `history` permission and cannot enumerate previously visited URLs.
- User account information or login state from captured pages. HTML is stripped before sending to the server; form field values, logged-in usernames rendered in the DOM, and similar PII are removed by content.js before the payload is assembled.
- IP addresses of extension clients. The Go server is behind nginx which is configured to not log or forward the client IP to the application layer.

**What the server operator can infer:**

The server operator has access to the full SQLite database and the full archive tree. amber is designed for personal or small-team self-hosted use, not multi-tenant SaaS. The privacy model assumes the server operator is the same person as the user. If amber is adapted for multi-user deployment, the `archives` table must be extended with a `user_id` column and row-level access control must be added to all API endpoints.

---

## Data Retention

**Archives (disk):** Kept indefinitely until the user explicitly deletes them through the extension UI or the server admin API (`DELETE /api/archives/:id`). Deletion removes both the database row (cascading to snippets and assets rows) and the directory tree on disk. There is no automatic expiry or rolling purge. Disk usage is bounded by the user's own capture behavior.

**SQLite snippets (local):** Kept indefinitely. The local snippet database is the primary research asset of the installation. Snippets are not automatically pruned even after they are submitted to the public repository. The `submitted_to_public` flag tracks which ones have been exported; local rows remain for local aggregation, review, and auditing.

**Public submissions (GitHub):** Permanent and immutable once pushed. The amber project makes no commitment to delete public submissions. Researchers relying on the dataset for reproducible research need to trust that entries will not disappear; this is enforced by the GitHub repository's branch protection rules: force-push to `main` is disabled, and the full `tracker_tactics.json` history is preserved in git. If a snippet is found to contain user-identifying information after submission (a bug in the redaction pass), the remediation is to add the entry's UUID to a `tracker-db/redacted.json` blocklist file and regenerate exports; the raw entry remains in git history but is excluded from all downstream export files and the current `tracker_tactics.json`.

**Export files (GitHub):** Regenerated on every push to `main` of the tracker-db repository. Consumers should treat them as ephemeral snapshots with up to 24-hour staleness.

---

## Migration Strategy

SQLite schema migrations are managed by sequential numbered SQL files in `tracker-db/migrations/`. The server applies pending migrations at startup using a minimal migration runner embedded in the Go server at `server/internal/db/migrate.go`.

The migration tracking table is created on first run if it does not exist:

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
  version    INTEGER  PRIMARY KEY,
  applied_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  filename   TEXT     NOT NULL
);
```

Migration files follow the naming convention `NNNN_description.sql` where `NNNN` is a zero-padded four-digit integer:

```
tracker-db/migrations/
  0001_initial_schema.sql
  0002_add_assets_table.sql
  0003_add_classifier_confidence.sql
  0004_add_attribute_name_to_snippets.sql
  0005_add_archives_status_index.sql
```

At startup, the runner:

1. Creates `schema_migrations` if it does not exist.
2. Reads the highest `version` from `schema_migrations` (0 if the table is empty).
3. Scans the migrations directory for files with version number greater than current.
4. Applies them in ascending version order, wrapping each file in its own transaction. If any SQL statement in a migration file fails, the transaction is rolled back and the server exits with a non-zero status rather than starting in a partially-migrated state.
5. Inserts a row into `schema_migrations` for each successfully applied file.

Migrations are forward-only. There is no `down` migration path. If a migration needs to be reversed, a new numbered migration that undoes the previous change is written. This matches SQLite's limited `ALTER TABLE` support (SQLite cannot drop columns or change column types without table recreation) and avoids maintaining bidirectional migration logic.

Destructive migrations (those requiring full table recreation, such as dropping or renaming a column) use the standard SQLite approach within a single migration file:

1. `CREATE TABLE new_table AS SELECT ...` — create replacement with new schema.
2. `INSERT INTO new_table SELECT ...` — copy data with any necessary transforms.
3. `DROP TABLE old_table`.
4. `ALTER TABLE new_table RENAME TO old_table`.
5. Recreate indexes.

All of these steps execute within the migration's single transaction, so the operation is atomic: either all steps succeed and the migration is recorded, or none of them are committed.

The `tracker-db/migrations/` directory is embedded into the server binary at compile time using Go's `//go:embed` directive, so the server binary is fully self-contained and does not require the source tree to be present at the deployment path.
