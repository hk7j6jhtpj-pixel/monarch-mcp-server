# Fix log — cookie authentication failure

**Date:** 2026-07-30
**Reported symptom:** `login_setup.py` option 1 (browser cookies) always failed with:

```
❌ Cookie login failed: Cannot connect to host api.monarch.com:443 ssl:True
[SSLCertVerificationError: (1, '[SSL: CERTIFICATE_VERIFY_FAILED]
certificate verify failed: unable to get local issuer certificate (_ssl.c:1032)')]
```

**Conclusion:** cookie authentication was never broken. Two unrelated
environment defects were stacked on top of it. Verified against the live
Monarch API: the cookie path returns 25 accounts end to end.

---

## Root cause 1 — Python had no CA certificates (this produced the visible error)

The python.org macOS build of Python 3.13 configures OpenSSL to read

```
/Library/Frameworks/Python.framework/Versions/3.13/etc/openssl/cert.pem
```

but the installer does not create that file. It ships
`Install Certificates.command` and expects you to run it once. It had never
been run, so:

```
ssl.create_default_context().get_ca_certs()  ->  0 certs
```

Every HTTPS request from this interpreter failed verification — nothing to do
with Monarch, cookies, or this repo.

This was easy to misdiagnose because `curl https://api.monarch.com` succeeded
at the same moment: curl uses the macOS system trust store, while Python's
`aiohttp` uses OpenSSL's, which was empty. Same host, opposite results.

### Fix applied

```bash
"/Applications/Python 3.13/Install Certificates.command"
```

Verified afterwards: `121` CA certs loaded; TLS to `api.monarch.com` succeeds
from the global `python3`; full cookie login returns 25 accounts.

If this ever regresses, the per-run workaround is:

```bash
SSL_CERT_FILE=$(python3 -c "import certifi;print(certifi.where())") python3 login_setup.py
```

---

## Root cause 2 — the MCP server could never launch

Independent of the SSL problem, `~/Library/Application Support/Claude/claude_desktop_config.json`
had three faults in the `"Monarch Money"` entry:

1. `"command": "/opt/homebrew/bin/uv"` — `uv` is not installed on this machine.
   The process could not start at all.
2. `--with-editable /path/to/your/monarch-mcp-server` — the literal placeholder
   from the README, never replaced with the real checkout path.
3. `--with mcp[cli]` unpinned — now resolves to **mcp 2.0.0**, which removed
   `mcp.server.fastmcp`. `app.py` imports that module, so the server raises
   `ModuleNotFoundError` on import even if the first two were fixed.

`login_setup.py` never imports `mcp`, which is why the SSL error was the only
symptom ever seen — the config faults were hidden behind it.

### Fix applied

Created a dedicated venv with the dependency pinned:

```bash
python3 -m venv .venv
.venv/bin/pip install -e . 'mcp[cli]<2'
```

Rewrote the config entry to use the project's own console script (no `uv`, no
placeholder path, correct pin). Previous config backed up to
`claude_desktop_config.json.bak`:

```json
"Monarch Money": {
  "command": "/Users/willmiles/Applications/git/monarch-mcp-server/.venv/bin/monarch-mcp-server",
  "args": []
}
```

### Repo changes

- `pyproject.toml` — `mcp[cli]>=1.10.0` → `mcp[cli]>=1.10.0,<2`, with a comment
  naming the reason.
- `requirements.txt` — same pin.

Worth upstreaming; any fresh install of this repo today hits the mcp 2.0 break.

---

## Investigated and cleared

- **Cookie parsing.** `session_id` and `csrftoken` both parsed correctly.
  Base64 `=` padding and newline-wrapped pastes are handled fine. Note for
  future debugging: pasting the header *with* its `cookie:` name prefix yields
  the misleading error `Missing required cookies: session_id`.
- **Session storage round-trip.** save → load → rebuild client is clean;
  `device_uuid` is preserved, `Authorization` correctly dropped in cookie mode,
  `X-Csrftoken` set, cookies reach the aiohttp transport.
- **Test suite.** 222 passed.

## Known, non-blocking

`monarch_auth.create_monarch_client()` sets
`monarch-client: monarch-core-web-app-graphql` / `monarch-client-version: v1.0.1668`,
but the upstream `set_cookies()` then overwrites both with its own stale
`web` / `2025.05` via `self._headers.update(MONARCH_COOKIE_HEADERS)`.

Tested both variants against the live API — **both return 25 accounts**.
Monarch enforces the client version on the *login* endpoint, not on GraphQL
with a cookie session, so this is cosmetic today. It would become real if
Monarch starts gating GraphQL on it. The fix, if ever needed, is to re-apply
the pinned headers after `set_cookies()` in both
`login_with_browser_cookies()` and `SecureMonarchSession.get_authenticated_client()`.

Also note `tests/test_secure_session.py::test_cookie_mode_calls_set_cookies_on_client`
passes a `MagicMock` client, so the real `set_cookies()` never executes in the
suite — cookie-mode header behavior is untested.

---

## Next step for the user

Restart Claude Desktop, then re-authenticate:

```bash
python3 login_setup.py
```

The session cookie used for this debugging session was shared in chat and
should be rotated — log out of Monarch in the browser to invalidate it, then
grab a fresh one.
