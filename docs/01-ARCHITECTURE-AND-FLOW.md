# 01 — Architecture and flow

## Why this exists

The NCM v1 remote status interface requires a browser-style session with a transient `cpAccountsJwt` cookie. For the same MCP server, normal NCM v2 header credentials and v3 API bearer credentials do not replace this browser session. An AI client should not receive the human's NCM password, MFA code or JWT.

The architecture separates four boundaries:

1. **MCP READ tool:** requests a device's live NCOS `status/...` path and reports `authentication_required` when needed.
2. **Browser auth portal:** operator-controlled HTTPS forms for username/password and MFA; independent from MCP tool inputs.
3. **NCM session broker:** talks to the approved NCM Accounts service, completes applicable login/MFA/session steps and stores a cookie in process memory.
4. **Gateway/client handoff:** validates an auth URL, prompts Kiro to open it, then resumes the identical status read with a bounded wait.

## Sequence

```text
Client -> Gateway.execute_read -> backend get_ncm_device_status_path
  -> strict ID/path validation -> try GET-only remote client
  -> session absent/expired
Backend -> Gateway: authentication_required=true, login_url, TTL
Gateway -> client: trusted HTTPS URL elicitation
Operator/browser -> private HTTPS portal: login form
Portal -> vendor Accounts service: authentication request
Vendor -> portal: MFA required (if configured)
Operator/browser -> portal: MFA form
Portal -> vendor Accounts: validate MFA + complete session
Vendor -> portal: short-lived cpAccountsJwt
Portal -> backend session holder: cookie stays in process RAM
Client/Gateway -> backend: retry identical device_id + status path
Backend -> NCM remote GET: status/... + in-memory cookie
Backend -> Gateway -> client: bounded live NCOS status result
```

**Important:** the browser portal processes the password and MFA code transiently before forwarding them to NCM Accounts. This is not the same as redirecting to an official vendor-hosted login page. It is an operator-controlled credential-handling service, with corresponding security responsibilities.

## MCP read contract

A registered, read-only tool accepts only a *positive* device ID and one short status path:

```json
{"device_id": 123456, "path": "status/system/uptime"}
```

For an unauthenticated call, a representative response is:

```json
{
  "ok": false,
  "authentication_required": true,
  "login_url": "https://auth.example.com/auth/ncm/<RANDOM_NONCE>",
  "login_expires_in_seconds": 300,
  "next_step": "Open the secure browser login and complete MFA, then retry this read."
}
```

Never put the user password, MFA code, session cookie or OAuth/OIDC state in MCP arguments or tool responses. Do not put the CSRF token into the login URL either.

## Browser flow state

A practical single-process flow manager stores a cryptographic nonce, a different per-stage CSRF token, the current stage (`login` -> `mfa` -> `complete`, or `login` -> `complete`) and an expiry. Suggested reference defaults:

- Browser link lifetime **300 seconds**; expired links fail with HTTP 410.
- Generate nonce and CSRF with a cryptographically secure RNG, and compare in constant time.
- Rotate CSRF whenever the browser advances a stage.
- Verify the expected Host and the exact Origin for form POSTs.
- Reject invalid content types, oversized/duplicate fields, expired nonce, wrong CSRF or out-of-order stages.
- Make the completion page invalidate the one-use flow.

If login without MFA is legitimately supported by the tenant, the flow may go directly to session completion; it must **not** skip an MFA challenge returned by the vendor.

## In-memory session management

The broker obtains `cpAccountsJwt` only after successful NCM Accounts authentication and permitted session-completion steps. It stores it inside one backend process object, never in Git, a prompt, a tool response, a persistent session file, or standard audit logs.

Reference behavior: prefer the token's future `exp` claim for conservative local expiry; if unreadable, use a **12-minute local fallback**, with a **30-second early expiry skew**. Decoding an unverified JWT payload to schedule local expiration is **not signature validation**. The actual NCM endpoint still validates it. Restarting the service loses the session; remote 401/403 clears it.

## GET-only remote status

A status request is conceptually:

```text
GET https://<APPROVED_NCM_REGION>/api/v1/remote/status/system/uptime?id=123456
Cookie: cpAccountsJwt=<BACKEND_RAM_ONLY>
```

The remote endpoint must be a **fixed, validated HTTPS base** from trusted configuration. The client may only append a validated status subtree and a positive device ID. Reject root-only status, `config/`, `control/`, arbitrary URLs, `..`, percent escapes, fragments, query strings, and unsupported characters. Use a lock to serialize reads and prevent a legacy remote API flood. Never expose arbitrary HTTP methods or commands.

## Gateway handoff

If the MCP SDK and Kiro support native URL elicitation, the Gateway can wrap the trusted `login_url` in `InputRequiredResult`/`ElicitRequestURLParams` (names depend on MCP SDK version). It first validates all of the following: registered NCM READ tool, known backend, configured HTTPS host, acceptable port, exact `/auth/ncm/<nonce>` shape, and no query/fragment or unexpected path segments.

On explicit accepted handoff, retry the **same original read**, not a reconstructed or expanded request, for no more than **180 seconds** at approximately **2-second intervals**. On cancel/decline stop. On timeout return `authentication_pending` so the user can retry later. If Kiro does not support native URL elicitation, provide the validated link as an ordinary response and ask for manual retry after browser completion.

These are design/reference defaults; treat supported MCP SDK and client behavior as something to verify, not assume.
