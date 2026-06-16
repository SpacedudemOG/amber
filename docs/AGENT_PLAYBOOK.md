# amber — Agent Coordination Playbook

**Version:** 1.0  
**Last updated:** 2026-06-16  
**Maintained by:** Orchestrator Agent (update after each milestone)

---

## Purpose

This playbook governs how AI agents (Claude Code instances, subagents, workflow agents) coordinate to develop, test, and maintain the amber project. It defines roles, handoff protocols, proof-of-completion criteria, and repository hygiene standards.

Every agent that touches this project must read this document before taking any action. Deviation from this playbook is grounds for an Orchestrator to discard the agent's output and restart the task.

---

## Repository Strategy

### Branching Model

| Branch pattern | Purpose | Merge target |
|---|---|---|
| `main` | Always deployable. Protected. | — |
| `develop` | Integration branch. CI must pass. | `main` via PR |
| `feature/<milestone>/<component>` | Feature work per task | `develop` |
| `fix/<issue-number>` | Bug fixes | `develop` |
| `docs/<topic>` | Documentation-only updates | `develop` or `main` |

Branch naming examples:
- `feature/m1/content-script-strip`
- `feature/m2/asset-fetch`
- `feature/m3/gemini-nano-classify`
- `fix/42`
- `docs/api-snippet-schema`

**Rules:**
- Never commit directly to `main`.
- Never commit directly to `develop` (except automated merges from feature branches via PR).
- Branch names must be lowercase, hyphen-separated. No underscores.
- Delete branch after merge.

### Commit Convention (Conventional Commits)

