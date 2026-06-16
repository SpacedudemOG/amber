# amber — API Specification

**Version:** 1.0.0
**Last updated:** 2026-06-16
**Server:** Go 1.22+, mattn/go-sqlite3
**Base URL:** `https://korh.one/amber/api/v1`

---

## Overview

This document is the authoritative specification for the amber Go server API. It covers every endpoint, the request and response schemas, error handling behavior, the Chrome extension internal message protocol, and the Gemini Nano classifier prompt contract. All prose here is normative — the Go server implementation must conform to what is described.

The amber server receives page captures from the Chrome extension, stores static HTML archives on disk, logs classified tracking snippets in SQLite, and (optionally) pushes anonymized findings to the public GitHub tracker database. All communication between extension and server is JSON over HTTPS.

amber is a single-user self-hosted tool. There is no multi-tenancy, no user registration flow, and no OAuth. Authentication is a single pre-shared API key configured at server startup. The server runs behind nginx on the `korh.one` host; all TLS termination happens at the nginx layer.

---

## Authentication

Every request to the server (except `GET /health`) must include the following header:

```
X-Amber-Key: <pre-shared-secret>
```

The server reads the expected value from the `AMBER_API_KEY` environment variable at startup. If the variable is unset or shorter than 32 characters, the server exits with a fatal error rather than starting in an unauthenticated state. The recommended generation command is `openssl rand -hex 32`.

Requests that omit the header or supply an incorrect value receive `401 Unauthorized` with no additional information. The server does not disclose whether the key exists, is wrong, or is missing — the response body is always:

```json
{ "error": "invalid api key" }
```

Key comparison uses `crypto/subtle.ConstantTimeCompare` to prevent timing-based key inference.

The key is set in the extension's background.js at build time via a build-time substitution step. It is not stored in the extension source tree.

---

## Global Response Conventions

All request bodies must be `Content-Type: application/json; charset=utf-8`. All response bodies are `Content-Type: application/json; charset=utf-8`. Non-JSON content types in requests receive `415 Unsupported Media Type`.

All timestamps in request and response bodies are formatted as RFC 3339 strings in UTC, for example `2026-06-16T14:32:11Z`. The server parses incoming timestamps using `time.Parse(time.RFC3339, ...)` and rejects malformed values with `400 Bad Request`. The server always emits timestamps in UTC with the `Z` suffix.

All entity IDs are JSON integers (`int64`). Do not assume ordering or gaplessness — autoincrement gaps can result from transaction rollbacks during failed archive writes.

---

## Endpoints

### POST /api/v1/archive

Receive a fully cleaned page capture from the extension. The extension must call this endpoint before `POST /api/v1/snippets` because the returned `archive_id` is required for the snippets payload. This is the primary write endpoint.

The extension calls it after: (1) content.js has serialized and stripped the rendered DOM, (2) background.js has fetched all assets through the user's live session, and (3) background.js has run Gemini Nano classification on all snippets or exhausted retries and fallen back to heuristic classification.

**Request**

```
Content-Type: application/json
X-Amber-Key: <key>
```

Maximum payload size: **50 MB**. The server enforces this via `http.MaxBytesReader` before JSON parsing begins. Payloads exceeding the limit receive `413 Payload Too Large` before any bytes are written to disk.

**Request body:**

```json
{
  "url": "https://example.com/article/how-to-cook-pasta",
  "captured_at": "2026-06-16T14:23:00Z",
  "amber_version": "1.0.0",
  "page_title": "How to Cook Pasta — Example.com",
  "html": "<html lang=\"en\"><head>...</head><body>...</body></html>",
  "assets": [
    {
      "original_url": "https://example.com/img/pasta.jpg",
      "content_type": "image/jpeg",
      "data": "base64-encoded-bytes",
      "size_bytes": 45231
    },
    {
      "original_url": "https://example.com/css/main.css",
      "content_type": "text/css",
      "data": "base64-encoded-bytes",
      "size_bytes": 8912
    }
  ],
  "snippet_count": 12
}
```

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `url` | string | yes | The canonical URL of the captured page. Must be a valid absolute URL with `https://` scheme. Query string is preserved in this field but stripped when computing `url_hash` for deduplication. Max 2048 characters. |
| `captured_at` | string (RFC 3339) | yes | Timestamp of when the user clicked capture in the extension popup. Set by the extension, not the server. Must be within 24 hours of the server's current clock to reject obviously stale or replayed requests. |
| `amber_version` | string | yes | Semantic version string of the extension that produced this payload, e.g. `1.0.0`. Stored but not enforced — future versions may reject very old clients if a breaking protocol change is made. |
| `page_title` | string | no | `document.title` at time of capture. Max 512 characters, truncated silently if longer. Empty string and null are both accepted; the server coerces both to NULL in the database. |
| `html` | string | yes | The fully cleaned HTML document as a UTF-8 string. All script tags, inline event handlers, tracking pixels, and fingerprinting code must already be stripped before submission. The server runs a secondary safety pass before writing to disk. Max 20MB as a string. |
| `assets` | array | yes | List of page assets fetched by the extension through the user's live browser session. May be empty (`[]`) but must be present. Max 200 items per request. |
| `assets[].original_url` | string | yes | The absolute URL from which the asset was fetched. Used as the deduplication key within an archive. Max 2048 characters. |
| `assets[].content_type` | string | yes | MIME type as returned by the browser fetch, e.g. `image/jpeg`, `text/css`, `font/woff2`. The server maps this to the `asset_type` field in the database: `image/*` → `image`, `text/css` → `stylesheet`, `font/*` → `font`, everything else → `other`. |
| `assets[].data` | string | yes | Standard base64-encoded bytes of the asset (RFC 4648). No line breaks. The server rejects malformed base64 with `400 Bad Request`. |
| `assets[].size_bytes` | integer | yes | Byte length of the decoded asset. The server validates this matches the actual decoded length. Mismatches cause `400 Bad Request`. |
| `snippet_count` | integer | yes | Number of stripped snippets the extension intends to send in a follow-up `POST /api/v1/snippets` call. May be 0 if no classifiable snippets were found. |

