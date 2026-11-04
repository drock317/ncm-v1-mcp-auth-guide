# 02 — File map and integration contracts

The module names below define an **illustrative integration layout**. Adapt to your own codebase instead of copying an entire private MCP server. No proprietary source files or production runtime secrets are included in this publication.

```text
my-mcp/
  services/ncm/
    ncm_v1_auth.py        # credential exchange + in-memory session
    ncm_auth_flow.py      # ephemeral nonce / stage / CSRF
    ncm_auth_web.py       # separate browser GET + POST endpoints
    ncm_v1_remote.py      # strict status-only GET client
    deps.py               # one shared auth instance
    server.py             # web route + MCP tool registration
    domains/cellular.py   # get_ncm_device_status_path read tool
    tests/                # mocked vendor auth, state and router reads
  services/gateway/
    server.py             # optional native URL elicitation bridge
    tests/                # allowlist/URL trust/resume tests
```

## Module-by-module specification

| Module | Minimum interface | Required behavior |
|---|---|---|
| `ncm_v1_auth.py` | `begin_login(email,password)`; `submit_mfa(code)`; `complete_session()`; `is_authenticated()`; `clear()` | Only fixed vendor-auth destinations; support MFA challenge; hold short-lived cookie in RAM; no logs of secrets; reject untrusted session-completion redirects |
| `ncm_auth_flow.py` | `issue()`; `form_context(nonce)`; `validate_post(nonce,csrf,stage)`; `transition(...)`; `abort()` | Random nonce/CSRF; 5-minute expiry; strict per-stage transitions; single-use completion |
| `ncm_auth_web.py` | protected GET `/auth/ncm/{nonce}`; POST `/auth/ncm/{nonce}/login`; POST `/auth/ncm/{nonce}/mfa` | HTTPS-facing login/MFA forms; Origin/Host/CSRF; no-store and security headers; safe generic errors; no secret echo |
| `ncm_v1_remote.py` | `normalize_status_path(path)`; `get_status_path(device_id,path)` | Only approved v1 `/remote/status/...` GET; cookie internally acquired; serial requests; clear session on 401/403 |
| `deps.py` | `get_v1_auth()`, `get_v1_remote_client()`, one `auth_flows` singleton | Browser handlers and MCP tool share *the same* process/session holder |
| `server.py` | register protected routes + one bounded MCP READ | No arbitrary MCP auth tool; transport/host controls; lazy client creation |
| `domains/cellular.py` | `get_ncm_device_status_path` | Catch auth-needed exception and produce `authentication_required` response; do not return credential data |
| `gateway/server.py` | `execute_read`, URL trust validation, optional native elicitation | Only a known NCM READ may elicit; bounded same-request retry; no secret/URL passthrough from arbitrary services |

## Credentials by API family

| Family | Intended scope | Credential form |
|---|---|---|
| NCM v2 | Fleet, monitoring and documented resources | Header key set in server-managed credentials store |
| NCM v3 | Supported cloud services | Scoped bearer token inside server-managed credentials store |
| NCM v1 remote | Live NCOS status through legacy browser remote path | Short-lived `cpAccountsJwt` cookie derived from interactive human login/MFA |

Do not try to use an NCM v2 API key as a browser cookie or prompt the user to paste the cookie into an MCP argument.

## Important contracts

1. **Shared auth instance:** `ncm_auth_web` and `ncm_v1_remote` use the same session broker. Creating a new object per HTTP request will break login.
2. **Fixed hosts:** the approved HTTPS NCM Accounts host and tenant's regional v1 remote host are trusted configuration only. Neither comes from MCP tool arguments.
3. **Browser route boundary:** browser routes are separate HTTP endpoints, not MCP tools. Their authentication, CSRF and rate/size limits are independent of MCP authorization.
4. **Read-only tool:** no `request(method,url)` helper, no `POST /remote`, no `config/`, no `control/` or broad status dump.
5. **One-session limitation:** one shared in-memory cookie may apply to all callers reaching that backend. Do not deploy to a multi-user system without principal isolation and threat review.
6. **Native URL elicitation:** verify the receiving client and MCP SDK actually implement URL elicitation and multi-round accepted input. A plain link + manual retry is a valid fallback.

## Recommended dependency and config values

- MCP SDK and web framework: lock versions that support custom browser routes on the same backend and native input elicitation where applicable.
- HTTP client: strict TLS validation, no unrestricted redirects, bounded timeouts; suppress request URL logs carrying transient OIDC state.
- Browser flow TTL: 300 seconds; JWT fallback 720 seconds; safety skew 30 seconds.
- Auth form body: small capped size (e.g., 8192 bytes), strict UTF-8 URL-encoded parsing; deny duplicate fields.
- Gateway accepted-auth retry: at most 180 seconds, roughly 2-second interval.
- Remote status: one request per call, 30-second network timeout, serialized in-flight requests.

The precise vendor authentication endpoint paths and request bodies are intentionally **not reproduced** here: they may be internal, region- or release-specific, and must be validated against authorized vendor resources. Kiro should inspect permitted documentation and available supported APIs before implementing an adapter.