```
<type>(<scope>): <description>

[optional body — wrap at 72 chars]

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

**Types:**

| Type | When to use |
|---|---|
| `feat` | New functionality visible to user or downstream system |
| `fix` | Bug correction |
| `test` | Adding or correcting tests (no production code change) |
| `docs` | Documentation only |
| `refactor` | Code restructure with no behavior change |
| `chore` | Dependency updates, config, tooling |
| `ci` | GitHub Actions, Dockerfile, deploy scripts |

**Scopes:**

| Scope | What it covers |
|---|---|
| `extension` | Chrome extension (content.js, background.js, popup.*) |
| `server` | Go server (handlers, models, db, main.go) |
| `tracker-db` | SQLite schema, migrations, export scripts |
| `docs` | /root/amber/docs/ |
| `ci` | .github/workflows/ |

**Examples:**
```
feat(extension): add canvas fingerprint detection to strip pipeline
fix(server): sanitize domain path to prevent directory traversal
test(extension): add unit tests for inline handler stripping
docs(api): update snippet endpoint request schema
refactor(server): extract archive writer into separate package
chore(ci): pin go version to 1.22.4 in workflow
```

**Rejection criteria:** A commit that uses a scope outside the list above, omits Co-Authored-By, or uses a vague description ("fix stuff", "update", "wip") will be rejected by the Reviewer Agent and must be amended before the PR is opened.

### PR Requirements

Every PR must include:

1. **Description** — what changed and why, referencing the plan task by number.
2. **Test evidence** — paste the full output of the final test run. Screenshots acceptable for extension UI changes.
3. **Security note** — explicitly state "no security-sensitive code changed" OR describe what changed and that Security Agent reviewed it.
4. **No open TODOs** — search the diff for `TODO`, `FIXME`, `HACK`, `XXX`, `placeholder`. Any hit is a blocker.
5. **CI passes** — Go tests and ESLint must be green before requesting review.

Self-hosted single-developer workflow: PRs are merged by the developer (no external review required). When an external contributor opens a PR, require 1 review from a project maintainer.

### Repository Sync

- `/root/kage/` — kage fork, kept entirely separate. Implementer agents must never modify files there unless the task explicitly involves kage integration.
- `/root/amber/docs/` — source of truth for all design decisions. If implementation diverges from a doc, the doc must be updated before the PR merges.
- `/root/amber/server/` — server source, deployed via Docker.
- `/root/amber/extension/` — extension source, loaded unpacked in Chrome dev mode.
- `/root/amber/tracker-db/` — SQLite schema and export tooling.

---

## Agent Roles

### Orchestrator Agent

**Responsibility:** Decompose milestones into tasks, dispatch subagents with full context, collect completion reports, maintain project state, decide task ordering and dependency resolution.

**Tools available:** All tools. Can read/write any file in `/root/amber/`, run bash, create PRs, query GitHub.

**When invoked:**
- At the start of every milestone.
- When a subagent returns a completion report (COMPLETE or BLOCKED).
- When CI fails on `develop`.
- When a Security Agent returns findings.

**Does NOT:**
- Write implementation code directly. All code comes from Implementer agents.
- Merge PRs without Reviewer approval.
- Proceed past a BLOCKED report without resolving the block.

**State tracking:** Orchestrator maintains task state in `/root/amber/docs/superpowers/plans/<plan-file>.md`. After each task completes, it marks the task `[x]` in the plan file and commits the update.

---

### Implementer Agent

**Responsibility:** Execute a single, fully-specified task. Write tests first, then implementation, then commit.

**Tools available:** Read, Write, Edit, Bash, Grep, Glob.

**Receives from Orchestrator:**
- Task title and number
- Plan file path and line range
- Full file paths of files to create or modify
- Any dependencies (other completed task outputs it needs)

**Returns to Orchestrator:**
```
TASK: [title]
STATUS: COMPLETE | BLOCKED
FILES_CHANGED: [list of absolute paths]
TEST_OUTPUT: [full output of final test run]
COMMIT: [full 40-char hash]
NOTES: [anything the Reviewer should know, or block reason]
```

**Mandatory sequence:**
1. Read the task from the plan file — confirm understanding before touching files.
2. Write failing tests first.
3. Run tests to confirm they fail for the right reason (not a compilation error).
4. Write minimal implementation to make tests pass.
5. Run tests to confirm they pass.
6. Run linter (`go vet ./...` for server, ESLint for extension).
7. Commit with conventional commit message.
8. Return completion report.

**Must NOT:**
- Skip writing tests first under any circumstances.
- Commit without passing tests.
- Create new files outside `/root/amber/` without explicit Orchestrator instruction.
- Leave `console.log` statements in extension production code.
- Use `eval()` anywhere in extension code.
- Hardcode secrets, API keys, tokens.

**If stuck for more than 15 minutes** on a single problem, return BLOCKED immediately. Do not keep retrying. Attach everything tried so far.

---

### Reviewer Agent

**Responsibility:** Verify that a completed implementation matches the spec, is secure, is well-tested, and follows project conventions.

**Tools available:** Read, Grep, Glob, Bash (read-only commands only — no writes, no git commits).

**Receives from Orchestrator:**
- Original task spec (verbatim from plan file)
- Implementer completion report
- Git diff of the changes

**Review checklist:**

1. **Spec compliance** — Does the implementation do exactly what the spec says? No more, no less.
2. **Test quality** — Are tests meaningful? Do they test behavior, not just that code runs? Are edge cases covered?
3. **Security** — SQL injection via unparameterized queries? Path traversal via unsanitized file paths? XSS via unescaped HTML? Secrets in code?
4. **TODO/placeholder scan** — `grep -rn "TODO\|FIXME\|HACK\|XXX\|placeholder"` on changed files.
5. **Commit message** — Follows conventional commits? Includes Co-Authored-By?
6. **Readability** — Consistent with existing code style? Variable names clear? No magic numbers without constants?
7. **No dead code** — No commented-out blocks left in.

**Returns ONE of:**
```
APPROVED: [brief rationale, 1-3 sentences]
```
or
```
CHANGES_REQUESTED:
- [file.go:42] [specific issue]
- [content.js:87] [specific issue]
```

Reviewer must be specific. "Clean it up" is not acceptable feedback. Every CHANGES_REQUESTED item must reference file and line, and state what the fix is.

---

### Security Agent

**Responsibility:** Perform a focused security review of any change touching authentication, file paths, SQL queries, network communication, or user data handling.

**Tools available:** Read, Grep, Glob.

**Automatically invoked by Orchestrator for:**
- Any change to `server/handlers/`
- Any change to `extension/background.js`
- Any change involving `database/sql`, file I/O, or HTTP client code
- Any change to authentication or API key handling

**STRIDE review matrix:**

| Threat | What to check in amber |
|---|---|
| Spoofing | API key validation on `/api/archive` and `/api/snippets` — constant-time comparison |
| Tampering | Archive files — can a malicious payload in HTML escape to filesystem? |
| Repudiation | Are capture events logged with enough context to audit? |
| Information Disclosure | Does `/api/archives` expose paths or metadata that leak user info? |
| Denial of Service | Is there a size limit on POST bodies? Can an attacker fill disk? |
| Elevation of Privilege | Can extension content script escape to background page with elevated perms? |

**Returns ONE of:**
```
CLEAR: [1-sentence rationale]
```
or
```
FINDINGS:
- [file:line] [STRIDE category] [severity: LOW|MEDIUM|HIGH|CRITICAL] [description]
  RECOMMENDATION: [specific fix]