**Payload limits:**

- Maximum total request body: 50 MB
- Maximum `html` field (as JSON string): 20 MB
- Maximum assets per request: 200
- Maximum individual asset size (decoded): 10 MB

**Deduplication behavior:**

Before writing anything to disk or inserting any database rows, the server computes `sha256(normalize(url))` where normalization strips the query string and fragment and lowercases the host. It queries `archives` for a row with a matching `url_hash`. If a match exists and its `status` is `complete`, the server returns `200 OK` (not `201 Created`) with the existing archive's response fields and `"duplicate": true`. If the existing row's `status` is `partial` or `failed`, the server deletes the old row and its associated disk files and proceeds with a fresh write.

**Server processing steps:**

1. Validate authentication header.
2. Read body with `http.MaxBytesReader(50MB)`.
3. Decode JSON and validate all required fields.
4. Compute `url_hash`; check for duplicate.
5. Insert `archives` row with `status = 'partial'`.
6. Run server-side HTML postprocessing: strip residual `<script>` tags, remove `on*` attribute handlers, rewrite asset `src`/`href` to `_assets/` relative paths, inject `<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'none';">`.
7. Compute disk directory path from URL components.
8. Write `index.html` to disk.
9. For each asset: decode base64, verify size, compute `sha256` content hash, write to `_assets/<first8ofhash>.<ext>`, insert `assets` row. Failed assets are recorded with `fetch_status = 'failed'` rather than aborting the entire request.
10. Write `_amber-meta.json` sidecar.
11. Update `archives` row: `status = 'complete'`, `asset_count` = number successfully saved.
12. Return `201 Created`.

Steps 5 through 11 execute inside a single database transaction.

**Response 201 Created:**

```json
{
  "archive_id": 42,
  "archive_url": "https://korh.one/amber/archives/example.com/article/how-to-cook-pasta/",
  "file_path": "/var/www/html/sharex/uploads/kage/example.com/article/how-to-cook-pasta/index.html",
  "assets_saved": 22,
  "assets_failed": 0,
  "duplicate": false
}
```

**Response 200 OK (duplicate URL):**

Same shape as 201 but `"duplicate": true` and counts reflect the original archive.

**Response fields:**

| Field | Type | Description |
|---|---|---|
| `archive_id` | integer | Stable database ID for this archive. Pass to `POST /api/v1/snippets`. |
| `archive_url` | string | Public HTTPS URL where the archived page is accessible via the nginx static file server. |
| `file_path` | string | Absolute server-side path to the saved `index.html`. Included for debugging; the extension does not need it for normal operation. |
| `assets_saved` | integer | Number of assets successfully written to disk. |
| `assets_failed` | integer | Number of assets skipped due to decode errors or size violations. Non-zero does not cause the overall request to fail. |
| `duplicate` | boolean | `true` when the URL hash matched an existing complete archive. |

**Error responses:**

| Status | Example body | Condition |
|--------|-------------|-----------|
| `400 Bad Request` | `{"error": "missing required field: url"}` | Required field absent or null. Message names the first missing field in validation order. |
| `400 Bad Request` | `{"error": "invalid url: must be absolute https URL"}` | URL fails parse or scheme check. |
| `400 Bad Request` | `{"error": "invalid captured_at: must be RFC3339 timestamp"}` | Timestamp parse failure. |
| `400 Bad Request` | `{"error": "captured_at is too old: must be within 24 hours of server time"}` | Replay guard rejected the timestamp. |
| `400 Bad Request` | `{"error": "invalid base64 in asset[2].data"}` | Zero-indexed position where base64 decode failed. |
| `400 Bad Request` | `{"error": "asset[2].size_bytes mismatch: declared 45231, decoded 45229"}` | Size verification failure. |
| `400 Bad Request` | `{"error": "too many assets: max 200"}` | Asset array exceeds limit. |
| `401 Unauthorized` | `{"error": "invalid api key"}` | Missing or wrong `X-Amber-Key`. |
| `413 Payload Too Large` | `{"error": "payload exceeds 50MB limit"}` | Body exceeded limit before JSON parsing. |
| `500 Internal Server Error` | `{"error": "internal server error"}` | Database error, disk write error, or unhandled panic. Details written to server's structured log. |

