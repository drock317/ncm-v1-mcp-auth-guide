# 05 — Ready-to-paste Kiro prompts

Copy one complete prompt into a new Kiro session at the indicated phase. Kiro must inspect **your** source tree, current MCP SDK and existing security controls; these are not instructions to use someone else's NCM account or copy a private server's passwords.

## Prompt 1 — Inventory and design (read-only)

```text
Act as senior MCP infrastructure and security engineer for THIS repository.
Goal: evaluate adding a human-in-the-loop browser login/MFA flow for
Cradlepoint NCM v1 /remote/status READ-ONLY data. First read this repository's
ncm-v1-mcp-auth-guide chapters 01-04 and inspect our current architecture.

Inventory MCP client and URL-elicitation support; Gateway/read tool contracts;
Python/framework and HTTP transport; NCM v2/v3 clients; credentials boundaries;
login form hosting and TLS/reverse proxy; NCM tenant region/access; service
process count; whether multiple users share this MCP.

Produce a source-backed, file-by-file integration plan with risks, tests,
dependencies, rollback and specific remaining unknowns. Distinguish VERIFIED,
PROPOSED and UNKNOWN. Prefer supported vendor APIs; do not assume an internal
vendor auth endpoint is stable or authorized.

No implementation, git changes, live logins, NCM writes or new public routes.
Never ask for a password, MFA code, cookie or bearer token in Kiro chat.
```

## Prompt 2 — Implement isolated components with fakes

```text
Implement the approved design from Prompt 1 in a focused feature branch.
Read the architecture and security guide before editing. Adapt to the
installed MCP SDK and existing service patterns; do not replace our Gateway.

Create modules or equivalents for: a fixed-endpoint NCM Accounts broker that
supports login/MFA/session completion and a RAM-only short-lived cookie;
a cryptographic nonce/CSRF flow manager with 5-minute link expiration;
a protected HTTPS browser GET/POST portal with exact Host/Origin, no-store,
CSP and capped forms; one GET-only status/... remote client with verified
HTTPS region host, path validator, positive device ID, serial request limit;
a single dependency instance shared by web routes and read tools; a bounded
MCP READ tool that emits authentication_required and a trusted link.

If this SDK/Kiro actually support MCP URL elicitation, add a Gateway handler
restricted to the registered NCM read tool and one expected auth hostname,
with bounded same-read retry; otherwise implement trusted manual URL retry.

Never accept credentials or JWT as MCP tool arguments. No arbitrary HTTP
endpoint or method, control/config tools, raw router commands or repeated
live status crawls. No changes to existing NCM v2/v3 credentials.

Use a fake Accounts service, fake router, mock time and test browser client.
Tests must cover login success with/without MFA, rejection, CSRF, nonce expiry,
redirect-host rejection, JWT expiry, no cookie, remote 401/403, forbidden path,
Gateway URL trust, cancellation and timeouts. No real NCM access.

Run repo tests and git diff --check; report test count, contracts and risks.
Commit/push for review only. Do not merge or deploy.
```

## Prompt 3 — Security review and regression gate

```text
Review the proposed NCM interactive authentication change for real flaws.
Read guide chapter 04 first. Show findings with file/line evidence.

Check: browser credentials only transient; no secret or nonce URL logs; exact
Host/Origin, CSRF and stage; callback host and redirect restrictions; SSRF;
fixed NCM endpoints; no arbitrary status/config/control; GET-only enforcement;
backend session expiry and 401/403; one process vs multiple workers;
identity binding for any multi-user service; Gateway trusted URL validation;
client support for URL elicitation; preservation of existing v2/v3 tools.

Test realistic failures with fakes only. Do not simulate successful security
without testing. If credentials can leak, session can cross users, or any
untrusted service can elicit browser navigation, mark BLOCKING and stop.

Return approve/reject recommendation, exact corrections, risk classification,
rollback plan, and verification evidence. No live NCM, merge or deployment.
```

## Prompt 4 — Controlled deployment only after approval

```text
The NCM v1 browser authentication branch has passed review. Do not deploy
until I separately approve the exact plan and commit.

Verify merged/approved SHA, clean source, no secrets, fixed trusted HTTPS
origin, TLS/reverse-proxy routing for /auth/ncm/* to the SAME backend process,
proper NCM vendor egress, Gateway URL-elicitation support, service inventory,
existing test results, backup snapshot and rollback plan.

Generate a fresh controlled deployment plan and show affected service files,
ports/routes, service restarts and rollback. Stop on unexpected changes.
Ask for explicit approval, then apply through the approved deployment helper.
Verify service health and read-only contracts only; do not automatically
perform a browser login or run a live router test. Do not expand the action
allowlist or silently turn on an unapproved Internet-facing login endpoint.
```

## Prompt 5 — One approved live authentication/read acceptance

```text
Validate one authorized LAB router and one harmless live NCOS leaf path.
First discover the exact router ID from a native NCM READ tool. Use the
registered NCM READ tool get_ncm_device_status_path with a path such as
status/system/uptime. No configs, control paths, bulk trees or writes.

Call the read unauthenticated and verify authentication_required with a
trusted temporary HTTPS URL. Verify the Gateway/client presents native MCP
URL elicitation if supported (otherwise a verified URL and manual retry).

STOP for me to use my own browser and operator-controlled approved portal.
I will type credentials and MFA THERE ONLY. Never request them in chat.
After I complete it, accept/resume the same read within the bounded wait
or explicitly retry it once. Confirm the returned device ID, path, response
success and observed runtime data. Validate expiration and reauth only with
controlled, minimal reads. Audit that there are no secrets in output/logs.

Report PASS/FAIL for nonce/CSRF, login/MFA, cookie RAM-only session,
trusted URL, bounded Gateway resume, GET-only status, health and rollback.
Do not run any state-changing NCM tool.
```