```

Security agent findings block the task. Orchestrator dispatches Implementer to fix before Reviewer sees the code.

---

### Documentation Agent

**Responsibility:** Keep `/root/amber/docs/` in sync with actual implementation after each milestone completes.

**Tools available:** Read, Write, Edit.

**Invoked by Orchestrator:** After every milestone's integration test passes, before the develop→main PR is opened.

**Process:**
1. Read the milestone's list of completed tasks.
2. Compare each task's implementation against the corresponding doc section.
3. Update docs to reflect actual behavior (endpoints, schemas, configuration, flags).
4. Commit with `docs(<scope>): sync docs after milestone N completion`.
5. Return list of files updated.

**Never:** Invent API behavior that hasn't been implemented. Only document what exists.

---

## Task Lifecycle

```
Orchestrator reads plan task
  → Orchestrator dispatches Implementer with full context
      → Implementer writes failing tests
      → Implementer verifies tests fail (correct failure, not crash)
      → Implementer writes implementation
      → Implementer verifies tests pass
      → Implementer runs linter
      → Implementer commits
      → Implementer returns completion report
  → Orchestrator checks: is this security-sensitive?
      → YES: dispatches Security Agent with diff
          → Security Agent returns CLEAR or FINDINGS
          → If FINDINGS: dispatch Implementer with findings, loop
          → If CLEAR: proceed to Reviewer
      → NO: proceed directly to Reviewer
  → Orchestrator dispatches Reviewer with diff + spec + report
      → Reviewer returns APPROVED or CHANGES_REQUESTED
      → If CHANGES_REQUESTED: dispatch Implementer again with feedback
      → If APPROVED: Orchestrator marks task [x] in plan, dispatches next task
```

Parallel execution is allowed for tasks with no shared file dependencies. Orchestrator identifies these from the plan's dependency graph and dispatches both Implementers simultaneously. Parallel tasks must not write to the same file.

---

## Proof of Completion Criteria

### Per-Task Completion

A task is COMPLETE when ALL of the following are true:

- [ ] All files listed in the task "Files" section have been created or modified as specified.
- [ ] All tests listed in the task pass. Full test output is attached to the completion report.
- [ ] `grep -rn "TODO\|FIXME\|HACK\|XXX"` returns zero hits in changed files.
- [ ] `go vet ./...` passes (server tasks) or ESLint passes (extension tasks).
- [ ] Commit exists on feature branch with conventional commit message and Co-Authored-By.
- [ ] Reviewer has returned APPROVED.
- [ ] If security-sensitive: Security Agent has returned CLEAR.

### Per-Milestone Completion

A milestone is COMPLETE when ALL of the following are true:

- [ ] Every task in the milestone has individual task completion (all boxes above checked).
- [ ] Integration test from the milestone's "Definition of Done" section passes in full — not just unit tests.
- [ ] Documentation Agent has updated any docs that diverged from implementation.
- [ ] `develop` branch merged to `main` via PR with CI passing.
- [ ] If server changes included: Docker image rebuilt and container restarted on host.
- [ ] If extension changes included: extension reloaded in Chrome and smoke-tested (open a page, click capture, verify archive appears).

---

## Implementer Agent Prompt Template

Use this exact template when dispatching an Implementer. Fill in all bracketed fields. Do not omit any section.

```
You are implementing Task [N] of the amber project.