---

### POST /api/v1/snippets

Receive classified tracking snippets for an already-created archive. The extension calls this immediately after `POST /api/v1/archive` returns an `archive_id`. Each call is additive — multiple calls with the same `archive_id` append rows rather than replacing them. This allows the extension to stream snippet batches as Gemini Nano finishes classifying them.

**Request**

```
Content-Type: application/json
X-Amber-Key: <key>
```

Maximum payload size: **5 MB**.

**Request body:**

```json
{
  "archive_id": 42,
  "snippets": [
    {
      "tactic_type": "canvas_fingerprint",
      "raw_snippet": "var canvas = document.createElement('canvas'); var ctx = canvas.getContext('2d'); ctx.fillText('Cwm fjordbank glyphs vext quiz', 2, 15);",
      "context": "<script>!function(){var canvas=document.createElement('canvas');var ctx=canvas.getContext('2d');ctx.fillText('Cwm fjordbank glyphs vext quiz',2,15);var hash=canvas.toDataURL();navigator.sendBeacon('/track',hash)}()</script>",
      "element_type": "script",
      "attribute_name": null,
      "severity": "high",
      "classifier_confidence": 0.94
    },
    {
      "tactic_type": "inline_handler",
      "raw_snippet": "dataLayer.push({'event':'pageview','user_id':'u_9182'})",
      "context": "onclick=\"dataLayer.push({'event':'pageview','user_id':'u_9182'})\"",
      "element_type": "attr",
      "attribute_name": "onclick",
      "severity": "medium",
      "classifier_confidence": 0.81
    },
    {
      "tactic_type": "pixel",
      "raw_snippet": "https://pixel.example.com/px?v=1&t=pageview",
      "context": "<img src=\"https://pixel.example.com/px?v=1&t=pageview\" width=\"1\" height=\"1\" style=\"display:none\">",
      "element_type": "pixel",
      "attribute_name": null,
      "severity": "medium",
      "classifier_confidence": null
    }
  ],
  "submit_to_public": true
}
```

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `archive_id` | integer | yes | Must reference an existing `archives.id`. Returns `404 Not Found` if the ID does not exist. |
| `snippets` | array | yes | List of classified snippets. Must not be empty. Max 500 items per request. |
| `snippets[].tactic_type` | string | yes | Must be one of the `tactic_types.id` values seeded at server startup. The server rejects unknown tactic types with `400 Bad Request` rather than silently inserting a broken foreign key. |
| `snippets[].raw_snippet` | string | yes | The minimal code fragment that demonstrates the tactic. UTF-8. Max 16,384 characters. Snippets longer than this were chunked by the extension before classification; each chunk is a separate array entry. |
| `snippets[].context` | string | no | Up to 500 characters of surrounding HTML — 200 before and 300 after the stripped element. Truncated (not errored) if longer. |
| `snippets[].element_type` | string | yes | Enum: `script`, `attr`, `pixel`, `iframe`, `style`, `other`. Must match one of these exact values. |
| `snippets[].attribute_name` | string or null | no | Required when `element_type` is `attr`; must be null otherwise. The server enforces this consistency and returns `400 Bad Request` if violated. |
| `snippets[].severity` | string | yes | One of: `low`, `medium`, `high`, `critical`. This is the Gemini Nano model's severity assessment, or the extension's heuristic assessment when `classifier_confidence` is null. |
| `snippets[].classifier_confidence` | float or null | no | Gemini Nano confidence score, 0.0–1.0. Null when classified by heuristics rather than Gemini Nano. When non-null, the server rejects values outside [0.0, 1.0]. |
| `submit_to_public` | boolean | yes | When `true`, snippets that pass the public eligibility check (see below) are queued for the nightly anonymized export to the public GitHub repository. When `false`, all snippets are stored locally only. |

**Public submission eligibility:**

A snippet is eligible for the next nightly public export only when all of the following are true:
- `submit_to_public` is `true`.
- `classifier_confidence` is not null (heuristic snippets are excluded — they lack the AI confidence signal).
- `classifier_confidence >= 0.70` (configurable via `AMBER_MIN_CONFIDENCE` env var, default 0.70).
- `raw_snippet` passes the server-side secondary redaction scan (no email addresses, UUID-like strings, or numeric identifiers longer than 8 digits detected).

The export job applies full anonymization (domain hashing, URL stripping) before the GitHub push. See the Data Model document for the complete anonymization pipeline.

**Server processing steps:**

1. Validate authentication and content-type.
2. Verify `archive_id` exists in SQLite.
3. Validate every snippet object. Validation is all-or-nothing: if any snippet fails validation, the entire request is rejected and zero rows are inserted.
4. Insert all valid snippet rows in a single database transaction.
5. For snippets that meet public eligibility, set `submitted_to_public = 0` (queued, not yet pushed) on the inserted rows.
6. Update `archives.snippet_count` to the current `COUNT(*)` for that `archive_id` within the same transaction.
7. Return `201 Created`.

**Response 201 Created:**

```json
{
  "snippets_stored": 8,
  "snippets_queued_for_public": 6,
  "skipped": 2
}
```

