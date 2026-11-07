# Findings Detail — test-target run-1

Source ref: `cbf2ba4` (clean). All checks are source-only; no target code was executed.

---

## F1 — OS Command Injection via GET /ping

**Fingerprint:** `ping-os-command-injection`  
**Severity:** High (impact: critical; overall: high — requires authentication)  
**Confidence:** High

### Source path

```
app.js:3   const { exec } = require("child_process")        // exec invokes /bin/sh -c
app.js:42  const host = url.searchParams.get("host") || "127.0.0.1"   // untrusted input
app.js:43  exec(`ping -c 1 ${host}`, (err, out) => res.end(out || String(err)))
           //   ^^^^^^^^^^^^^^                              // template literal, no sanitization
```

### Why this is exploitable

`child_process.exec` passes its first argument to `/bin/sh -c`. Template literal interpolation puts the raw query string into that argument. The shell interprets `;`, `&&`, `||`, `$()`, and backticks as command separators and substitutions. No escaping, no allowlist, and no use of the safe `execFile` with an argument array exists anywhere in the handler.

### Reproduction (source-verified, not executed)

```
GET /ping?host=127.0.0.1;id HTTP/1.1
Authorization: Bearer tok-alice

→ /bin/sh -c 'ping -c 1 127.0.0.1;id'
→ stdout contains uid=...
→ 200 OK — command output in body
```

Additional payloads: `127.0.0.1;cat /etc/passwd`, `127.0.0.1$(whoami)`, `` `whoami` ``

### Fix

Replace `exec` with `execFile` and validate host against an allowlist:

```javascript
// app.js — replace the /ping handler
const { execFile } = require("child_process");
// ...
if (url.pathname === "/ping") {
  const host = url.searchParams.get("host") || "127.0.0.1";
  const SAFE_HOST = /^[a-zA-Z0-9._-]{1,253}$/;
  if (!SAFE_HOST.test(host)) return res.writeHead(400).end("invalid host");
  execFile("ping", ["-c", "1", host], (err, out) => res.end(out || "ping failed"));
  return;
}
```

---

## F2 — Path Traversal via GET /files

**Fingerprint:** `files-path-traversal-no-canonicalization`  
**Severity:** High  
**Confidence:** High

### Source path

```
app.js:13  const FILE_ROOT = path.join(__dirname, "public")   // intended boundary
app.js:48  const name = url.searchParams.get("name") || "index.txt"   // untrusted input
app.js:49  fs.readFile(path.join(FILE_ROOT, name), ...)
           //          ^^^^^^^^^^^^^^^^^^^^^^^^^^          // no canonicalization guard
```

### Why this is exploitable

`path.join` normalizes `..` segments at the OS level. `path.join("/srv/public", "../../etc/passwd")` resolves to `/etc/passwd`. There is no `path.resolve` call, no `startsWith(FILE_ROOT)` guard, and no character filtering between line 48 and line 49. The 404 branch fires only on an `fs` error, not on path escape — so a valid file outside `public/` is served with HTTP 200.

### Reproduction (source-verified, not executed)

```
GET /files?name=../../etc/passwd HTTP/1.1
Authorization: Bearer tok-alice

→ path.join("/path/to/public", "../../etc/passwd") = "/etc/passwd"
→ fs.readFile("/etc/passwd", ...) reads and returns file contents
→ 200 OK — file contents in body
```

Additional paths: `../../etc/shadow`, `../../../root/.ssh/id_rsa`, `../../app.js`

### Fix

```javascript
// app.js — replace the /files handler
if (url.pathname === "/files") {
  const name = url.searchParams.get("name") || "index.txt";
  const resolvedPath = path.resolve(path.join(FILE_ROOT, name));
  const resolvedRoot = path.resolve(FILE_ROOT);
  if (!resolvedPath.startsWith(resolvedRoot + path.sep) && resolvedPath !== resolvedRoot) {
    return res.writeHead(400).end("invalid path");
  }
  fs.readFile(resolvedPath, (err, data) => {
    if (err) return res.writeHead(404).end();
    res.end(data);
  });
  return;
}
```

---

## F3 — Broken Object-Level Authorization on GET /invoices/:id

**Fingerprint:** `invoices-missing-ownership-check`  
**Severity:** High  
**Confidence:** High

### Source path

```
app.js:9   INVOICES = { "1": { owner: "alice", amount: 120 }, "2": { owner: "bob", amount: 9800 } }
app.js:22  const user = currentUser(req)   // alice or bob — authenticated and in scope
app.js:33  const m = url.pathname.match(/^\/invoices\/(\d+)$/)
app.js:35  const inv = INVOICES[m[1]]      // fetched by ID, no ownership check
app.js:37  res.end(JSON.stringify(inv))    // returned to ANY authenticated user
```