TASK: [task title]
PLAN FILE: /root/amber/docs/superpowers/plans/[plan-file].md
TASK LINES: [start-line]-[end-line]

CONTEXT:
- Project root: /root/amber/
- Go module: github.com/spacedudem/amber
- Extension: /root/amber/extension/ (Manifest V3, Chrome 127+, ES2022)
- Server: /root/amber/server/ (Go 1.22, net/http, mattn/go-sqlite3)
- Database: /root/amber/data/amber.db (SQLite3)

DEPENDENCIES COMPLETED:
[List any prior tasks whose output this task builds on, with their commit hashes]

REQUIREMENTS:
1. Read the task from the plan file (lines [start-line]-[end-line]) before touching any files.
2. Write failing tests FIRST. For Go: *_test.go files. For extension: test files in extension/test/.
3. Run tests to confirm they fail (go test ./... or npm test). Paste the failure output.
4. Write minimal implementation to make tests pass. Do not over-engineer.
5. Run tests to confirm they pass. Paste the passing output.
6. Run go vet ./... (server) or eslint extension/ (extension). Must be clean.
7. Commit: git add [specific files] && git commit -m "[conventional commit message]"
8. Return the completion report below.

COMPLETION REPORT FORMAT:
---
TASK: [title]
STATUS: COMPLETE
FILES_CHANGED:
  - /root/amber/[path/to/file1]
  - /root/amber/[path/to/file2]
TEST_OUTPUT:
  [full output of final passing test run]
COMMIT: [full 40-char hash]
NOTES: [anything the Reviewer should know]
---

RULES:
- Never skip writing tests first. If a component is untestable, return BLOCKED and explain why.
- Never commit without passing tests and a clean linter.
- Never modify files outside /root/amber/ without explicit Orchestrator instruction.
- Never use eval() in extension code.
- Never hardcode secrets. Use environment variables.
- If stuck for more than 15 minutes on a single problem, return BLOCKED with full context of what you tried.
- Branch must be: feature/[milestone]/[component] (already created by Orchestrator, or create it if specified).
```

---

## Reviewer Agent Prompt Template

```
You are reviewing a completed task for the amber project.

TASK SPEC (verbatim from plan):
[paste task text here]

IMPLEMENTER REPORT:
[paste completion report here]

GIT DIFF:
[paste output of: git diff main...HEAD or the PR diff]

Review the implementation. Check every item in this list:

1. SPEC COMPLIANCE: Does the code implement exactly what the spec says?
   - Not more (scope creep), not less (missing requirements).

2. TEST QUALITY:
   - Do tests assert meaningful behavior, not just that code runs?
   - Are error paths tested (invalid input, missing file, DB error)?
   - Are edge cases covered (empty input, max-size input, unicode)?

3. SECURITY:
   - SQL: all queries parameterized? No string concatenation into SQL?
   - File paths: filepath.Clean() used? Checked against base directory?
   - Extension: no eval(), no innerHTML with user-provided content?
   - No secrets or tokens in code?

4. OPEN ITEMS:
   - grep for TODO, FIXME, HACK, XXX, placeholder in changed files.
   - Any commented-out code blocks left in?

5. COMMIT MESSAGE:
   - Follows <type>(<scope>): <description> format?
   - Type and scope from approved lists?
   - Includes Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>?

6. CODE QUALITY:
   - Consistent with existing code style?
   - Variable and function names clear and descriptive?
   - No magic numbers (use named constants)?
   - Error returns checked (Go: no ignored errors)?

Return ONE of:

APPROVED: [1-3 sentence rationale]

or

CHANGES_REQUESTED:
- [path/to/file.go:42] [specific issue and required fix]
- [extension/content.js:87] [specific issue and required fix]