| Field | Type | Description |
|---|---|---|
| `snippets_stored` | integer | Total snippets inserted into the local SQLite `snippets` table. |
| `snippets_queued_for_public` | integer | Snippets that passed eligibility and will be included in the next nightly public export. |
| `skipped` | integer | Snippets not inserted because a row with the same `raw_snippet` hash and `archive_id` already exists (retry deduplication), or the `raw_snippet` matched the internal PII blocklist. Skipped snippets do not cause the request to fail. |

**Error responses:**

| Status | Example body | Condition |
|--------|-------------|-----------|
| `400 Bad Request` | `{"error": "missing required field: snippets"}` | `snippets` key absent or null. |
| `400 Bad Request` | `{"error": "snippets array must not be empty"}` | Zero-length array. |
| `400 Bad Request` | `{"error": "snippets[2].tactic_type \"click_heatmap\" is not a known tactic type"}` | Unknown tactic type; zero-indexed. |
| `400 Bad Request` | `{"error": "snippets[1].attribute_name is required when element_type is attr"}` | Consistency violation. |
| `400 Bad Request` | `{"error": "snippets[7].classifier_confidence 1.23 is out of range [0.0, 1.0]"}` | Out-of-bounds confidence value. |
| `401 Unauthorized` | `{"error": "invalid api key"}` | Bad or missing key. |
| `404 Not Found` | `{"error": "archive not found"}` | `archive_id` does not exist. |
| `500 Internal Server Error` | `{"error": "internal server error"}` | Transaction failure. |

---

### GET /api/v1/archives

Returns a paginated list of saved archives, optionally filtered by domain. Results are sorted by `captured_at` descending by default (most recent first).

**Request**

```
X-Amber-Key: <key>
```

No request body.

**Query parameters:**

| Parameter | Type | Default | Constraints | Description |
|---|---|---|---|---|
| `domain` | string | none | max 253 chars | Filter to archives where `archives.domain` exactly matches this value. Case-insensitive. Example: `domain=example.com`. Does not match subdomains unless specified. |
| `limit` | integer | 50 | 1–200 | Maximum results to return. Values outside the range are clamped, not rejected. |
| `offset` | integer | 0 | >= 0 | Rows to skip. Use with `limit` for pagination. |
| `sort` | string | `captured_at_desc` | `captured_at_desc` or `captured_at_asc` | Sort order. Only `captured_at` is sortable in v1. |

**Example request:**
```
GET /amber/api/v1/archives?domain=example.com&limit=10&offset=0&sort=captured_at_desc
X-Amber-Key: <key>
```

**Response 200 OK:**

```json
{
  "total": 142,
  "limit": 10,
  "offset": 0,
  "archives": [
    {
      "id": 42,
      "domain": "example.com",
      "url": "https://example.com/article/how-to-cook-pasta",
      "page_title": "How to Cook Pasta — Example.com",
      "captured_at": "2026-06-16T14:23:00Z",
      "archive_url": "https://korh.one/amber/archives/example.com/article/how-to-cook-pasta/",
      "asset_count": 22,
      "snippet_count": 8,
      "status": "complete"
    },
    {
      "id": 41,
      "domain": "paulgraham.com",
      "url": "https://paulgraham.com/",
      "page_title": "Paul Graham",
      "captured_at": "2026-06-16T10:15:00Z",
      "archive_url": "https://korh.one/amber/archives/paulgraham.com/",
      "asset_count": 47,
      "snippet_count": 12,
      "status": "complete"
    }
  ]
}
```

`total` is the count of all matching rows before pagination (a `COUNT(*)` with the same `WHERE` clause but no `LIMIT`/`OFFSET`). The extension uses this to render "Showing 1–10 of 142 archives."

Archives with `status = 'failed'` are included in listing results and should display a visual indicator in the extension UI. Archives with `status = 'partial'` represent in-progress captures and may have `asset_count` and `snippet_count` of 0 while the write is still in flight.

**Error responses:**

| Status | Body | Condition |
|--------|------|-----------|
| `400 Bad Request` | `{"error": "invalid offset: must be non-negative integer"}` | Non-integer or negative `offset`. |
| `400 Bad Request` | `{"error": "invalid sort value: must be captured_at_desc or captured_at_asc"}` | Unrecognized sort parameter. |
| `401 Unauthorized` | `{"error": "invalid api key"}` | Bad or missing key. |

---

### GET /api/v1/archives/:id

Returns a single archive by its integer ID, including a tactic breakdown of its associated snippets.

**Request**

```
X-Amber-Key: <key>
```

No request body. `:id` is a positive integer in the URL path.

**Example request:**
```
GET /amber/api/v1/archives/42
X-Amber-Key: <key>
```

**Response 200 OK:**

