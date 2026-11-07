# Security Audit Report — test-target

**Run ID:** run-1  
**Profile:** quick  
**Scope:** `app.js` (entire application, 60 lines)  
**Source ref:** `cbf2ba4` (clean worktree)  
**Date:** 2026-11-08  
**Budget:** 12 agent invocations (12 spent)

## Execution statement

No OS-enforced sandbox was available for this run. Target code was not executed. All checks are source-only. No finding required dynamic execution to establish the decisive fact; all five vulnerabilities are unambiguous from static analysis.

## Coverage summary

| Status | Count |
|---|---|
| Covered (confirmed) | 5 |
| Deferred | 1 |
| Total units | 6 |

**This is a quick-profile, single-file run. It represents partial coverage of the target.** The deferred unit (resource exhaustion) was noted by the final critic and should be addressed in a follow-up run.

Coverage units not hunted in this pass: resource exhaustion against the unbounded `/ping` process spawning and `/files` read concurrency (localhost-only service with authentication; lower priority).

## Security posture summary

The application is a deliberately vulnerable Node.js fixture. Every HTTP endpoint that accepts user input has a critical or high-severity control failure:

- `/ping` has OS command injection via direct shell interpolation — the most severe finding.
- `/files` has path traversal with no canonicalization guard.
- `/invoices/:id` returns any invoice to any authenticated user (BOLA/IDOR).
- Authentication uses hardcoded, static, non-expiring tokens with no revocation capability.

The application should not be exposed beyond localhost. Any authenticated user can run arbitrary OS commands as the Node.js process UID.

## Confirmed findings

| # | Fingerprint | Title | Severity |
|---|---|---|---|
| 1 | `ping-os-command-injection` | OS command injection in GET /ping | **High** (impact: critical) |
| 2 | `files-path-traversal-no-canonicalization` | Path traversal in GET /files | **High** |
| 3 | `invoices-missing-ownership-check` | BOLA on GET /invoices/:id | **High** |
| 4 | `hardcoded-bearer-tokens-in-source` | Hardcoded session tokens in source | **High** |
| 5 | `hardcoded-static-session-tokens` | Static tokens — no expiry or revocation | **High** |
| 6 | `ping-error-information-leakage` | Error details leaked via String(err) | **Low** |

---

## Finding 1 — OS Command Injection in GET /ping

**Fingerprint:** `ping-os-command-injection`  
**Severity:** High (impact: critical; overall: high — authentication required)  
**Confidence:** High

The `/ping` endpoint passes the user-supplied `host` query parameter directly into a template literal passed to `child_process.exec`. Because `exec` invokes `/bin/sh -c`, any shell metacharacter in `host` causes arbitrary command execution with the privileges of the Node.js process.

**Source:** `app.js:43` — `exec(\`ping -c 1 ${host}\`, ...)`  
**Input:** `app.js:42` — `url.searchParams.get("host")`

**Precondition:** Valid Bearer token (tok-alice or tok-bob).

**Payloads:** `127.0.0.1;id`, `127.0.0.1;cat /etc/passwd`, `127.0.0.1$(whoami)`

**Fix:** Replace `exec` with `execFile("ping", ["-c", "1", host], ...)` and validate `host` against `/^[a-zA-Z0-9._-]{1,253}$/` before calling. See FINDINGS-DETAIL.md §F1 for the complete patch.

---

## Finding 2 — Path Traversal in GET /files

**Fingerprint:** `files-path-traversal-no-canonicalization`  
**Severity:** High  
**Confidence:** High

The `/files` endpoint constructs a file path with `path.join(FILE_ROOT, name)` where `name` is the raw query parameter. `path.join` normalizes `..` segments, so `path.join("/srv/public", "../../etc/passwd")` resolves to `/etc/passwd`. There is no `path.resolve` + `startsWith` guard.

**Source:** `app.js:49` — `fs.readFile(path.join(FILE_ROOT, name), ...)`  
**Input:** `app.js:48` — `url.searchParams.get("name")`

**Precondition:** Valid Bearer token.

**Payloads:** `../../etc/passwd`, `../../../root/.ssh/id_rsa`, `../../app.js`

**Fix:** `const resolved = path.resolve(path.join(FILE_ROOT, name)); if (!resolved.startsWith(path.resolve(FILE_ROOT) + path.sep)) return 400;` — see FINDINGS-DETAIL.md §F2 for the complete patch.

---

## Finding 3 — Broken Object-Level Authorization on GET /invoices/:id

**Fingerprint:** `invoices-missing-ownership-check`  
**Severity:** High  
**Confidence:** High

The `/invoices/:id` handler fetches the invoice by numeric ID and returns it to the caller without checking `inv.owner === user`. The authenticated user identity is in scope (`user` at line 22) but is never referenced in the invoice handler (lines 33–38). Any authenticated user can read any other user's invoice.

**Source:** `app.js:35` — `const inv = INVOICES[m[1]]` (no ownership check); `app.js:37` — `res.end(JSON.stringify(inv))`

**Precondition:** Valid Bearer token for any user.

**Payload:** `GET /invoices/2` with `Authorization: Bearer tok-alice` returns bob's invoice `{owner:"bob", amount:9800}`.

**Fix:** Add `if (inv.owner !== user) return res.writeHead(403).end();` after the null check — see FINDINGS-DETAIL.md §F3.

---

## Finding 4 — Hardcoded Session Tokens in Source Code

**Fingerprint:** `hardcoded-bearer-tokens-in-source`  
**Severity:** High  
**Confidence:** High

`const SESSIONS = { "tok-alice": "alice", "tok-bob": "bob" }` at `app.js:7` is committed as a plain-text source literal with no environment variable indirection. Any party with repository read access can authenticate as alice or bob indefinitely.

**Fix:** `const SESSIONS = JSON.parse(process.env.SESSIONS_JSON || "{}");` — see FINDINGS-DETAIL.md §F4.

---

## Finding 5 — Session Tokens Have No Expiry, Entropy, or Revocation

**Fingerprint:** `hardcoded-static-session-tokens`  
**Severity:** High  
**Confidence:** High

Independently from the source-exposure issue (F4), the token design itself is fatally flawed: tokens are static constants with no randomness, no expiry timestamp, no nonce, and no revocation path. `currentUser()` performs a plain property lookup with no time-based validation. No `crypto` module is imported. A token obtained by any means is permanently valid.

**Fix:** Implement `crypto.randomBytes`-based generation, server-side TTL storage, and `timingSafeEqual` comparison — see FINDINGS-DETAIL.md §F5 for the complete implementation.

---

## Finding 6 — Error Details Leaked via GET /ping

**Fingerprint:** `ping-error-information-leakage`  
**Severity:** Low  
**Confidence:** High

`exec(..., (err, out) => res.end(out || String(err)))` at `app.js:43` returns the Node.js `Error.toString()` in the HTTP body when stdout is empty. This exposes the internal command template (`ping -c 1 <host>`), OS error messages, and exit codes to authenticated clients.

**Fix:** Log the error server-side and return a generic `"ping failed"` response — see FINDINGS-DETAIL.md §F6.

---

## Needs validation

None. See NEEDS-VALIDATION.md.

## Hardening notes

No hardening was applied to the target. The audit describes fixes only; source was not modified.

## Deferred coverage

| Unit | Reason |
|---|---|
| Resource exhaustion — `/ping` process spawning and `/files` read concurrency | localhost-only authenticated service; lower priority; defer to follow-up run |
