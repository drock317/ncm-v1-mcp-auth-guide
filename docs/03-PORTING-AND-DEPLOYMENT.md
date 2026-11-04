# 03 — Porting and deployment playbook

## 1. First inspect the receiving project

Determine the runtime (Python/MCP SDK version), existing Gateway, backend services, HTTP transport, reverse proxy, trusted HTTPS hostname, credentials model, current NCM v2/v3 clients, how Kiro handles URL elicitation, and whether the service is single-user or multi-user. Do not replace a functioning authentication system with this pattern without a demonstrated need.

**Stop condition:** if the server is shared by multiple people and cannot bind a browser flow/session to a verified individual MCP principal, do not implement the shared singleton variant.

## 2. Integrate in dependency order

1. **Auth broker:** fixed trusted NCM Accounts endpoint set, login/MFA/session-completion adapter, memory-only cookie, expiry and close/clear. Handle vendor errors using safe bounded messages and never log forms or callbacks.
2. **Flow manager:** cryptographic nonce and CSRF, TTL, explicit login/MFA/complete stages; mockable clock and deterministic tests.
3. **Browser portal:** secured HTTPS routes for login and MFA, with Host and Origin checks, input validation, CSP, no-store, safe HTML and limited request bodies.
4. **Remote reader:** strictly GET-only status path reader; hardcode/allowlist a verified NCM region URL; positive device ID, serialized calls, and remote 401/403 invalidation.
5. **Dependency wiring:** one instance of auth and flow manager shared by HTTP routes and the status tool in the same backend process.
6. **NCM tool:** emit `authentication_required` with a short-lived HTTPS portal link when the status read cannot authenticate.
7. **Gateway bridge (optional):** native URL elicitation for a trusted NCM READ only; bounded same-read retry after human acceptance, or manual retry fallback.

At each stage preserve existing NCM v2/v3 behavior and already-reviewed tool registrations. No new action tools are required.

## 3. Reverse proxy and networking

```text
Kiro -> https://gateway.example.com/mcp -> Gateway service -> NCM backend
Browser -> https://auth.example.com/auth/ncm/<nonce>
        -> trusted reverse proxy -> SAME NCM backend process holding flow state
NCM backend -> verified HTTPS NCM Accounts host (login/MFA/session completion)
NCM backend -> verified HTTPS NCM regional remote endpoint (status-only GET)
```

**Checklist:** TLS certificate valid, HTTPS-only, DNS resolves from browser, reverse proxy preserves expected Host and Origin semantics, browser endpoints route to the correct service, backend egress works, form POST limits are enforced, logs do not expose path nonce or body, and no unrelated virtual host can reach authentication routes. Never disable TLS verification or Host/Origin checks just to get a successful login.

Use a private portal or protected-access layer when appropriate. The mere presence of a random nonce does not make a password-collecting endpoint safe for unrestricted Internet exposure.

## 4. Integration testing before deployment

Use fake NCM Accounts responses and fake router-status returns. Test correct MFA and no-MFA branches, callback host rejection, missing cookie, timeout, CSRF replay, expired nonce, wrong origin, invalid status path, remote 401/403, cross-session contamination, and Gateway trusted link validation. Run current project contract/regression tests and check that no secrets were added to commits.

Do not make real login requests in CI. Do not test a production router while implementing the portal.

## 5. Controlled deployment

Record the base Git SHA, target Git SHA and service manifest. Verify deployment plan affects only intended backend/Gateway files and ports. Check proxy configuration and whether every auth request will reach the same process instance. Acquire operator approval for any externally reachable login endpoint and any production change. Keep a rollback snapshot. Test service startup and known read-only operations before initiating a browser login.

## 6. Operator-approved live acceptance

- Select one permitted LAB device using an existing NCM read tool. Choose a narrow path such as `status/system/uptime`.
- First call it unauthenticated; verify a **trusted** auth URL is returned, without any credential material.
- Human opens the operator-approved HTTPS portal, checks hostname, enters password and MFA **there only**.
- Client accepts native URL elicitation, or user manually retries the same read once the browser form says complete.
- Verify actual live status `success=true`, correct device ID/path, and no persistent configuration change.
- Verify expiry/reauth, Gateway/backend health, safe audit logging, no unattended loops.

## 7. Rollback

Restore prior source and configuration using the project's controlled deployment mechanism; restart only affected backend/Gateway services. Confirm the short-lived in-memory session disappears after restart and previous NCM v2/v3 tools still work. Remove unintended auth proxy exposure and revert corresponding proxy routing. Do **not** store JWT in a file to survive rollback.

## 8. License and publication

Keep copies of the private source and logs out of any public derivation. This repository is a standalone documentation reference under GPL-3.0. For production software, review your own service license, vendor terms and authorization requirements before integrating any authentication protocol. Users are responsible for securing their own private broker and selecting an appropriate NCM authentication mechanism.