Every CHANGES_REQUESTED item must reference file:line and state the exact fix needed.
```

---

## GitHub Sync Protocol

### After Each Merged PR

1. Pull on host:
   ```bash
   git -C /root/amber pull origin main
   ```
2. If `server/` changed: rebuild and restart (see Server Deployment below).
3. If `extension/` changed: reload extension in Chrome (see Extension Reload below).
4. If `tracker-db/` schema changed: run migration:
   ```bash
   sqlite3 /root/amber/data/amber.db < /root/amber/tracker-db/migrations/latest.sql
   ```
5. Verify health endpoint responds:
   ```bash
   curl -sf http://127.0.0.1:8090/health && echo "OK"
   ```

### Server Deployment

```bash
cd /root/amber
docker build -t amber-server ./server/
docker stop amber-server 2>/dev/null || true
docker rm amber-server 2>/dev/null || true
docker run -d --name amber-server \
  --restart unless-stopped \
  -v /var/www/html/sharex/uploads/kage:/archives \
  -v /root/amber/data/amber.db:/data/amber.db \
  -e AMBER_API_KEY="${AMBER_API_KEY}" \
  -e GITHUB_TOKEN="${GITHUB_TOKEN}" \
  -p 127.0.0.1:8090:8090 \
  amber-server
```

Verify after deploy:
```bash
curl -sf http://127.0.0.1:8090/health
docker logs amber-server --tail 20
```

If the container exits immediately, check logs before attempting a retry.

### Extension Reload After Changes

1. Open `chrome://extensions` in Chrome.
2. Confirm Developer Mode is toggled on.
3. Find the amber extension card.
4. Click the circular reload icon.
5. Open the background service worker DevTools (click "service worker" link on the card).
6. Verify no errors in the console.
7. Open a test page, click the amber popup, confirm capture completes without error.

---

## Quality Gates

### Code Quality — Go (server/)

All of the following must pass before a task is considered complete:

```bash
go build ./...           # No build errors
go vet ./...             # Static analysis clean
go test ./... -count=1   # All tests pass, no caching
```

Additional checks:
- No `log.Fatal` in library code (only in `main()`).
- All exported functions have doc comments.
- No `panic()` in request handlers — return proper HTTP errors.
- No `os.Exit()` outside `main()`.

### Code Quality — Extension (extension/)

```bash
npx eslint extension/    # Must be clean (no warnings, no errors)
```

Additional checks:
- No `console.log` in production paths (content.js, background.js). `console.error` for genuine errors is acceptable.
- No `eval()` or `new Function()` anywhere.
- No `innerHTML` assignment with user-provided or page-provided content — use `textContent` or `DOMParser`.
- Manifest `permissions` array must not grow without explicit plan task authorizing it.

### Security (Automatic Checks — Applied by Implementer Before Commit)

These are not optional. Run them and confirm clean before committing:

**SQL queries:**
```bash
# Every sql.Exec and sql.Query call must use ? placeholders
grep -n 'Exec\|Query\|QueryRow' server/**/*.go | grep -v '?' | grep -v '_test.go'
# Expected: no output
```

**File path sanitization:**
```bash
# Any filepath operation must be preceded by filepath.Clean
grep -n 'os.Open\|os.Create\|os.WriteFile\|ioutil.WriteFile' server/**/*.go
# Review each hit manually to confirm filepath.Clean + base-dir check is applied
```

**Extension eval check:**
```bash
grep -rn 'eval\|new Function' extension/*.js
# Expected: no output
```

**Secret scan:**
```bash
grep -rn 'API_KEY\s*=\s*"[^"]\|TOKEN\s*=\s*"[^"]\|SECRET\s*=\s*"[^"' server/ extension/
# Expected: no output (secrets come from env vars, not literals)
```

---

## Escalation Protocol

### BLOCKED

Return immediately when:
- Stuck on the same problem for more than 15 minutes.
- A test fails in a way that cannot be explained by the implementation.
- A dependency task's output is missing or incorrect.
- The spec is contradicted by existing code.

BLOCKED report format:
```
TASK: [title]
STATUS: BLOCKED
BLOCKED_AT: [what step you were on]
TRIED:
  1. [thing tried]
  2. [thing tried]
CURRENT_ERROR: [full error output]
QUESTION: [specific question for Orchestrator]
```