```json
{
  "id": 42,
  "domain": "example.com",
  "url": "https://example.com/article/how-to-cook-pasta",
  "page_title": "How to Cook Pasta — Example.com",
  "captured_at": "2026-06-16T14:23:00Z",
  "amber_version": "1.0.0",
  "archive_url": "https://korh.one/amber/archives/example.com/article/how-to-cook-pasta/",
  "file_path": "/var/www/html/sharex/uploads/kage/example.com/article/how-to-cook-pasta/index.html",
  "asset_count": 22,
  "snippet_count": 8,
  "status": "complete",
  "tactic_summary": [
    {
      "tactic_type": "canvas_fingerprint",
      "display_name": "Canvas Fingerprinting",
      "count": 1,
      "severity": "high",
      "cwe_id": "CWE-359"
    },
    {
      "tactic_type": "beacon",
      "display_name": "Navigator.sendBeacon Exfiltration",
      "count": 3,
      "severity": "high",
      "cwe_id": "CWE-201"
    },
    {
      "tactic_type": "inline_handler",
      "display_name": "Inline Event Handler (generic)",
      "count": 2,
      "severity": "low",
      "cwe_id": null
    },
    {
      "tactic_type": "pixel",
      "display_name": "Tracking Pixel",
      "count": 2,
      "severity": "medium",
      "cwe_id": "CWE-201"
    }
  ],
  "severity_summary": {
    "low": 2,
    "medium": 2,
    "high": 4,
    "critical": 0
  }
}
```

The `tactic_summary` array contains one entry per distinct `tactic_type` found in the archive's snippets, computed by a `GROUP BY tactic_type` query joining `snippets` and `tactic_types`. Tactic types with zero occurrences are omitted. `file_path` is included in the single-archive response but omitted from the list endpoint — it is useful for locating the file directly but is noise in list context.

**Error responses:**

| Status | Body | Condition |
|--------|------|-----------|
| `400 Bad Request` | `{"error": "invalid archive id: must be a positive integer"}` | `:id` is not a valid positive integer. |
| `401 Unauthorized` | `{"error": "invalid api key"}` | Bad or missing key. |
| `404 Not Found` | `{"error": "archive not found"}` | No row with the given ID. |

---

### DELETE /api/v1/archives/:id

Permanently deletes an archive and all its associated snippets and assets. This action is irreversible.

**Request**

```
X-Amber-Key: <key>
```

No request body.

**Processing steps:**

1. Verify the archive ID exists; return `404` if not.
2. Begin a database transaction.
3. Delete the `archives` row. `ON DELETE CASCADE` removes all child `snippets` and `assets` rows automatically.
4. Commit the transaction.
5. Delete the archive directory tree from disk via `os.RemoveAll`.
6. Return `204 No Content`.

Steps 4 and 5 are not atomic. If the database transaction commits but the disk removal fails, the server logs an error to stderr with the orphaned path and still returns `204`. The database is authoritative; orphaned disk files can be cleaned up manually and pose no harm while served by nginx.

Snippets already submitted to the public GitHub repository are not affected by deletion. Once pushed, they are considered permanently public (see Data Model: Data Retention).

**Response 204 No Content:** Empty body.

**Error responses:**

| Status | Body | Condition |
|--------|------|-----------|
| `401 Unauthorized` | `{"error": "invalid api key"}` | Bad or missing key. |
| `404 Not Found` | `{"error": "archive not found"}` | No row with the given ID. |
| `500 Internal Server Error` | `{"error": "internal server error"}` | Database transaction failure. |

---

### GET /health

Health check endpoint. Does not require authentication. Called by nginx's upstream health probe on a 30-second interval and by the extension on startup to verify server reachability.

**Response 200 OK:**

```json
{
  "status": "ok",
  "version": "1.0.0",
  "db": "ok",
  "disk_free_gb": 45.2,
  "uptime_seconds": 3721
}
```

| Field | Type | Description |
|---|---|---|
| `status` | string | `"ok"` when all subsystems are healthy. `"error"` when SQLite is unreachable. |
| `version` | string | Server binary version set via `-ldflags` at build time. |
| `db` | string | `"ok"` if `SELECT 1` succeeds against the SQLite connection pool. `"error"` if not. |
| `disk_free_gb` | float | Available space in gigabytes on the filesystem containing the archive root. Reported to one decimal place. The extension shows a warning when this falls below 2.0. |
| `uptime_seconds` | integer | Seconds the Go server process has been running, derived from a package-level start time recorded at startup. |

If SQLite is unavailable, the server returns `200 OK` with `"status": "error"` and `"db": "error"` rather than a 5xx. This allows nginx's probe (which only inspects HTTP status) to pass while the extension (which inspects the JSON body) shows a degraded-state indicator. If the server itself fails to start — for example due to a missing `AMBER_API_KEY` or migration failure — no health endpoint is served at all.

**Degraded response example:**

```json
{
  "status": "error",
  "version": "1.0.0",
  "db": "error",
  "disk_free_gb": 45.2,
  "uptime_seconds": 3721
}
```

---

## Extension ↔ Server Message Protocol

The Chrome extension has three components that communicate via `chrome.runtime.sendMessage` / `chrome.runtime.onMessage`. These are not HTTP requests — they are internal Chrome extension IPC messages. They never leave the browser. This section documents the message protocol as the canonical contract between the three extension components.

### Message: CAPTURE_COMPLETE

**Sender:** `content.js`
**Receiver:** `background.js`

Sent when the content script has finished serializing and stripping the current tab's DOM. At this point snippets are not yet classified — Gemini Nano classification happens in background.js after receiving this message.

