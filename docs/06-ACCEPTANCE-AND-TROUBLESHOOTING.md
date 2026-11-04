# 06 — Testing, acceptance and troubleshooting

## Offline tests (must pass before live use)

| Case | Expected safe behavior |
|---|---|
| Missing browser session | Read returns `authentication_required` and one trusted HTTPS URL; remote GET not issued |
| Concurrent missing-auth reads | Reuse one currently valid flow instead of invalidating the previous nonce |
| Expired or invented nonce | HTTP 410; no credential validation requested |
| Wrong Host, wrong Origin | Reject request, no vendor login attempt |
| Missing/replayed CSRF or incorrect stage | Reject and keep flow unadvanced |
| Wrong form content type/oversize/duplicate fields | Reject without echoing submitted values |
| Login requires MFA | Keep short-lived pending challenge, rotate CSRF, show MFA form |
| Login returns direct success legitimately | Complete session only when vendor response permits it |
| MFA invalid or expired | Clear pending state, reset/abort safely |
| Redirect to untrusted callback host | Reject, do not follow |
| Vendor session completion lacks cookie | Fail closed, no authenticated status read |
| JWT expires or broker restarts | Forget cookie, require fresh browser login |
| Remote HTTP 401/403 | Clear session, return new auth-needed state |
| `device_id <= 0` | Reject before HTTP |
| `config/...`, `control/...`, URL, traversal, root `status` | Reject before HTTP |
| Valid narrow `status/...` | Exactly one serialized GET to a fixed, approved NCM remote host |
| Non-NCM/non-READ tool emits a login URL | Gateway must not open/elicit |
| Untrusted scheme/host/port/path/query/fragment | Gateway rejects URL |
| Kiro accepts auth, browser completes | Same registered read resumes, not a rewritten command |
| Kiro cancels or browser exceeds timeout | Clean cancellation/pending result, no infinite retry |
| Proxy/audit captures | No password, MFA, JWT, OIDC code, active nonce URL or cookie |
| Another user/client connects | Must not inherit someone else's NCM identity in a multi-user design |

## Staged acceptance

1. **Environment:** confirm HTTPS and expected auth hostname, stable backend instance, service status, and NCM account authorization.
2. **Target:** discover an eligible LAB router using ordinary NCM APIs; request only `status/system/uptime` or another small status leaf.
3. **Unauthenticated call:** check `authentication_required`, fixed-origin `login_url`, short TTL, and absence of secret output.
4. **Human browser step:** open the verified hostname, authenticate, complete MFA if required, and close success page. Do not send credentials or a copied login URL to Kiro.
5. **Resume:** Gateway retries the same read within bounded wait, or Kiro repeats manually. Confirm the response says per-device `success=true` and matches the intended router/path.
6. **Negative cases:** expired link, rejected CSRF, bad status path and remote 401/403 must fail closed; no uncontrolled repeated login attempts.
7. **Service safety:** no router configuration change, no unapproved tool registrations, no unexpected restarts, and no secret leakage in normal logs.

## Troubleshooting (one checked hypothesis at a time)

| Symptom | First evidence to inspect | Avoid |
|---|---|---|
| Kiro doesn't offer a login URL | Verify read tool discovery, `authentication_required`, and SDK client URL-elicitation support | Assuming all MCP clients show elicitation identically |
| Browser gets 404 | Check proxy path mapping for `/auth/ncm/*` and backend custom routes | Making random route changes |
| Invalid Host | Inspect trusted proxy Host forwarding and configured HTTPS origin | Disabling Host checks |
| POST gets 403 | Compare Origin exactly to configured origin, check CSRF | Disabling Origin/CSRF globally |
| Browser says link expired | Check five-minute TTL and whether service restarted | Logging raw nonce URLs |
| MFA fails | Re-enter code in verified portal, use expected restart-login semantics | Copying MFA to chat |
| Session apparently completes but remote read is unauthenticated | Verify browser portal and MCP backend share the same process/auth object | Persisting raw JWT to fix sticky sessions |
| Gateway times out | Check 180-second bounded wait and completion state; manually retry identical read | Infinite polling or generating new link every retry |
| NCM remote returns 401/403 | Validate session expiration and NCM permissions; new login may be needed | Passing browser cookie through Kiro |
| Remote path rejected | Use one documented, validated status leaf | Enabling config/control/raw URL input |
| Unexpected router data | Confirm device ID and response provenance (actual NCOS vs cached NCM) | Guessing the target's raw config |
| Secret in logs | Stop exposure, remove logging path, follow incident/rotation process | Continuing to test with real credentials |

## Exit criteria

Only declare success after: deterministic offline security tests, a human-authorized browser login/MFA path, trusted URL handoff, short-lived backend-only cookie, one correct live status GET response, healthy MCP Gateway/backend, and evidence that all negative tests fail safely. A successful HTTP response alone is not enough if the device's returned `success` field is false.

**Not validated by this reference:** current vendor-internal endpoint compatibility in every NCM region, enterprise multi-user isolation, support for all MCP clients, or any public-Internet deployment.