### Security Finding

Security Agents must return findings immediately to the Orchestrator. Do not attempt to fix security issues without Orchestrator knowledge — the Orchestrator decides whether to fix-in-place, redesign, or escalate to the developer.

### Spec Ambiguity

If the spec has two valid interpretations, do not pick one and proceed. Return to Orchestrator with:
- The ambiguous text (verbatim)
- The two interpretations
- Which one you believe is correct and why

Orchestrator clarifies, updates the plan file, then re-dispatches.

---

## Milestone Handoff Checklist

When handing off between milestones, the Orchestrator runs this checklist:

```
MILESTONE [N] HANDOFF
=====================
[ ] All tasks marked [x] in plan file
[ ] `go test ./... -count=1` passes on develop
[ ] ESLint passes on develop
[ ] Integration test from milestone DoD passes
[ ] Documentation Agent has run and committed any doc updates
[ ] Server redeployed (if server changed): health check returns 200
[ ] Extension reloaded (if extension changed): capture smoke test passes
[ ] PR from develop to main is open, CI is green
[ ] PR has been merged
[ ] main has been pulled on host
[ ] Tag created: git tag m[N]-complete && git push origin m[N]-complete
[ ] Orchestrator ready to start milestone [N+1]
```

---

## Anti-Patterns to Reject

The following patterns, if found in an implementation, are automatic CHANGES_REQUESTED regardless of tests passing:

| Anti-pattern | Why |
|---|---|
| `db.Exec("SELECT ... WHERE id = " + id)` | SQL injection |
| `os.Open(userInput)` without `filepath.Clean` | Path traversal |
| `innerHTML = someVar` in extension | XSS on archive reopen |
| `console.log(...)` in production extension code | Information leakage, noise |
| `eval(code)` anywhere in extension | Code injection |
| `AMBER_API_KEY := "abc123"` literal | Secret exposure |
| `http.ListenAndServe("0.0.0.0:8090", ...)` | Binding to all interfaces |
| `if err != nil { return }` without logging | Silent failure, debug nightmare |
| Empty test body (`func TestFoo(t *testing.T) {}`) | Fake coverage |
| PR description that says only "fixes bug" | No context for future readers |

---

## Environment Reference

| Variable | Where set | Purpose |
|---|---|---|
| `AMBER_API_KEY` | Host environment / Docker env | Authenticates extension-to-server POST requests |
| `GITHUB_TOKEN` | Host environment / Docker env | Server pushes tracker-db exports to GitHub |
| `AMBER_DB_PATH` | Docker env (default: `/data/amber.db`) | SQLite database path inside container |
| `AMBER_ARCHIVE_PATH` | Docker env (default: `/archives`) | Where static archive files are written |

Extension reads `AMBER_API_KEY` from its own `manifest.json` externally-connectable config — never hardcoded, set at extension build time via `build.sh`.

Server listens on `127.0.0.1:8090` only. nginx reverse proxy handles TLS termination and forwards from `https://korh.one/amber/`.

---

## First-Run Verification

After a fresh deploy or after resuming development, run this sequence to confirm the stack is healthy:

```bash
# 1. Server health
curl -sf http://127.0.0.1:8090/health && echo "server OK"

# 2. Database readable
sqlite3 /root/amber/data/amber.db "SELECT count(*) FROM domains;" && echo "db OK"

# 3. Archives directory writable
touch /var/www/html/sharex/uploads/kage/.amber-write-test && \
  rm /var/www/html/sharex/uploads/kage/.amber-write-test && \
  echo "archives dir OK"

# 4. Go tests pass
cd /root/amber && go test ./... -count=1

# 5. Extension loads without errors
# Manual: chrome://extensions → amber → service worker → no console errors
```

All five must pass before beginning any development work.

---

*This playbook is a living document. Orchestrator Agent updates it at the end of each milestone. Any agent that notices an inconsistency between this playbook and actual project behavior should return the finding to the Orchestrator rather than silently following the wrong behavior.*