```json
{
  "type": "CAPTURE_COMPLETE",
  "payload": {
    "url": "https://example.com/article/how-to-cook-pasta",
    "capturedAt": "2026-06-16T14:23:00Z",
    "pageTitle": "How to Cook Pasta — Example.com",
    "html": "<html lang=\"en\">...</html>",
    "assetUrls": [
      "https://example.com/img/pasta.jpg",
      "https://example.com/css/main.css"
    ],
    "strippedSnippets": [
      {
        "rawSnippet": "var canvas = document.createElement('canvas')...",
        "context": "<script>!function(){var canvas=...</script>",
        "elementType": "script",
        "attributeName": null
      },
      {
        "rawSnippet": "dataLayer.push({'event':'pageview'})",
        "context": "onclick=\"dataLayer.push({'event':'pageview'})\"",
        "elementType": "attr",
        "attributeName": "onclick"
      }
    ]
  }
}
```

**Field notes:**

- `html` is the already-stripped document. background.js must not re-strip it.
- `assetUrls` are absolute URLs. background.js fetches each using `fetch(url, { credentials: 'include' })` within the extension service worker, inheriting the tab's cookies and session without reading the cookie values.
- `strippedSnippets` are pre-identified by content.js using regex heuristics. They have no `tacticType` or `severity` field yet. background.js feeds them to Gemini Nano for authoritative classification before submitting to the server.
- `elementType` is one of: `script`, `attr`, `pixel`, `iframe`, `style`, `other`.
- `attributeName` is the HTML attribute name for `elementType: "attr"` entries, e.g. `onclick`, `onmouseover`, `data-analytics`. Null for all other element types.

### Message: CAPTURE_STATUS

**Sender:** `background.js`
**Receiver:** `popup.js`

Sent at each major stage transition so the popup can update its progress UI. background.js uses `chrome.runtime.sendMessage` to broadcast to all extension pages; the popup uses `chrome.runtime.onMessage.addListener` to receive updates.

```json
{
  "type": "CAPTURE_STATUS",
  "payload": {
    "stage": "fetching_assets",
    "progress": 0.52,
    "message": "Fetching 12 of 23 assets...",
    "archiveUrl": null
  }
}
```

**Stage values and progress ranges:**

| Stage | Description | Progress range |
|---|---|---|
| `stripping` | content.js is serializing and stripping the DOM. Sent by content.js itself before `CAPTURE_COMPLETE`. | 0.00 – 0.10 |
| `fetching_assets` | background.js is fetching asset URLs through the user's browser session. | 0.10 – 0.50 |
| `classifying` | Gemini Nano is classifying stripped snippets. | 0.50 – 0.75 |
| `uploading` | background.js has sent `POST /api/v1/archive` and is awaiting the response. | 0.75 – 0.95 |
| `complete` | `POST /api/v1/snippets` returned `201`. `archiveUrl` is now populated. | 1.00 |
| `error` | An unrecoverable error occurred. `message` contains the user-facing description. | unchanged |

`progress` within `fetching_assets` increments as `0.10 + (assetsFetched / totalAssets) * 0.40` so that each fetched asset advances the bar proportionally. background.js sends at minimum one status message at the start of each stage transition and at minimum one update every 5 assets fetched during `fetching_assets`.

### Message: GET_STATUS

**Sender:** `popup.js`
**Receiver:** `background.js`

Sent when the popup is opened to request the current pipeline state. background.js responds synchronously via the `sendResponse` callback.

```json
{ "type": "GET_STATUS" }
```

**Response (synchronous via `sendResponse`):**

```json
{
  "currentStatus": {
    "stage": "classifying",
    "progress": 0.65,
    "message": "Classifying 8 of 12 snippets...",
    "archiveUrl": null
  }
}
```

Or if no capture is in progress:

```json
{ "currentStatus": null }
```

---

## Gemini Nano Classifier Prompt Protocol

Gemini Nano runs entirely within the user's Chrome browser via the `window.ai.languageModel` API (Chrome Prompt API, gated behind the `#optimization-guide-on-device-model` flag or an Origin Trial token). The background service worker calls it during the `classifying` stage, after assets are fetched but before the server POST. No snippet data ever leaves the user's machine during classification.

### Session Creation

background.js creates one Gemini Nano session per capture and destroys it after all snippets are classified.

```javascript
const session = await window.ai.languageModel.create({
  systemPrompt: SYSTEM_PROMPT,
  temperature: 0.1,
  topK: 3
});
// ... after all batches complete:
session.destroy();
```

Low `temperature` (0.1) and low `topK` (3) are used to maximize output determinism. Creative variation in classifier output is counterproductive.

### Token Budget

Gemini Nano has a context window of approximately 4096 tokens. background.js estimates token count as `Math.ceil(text.length / 4)` (a conservative approximation where 1 token is approximately 4 characters). Each batch is assembled so that the total estimated token count of all snippets in the batch remains under 3000 tokens, leaving headroom for the system prompt and the response. Snippets are never truncated within a batch — if a single snippet alone would exceed the budget, it is sent as a batch of one.

### System Prompt