The `user` variable is never referenced between lines 33 and 38. `inv.owner` is populated but never compared to `user`.

### Reproduction (source-verified, not executed)

```
GET /invoices/2 HTTP/1.1
Authorization: Bearer tok-alice    ← alice's token

→ INVOICES["2"] = { owner: "bob", amount: 9800 }
→ No ownership check
→ 200 OK: {"owner":"bob","amount":9800}   ← bob's financial data returned to alice
```

### Fix

```javascript
// app.js — replace lines 33–38
const m = url.pathname.match(/^\/invoices\/(\d+)$/);
if (m) {
  const inv = INVOICES[m[1]];
  if (!inv) return res.writeHead(404).end();
  if (inv.owner !== user) return res.writeHead(403).end();
  res.end(JSON.stringify(inv));
  return;
}
```

---

## F4 — Hardcoded Session Tokens in Source Code

**Fingerprint:** `hardcoded-bearer-tokens-in-source`  
**Severity:** High  
**Confidence:** High

### Source path

```
app.js:7   const SESSIONS = { "tok-alice": "alice", "tok-bob": "bob" }
           // plain-text string literals committed to source
app.js:17  return SESSIONS[auth.replace(/^Bearer /, "")] || null;
```

No `process.env` reference, no external loading, no secrets manager call anywhere in the file.

### Impact

Any party with repository read access (developer, CI, contractor, compromised dependency tooling) can extract `tok-alice` and `tok-bob` and authenticate as alice or bob indefinitely. The tokens never expire and cannot be rotated without a code change and redeployment.

### Fix

```javascript
// app.js — line 7
const SESSIONS = JSON.parse(process.env.SESSIONS_JSON || "{}");
```

---

## F5 — Session Tokens Have No Expiry, Revocation, or Entropy

**Fingerprint:** `hardcoded-static-session-tokens`  
**Severity:** High  
**Confidence:** High

### Source path

```
app.js:1-5  require("http"), require("url"), require("child_process"), require("fs"), require("path")
            // no "crypto" import — no randomness in auth path
app.js:7    SESSIONS = { "tok-alice": "alice", "tok-bob": "bob" }
            // static constant data, no generation, no timestamp, no nonce
app.js:15   function currentUser(req) {
app.js:17     return SESSIONS[auth.replace(/^Bearer /, "")] || null;
              // plain property lookup — no expiry check, no constant-time comparison
```

### Why this is distinct from F4

Even if tokens were moved to environment variables (fixing F4), the absence of expiry, revocation, and randomness would remain independently exploitable. A compromised token (intercepted on the network, leaked in logs, or obtained via social engineering) is permanently valid. There is no incident-response path to invalidate a session without redeploying.

### Fix

```javascript
const { randomBytes, timingSafeEqual } = require("crypto");
const SESSION_TTL_MS = 3600 * 1000;
const sessions = new Map(); // token -> { user, expiresAt }

function createSession(user) {
  const token = randomBytes(16).toString("hex");
  sessions.set(token, { user, expiresAt: Date.now() + SESSION_TTL_MS });
  return token;
}

function currentUser(req) {
  const auth = req.headers["authorization"] || "";
  const token = auth.replace(/^Bearer /, "");
  const session = sessions.get(token);
  if (!session || Date.now() > session.expiresAt) return null;
  return session.user;
}
```

---

## F6 — Error Information Leakage via GET /ping

**Fingerprint:** `ping-error-information-leakage`  
**Severity:** Low  
**Confidence:** High

### Source path

```
app.js:43  exec(`ping -c 1 ${host}`, (err, out) => res.end(out || String(err)))
           //                                                        ^^^^^^^^^^
           // Node.js Error.toString() includes message, exit code, and command
```

### Impact

When `out` is empty (exec errors, invalid hostname, unreachable host), `String(err)` serializes the full Node.js `Error` object, which includes text like `Error: Command failed with exit code 1: ping -c 1 <host>`. This reveals the internal command template, OS error messages, and exit codes to authenticated clients.

### Fix

```javascript
exec(`ping -c 1 ${host}`, (err, out) => {
  if (err) { console.error(err); return res.writeHead(500).end("ping failed"); }
  res.end(out);
});
```

Note: The command injection in F1 (ping-os-command-injection) must be fixed independently. This fix alone does not prevent injection.
