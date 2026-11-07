# Architecture Summary — test-target (commit cbf2ba4)

## 1. Product, Principals, Authority, and Protected Resources

**Product:** Tiny Node.js HTTP billing and file service (single file, `app.js`). Likely a deliberately vulnerable fixture for security testing.

**Principals:**
- `alice` — authenticated via static Bearer token `tok-alice`; owns invoice #1 (amount 120)
- `bob` — authenticated via static Bearer token `tok-bob`; owns invoice #2 (amount 9800)
- Unauthenticated caller — rejected at gateway with HTTP 401

**Normal authority (authenticated user):**
- `GET /me` — read own profile (email)
- `GET /invoices/:id` — intended: read own invoice; actual: read any invoice (no ownership check)
- `GET /ping?host=` — execute `ping -c 1 <host>` via shell
- `GET /files?name=` — read file from `<__dirname>/public/`

**Protected resources:** invoice ownership (financial data), server filesystem, OS shell

## 2. Comparable-Software Baseline

No meaningful source-grounded comparable. The service is a minimal test fixture with no referenced framework, protocol spec, or design document.

## 3. Tech Stack, Deployment, and Execution Limits

- **Language/Runtime:** JavaScript, Node.js v26.9.0 (no external dependencies; built-ins only: `http`, `child_process`, `fs`, `path`)
- **Build:** None required; `node app.js` starts the server
- **Deployment:** Binds to `127.0.0.1:3000` (localhost only, hardcoded)
- **Data stores:** All in-process, hardcoded constants (SESSIONS, INVOICES, USERS)
- **No OS sandbox available for this run.** All checks are source-only. No target code executed.

## 4. Entry Surfaces and Source-to-Sink Paths

| Endpoint | Input | Sink | Control |
|---|---|---|---|
| `GET /ping?host=` | `url.searchParams.get("host")` (app.js:42) | `exec(`ping -c 1 ${host}`)` (app.js:43) | None — direct shell interpolation |
| `GET /files?name=` | `url.searchParams.get("name")` (app.js:48) | `fs.readFile(path.join(FILE_ROOT, name))` (app.js:49) | None — no path canonicalization |
| `GET /invoices/:id` | URL path capture `\d+` (app.js:33) | `INVOICES[m[1]]` returned to caller (app.js:37) | Auth only — no ownership check |
| `GET /me` | `currentUser(req)` → SESSIONS lookup | USERS[user] in response (app.js:29) | Correct — returns only caller's own data |

## 5. Trust Boundaries and Source-Visible Controls

| Boundary | Location | Control | Strength |
|---|---|---|---|
| Unauthenticated → authenticated | `currentUser()` app.js:15–18; check at app.js:22–26 | Static token lookup in SESSIONS map | Enforced at all endpoints |
| alice → bob invoice | `INVOICES[m[1]]` app.js:35 | **Missing** — no `inv.owner === user` check | **Not enforced** |
| Authenticated → shell execution | `/ping` handler app.js:41–45 | **Missing** — user input passed raw to exec | **Not enforced** |
| Authenticated → filesystem | `/files` handler app.js:47–54 | **Missing** — no path traversal guard | **Not enforced** |

## 6. Repository-Relative Starting Paths

- `app.js` — entire application (60 lines)

## 7. Prior Coverage Gaps

No compatible prior ledger exists. This is the first structured run.

## 8. Companion Selection Summary

- **WEB-PROTOCOL-AND-AUTH.md** — considered; excluded. The service uses only a static hardcoded token map (no JWT, OAuth, OIDC, SAML, CSRF, session management, or custom HTTP parser). The core `Access control` and `Cryptography and secrets` attack-class blocks from ATTACK-CLASSES.md cover all relevant boundaries without protocol-level companion sections.
- All other companions (AI-AND-LLM, MEMORY-SAFETY-AND-BINARY, CLIENT-SIDE, SUPPLY-CHAIN-AND-RELEASE, CLOUD-AND-DEPLOYMENT, PROTOCOLS-RPC-AND-MESSAGING, RESOURCE-EXHAUSTION-AND-AVAILABILITY, DATA-ISOLATION-AND-LIFECYCLE, DESKTOP-MOBILE-AND-LOCAL-IPC) — excluded; not applicable to this target.