```
You are a web tracking code classifier. You receive code snippets that were stripped from web pages during a privacy archive process. Your sole task is to classify each snippet by its tracking tactic type.

Tactic types you must choose from (use the exact snake_case identifiers):
- canvas_fingerprint: Code that reads canvas rendering output to generate a device fingerprint
- webgl_fingerprint: Code using WebGL renderer info or rendering output for fingerprinting
- font_fingerprint: Code enumerating installed fonts via canvas text measurement or CSS
- audio_fingerprint: Code using AudioContext.createOscillator or AnalyserNode for fingerprinting
- battery_api: Code reading navigator.getBattery() for tracking
- beacon: Code calling navigator.sendBeacon() to exfiltrate data
- pixel: Tracking pixel URLs (1x1 images, noscript pixels)
- localStorage_abuse: Code storing unique identifiers in localStorage across sessions
- sessionStorage_tracking: Code using sessionStorage for cross-tab tracking or user identification
- indexeddb_tracking: Code using IndexedDB for persistent identifier storage
- cookie_sync: Code synchronizing cookie identifiers across domains via URL parameters or pixels
- cname_cloak: Third-party tracker scripts loaded via CNAME subdomain of first-party domain
- service_worker_tracking: Service worker registration for persistent cross-session tracking
- inline_handler: Inline event handler (onclick, onload, etc.) sending analytics events
- data_attribute: HTML data attributes used to pass tracking identifiers to scripts
- css_tracking: CSS-based tracking (e.g., :visited link color exfiltration, image request on :hover)
- third_party_loader: Script that loads additional third-party tracking scripts
- behavioral: Session replay, mouse tracking, keystroke logging, or heatmap code
- network_timing: High-resolution timing measurements used for network-based fingerprinting
- fetch_exfil: Fetch or XMLHttpRequest calls exfiltrating user data to third-party endpoints
- unknown: The snippet appears tracking-related but does not match any above category

Severity levels:
- critical: Directly captures user content (keystrokes, form data) or enables cross-site identity linking at scale
- high: Persistent device fingerprinting or covert data exfiltration (canvas, WebGL, audio fingerprinting, fetch exfiltration)
- medium: Persistent but lower-fidelity tracking (beacons, pixels, localStorage IDs)
- low: Passive or low-impact (inline analytics event handlers, data attributes, third-party iframes)

Respond ONLY with valid JSON. No explanation, no markdown, no prose before or after the JSON.
```

### User Message Format

background.js sends one prompt call per batch. IDs in the request are local sequence numbers reset per capture, not archive IDs or SQLite IDs.

```
Classify these snippets:

[{"id":1,"snippet":"var c=document.createElement('canvas');var ctx=c.getContext('2d');ctx.fillText('Cwm fjordbank',2,15);navigator.sendBeacon('/t',c.toDataURL())","context":"<script>!function(){...}()</script>","elementType":"script"},{"id":2,"snippet":"dataLayer.push({'event':'pageview','user_id':'u_9182'})","context":"onclick=\"dataLayer.push(...)\"","elementType":"attr"}]

Respond with a JSON array, one entry per input id:
[{"id":<int>,"tactic_type":"<tactic_id>","severity":"low|medium|high|critical","confidence":<float 0.0-1.0>},...]
```

### Expected Response

```json
[
  {
    "id": 1,
    "tactic_type": "canvas_fingerprint",
    "severity": "high",
    "confidence": 0.96
  },
  {
    "id": 2,
    "tactic_type": "inline_handler",
    "severity": "medium",
    "confidence": 0.81
  }
]
```

### Response Parsing and Fallback

background.js attempts to parse the Gemini Nano response as JSON. If parsing fails (prose output or malformed JSON), background.js retries the same batch once with the original prompt prepended by: `"Your previous response was not valid JSON. Output ONLY the JSON array."`. If the retry also fails to parse, background.js falls back to heuristic regex-based classification for every snippet in the failing batch. Heuristically classified snippets have `classifier_confidence = null` in the database and are never eligible for public export.

---

## Error Handling

### Error Response Envelope

All error responses use the following shape:

```json
{ "error": "<human-readable message>" }
```

The `error` field is always a string. There is no numeric error code field — the HTTP status code is the machine-readable signal. Internal details (stack traces, SQL errors, absolute file paths) are written to the server's structured stderr log and are never included in response bodies.

### HTTP Status Code Reference

| Code | Name | When used in amber |
|---|---|---|
| 200 | OK | Successful GET; successful POST /archive for a duplicate URL |
| 201 | Created | Successful POST /archive (new archive) or POST /snippets |
| 204 | No Content | Successful DELETE /archives/:id |
| 400 | Bad Request | Missing field, type error, constraint violation, unknown enum value |
| 401 | Unauthorized | Missing or incorrect X-Amber-Key header |
| 404 | Not Found | Archive ID does not exist |
| 405 | Method Not Allowed | Wrong HTTP method for an endpoint |
| 413 | Payload Too Large | Request body exceeds the configured limit |
| 415 | Unsupported Media Type | Content-Type is not application/json on a POST endpoint |
| 500 | Internal Server Error | Disk write failure, SQLite error, or unexpected panic |

### Extension Retry Strategy

background.js uses the following policy for server call failures:

- **401 Unauthorized:** Do not retry. Display "API key rejected — check extension settings" in the popup. No further requests are made until the extension is reloaded.
- **400 Bad Request:** Do not retry. Log the response body to the service worker console. Display a generic capture failure message in the popup.
- **413 Payload Too Large:** Do not retry. Display "Page too large to archive (>50 MB). Try a smaller page."
- **500 Internal Server Error:** Retry up to 3 times with exponential backoff: 2 s, 4 s, 8 s. If all retries fail, display "Server error — archive may be incomplete. Check server logs."
- **Network error (no response):** Retry up to 3 times with exponential backoff: 5 s, 10 s, 20 s.
- **Timeout per attempt:** 60 seconds for `POST /api/v1/archive` (large payload), 15 seconds for all other endpoints.
- **Snippet retry across sessions:** If `POST /api/v1/archive` succeeds but `POST /api/v1/snippets` fails after all retries, the `archive_id` and unsent snippets are stored in `chrome.storage.local`. background.js re-attempts snippet submission on the next browser session startup. After 5 failed attempts across sessions, the queued data is discarded and a warning is logged to the service worker console.

---

## Rate Limiting

No rate limiting is implemented in v1. amber is a single-user personal tool; the pre-shared key provides sufficient access control. If amber is ever extended for shared or public use, rate limiting must be added at the nginx layer using `limit_req_zone` before modifying the Go server.

---

## Versioning Policy

The API uses URL path versioning. The current version prefix is `/api/v1`.

**Non-breaking changes** (permitted without incrementing the version):
- Adding new optional fields to request bodies. Old clients omit them and the server uses defaults.
- Adding new fields to response bodies. Old clients ignore unknown keys via `json.Unmarshal`'s default behavior.
- Adding new tactic types to the `tactic_types` seed table.
- Adding new valid values for `sort` or other enum query parameters.

**Breaking changes** (require a new `/api/v2` prefix):
- Removing or renaming existing request fields.
- Removing or renaming existing response fields.
- Changing the type or format of any existing field.
- Changing authentication from header-based to any other scheme.
- Changing the meaning or domain of any existing enum value.

The server may serve both `/api/v1` and `/api/v2` simultaneously during a transition window of at least 90 days. The extension pins its API base URL to a specific version at build time and does not perform runtime version negotiation or capability discovery.

---

## Server Configuration Reference

The Go server is configured entirely via environment variables. No configuration file is used. All variables are read at startup; the server does not reload configuration at runtime without a process restart.

| Variable | Required | Default | Description |
|---|---|---|---|
| `AMBER_API_KEY` | yes | — | Pre-shared API key. Minimum 32 characters. Server exits on startup if unset or too short. |
| `AMBER_DB_PATH` | no | `/var/lib/amber/amber.db` | Path to the SQLite database file. Created on first run if it does not exist. Parent directory must exist and be writable. |
| `AMBER_ARCHIVE_ROOT` | no | `/var/www/html/sharex/uploads/kage/` | Root directory for archive files. Must exist and be writable by the server process. |
| `AMBER_LISTEN_ADDR` | no | `127.0.0.1:8090` | Address and port for the HTTP server. Default binds to loopback only — nginx proxies from the public interface. Must never be set to `0.0.0.0` in production. |
| `AMBER_VERSION` | no | set at build time via `-ldflags` | Semver string reported in health check and archive metadata. |
| `AMBER_PUBLIC_SUBMIT` | no | `false` | Master switch for public snippet submission. When `false`, all snippets are stored locally regardless of the `submit_to_public` field in requests. |
| `AMBER_MIN_CONFIDENCE` | no | `0.70` | Minimum `classifier_confidence` required for a snippet to be queued for public export. Float in [0.0, 1.0]. |
| `AMBER_MAX_BODY_BYTES` | no | `52428800` (50 MB) | Maximum HTTP request body size in bytes for `POST /api/v1/archive`. Do not lower below 1 MB. |

---

## Security Notes

- The `AMBER_API_KEY` must never appear in extension source code, manifests, or any file committed to a public repository. It is injected as an environment variable on the server and stored in the browser extension's `chrome.storage.local`, set once via the extension's options page on first install.
- All server-to-disk paths are constructed by sanitizing the URL hostname and path. Path components are stripped of `..` sequences, null bytes, and any character outside `[a-zA-Z0-9._-]`. This prevents path traversal attacks.
- HTML saved to disk is served by nginx as static files. The nginx vhost for the archive directory sets `Content-Security-Policy: default-src 'none'` and `X-Content-Type-Options: nosniff` to prevent any attempt to execute stored content as scripts. The server also injects a `<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'none';">` tag into every archived HTML document as a belt-and-suspenders measure.
- Base64 assets are decoded server-side with a strict RFC 4648 decoder that rejects non-standard characters. Decoded bytes are written with `O_WRONLY|O_CREATE|O_EXCL` to prevent silently overwriting existing files.
- The public export pipeline never writes the full captured URL to the public dataset — only a SHA-256 hash of the domain. This is enforced at the server level regardless of what the extension submits. See the Data Model document for the complete anonymization pipeline.
- `X-Forwarded-For` is deliberately not forwarded to the Go server by nginx. The server does not log or process client IP addresses.
