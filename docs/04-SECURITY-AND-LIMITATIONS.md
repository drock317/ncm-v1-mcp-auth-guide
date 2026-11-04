# 04 — Security and limitations

## Credential and identity boundaries

| Component | What it may see | What it must never expose |
|---|---|---|
| Kiro / LLM client | Trusted login **URL** (nonce in path) and bounded status result | Email/password/MFA/JWT, OIDC state/callback code |
| MCP Gateway | Backend `authentication_required` result and trusted URL | Credentials/MFA/JWT; arbitrary redirect URL |
| HTTPS login portal | Form email/password, then MFA as transient request inputs | Credentials in logs, URL, redirects, persistent state, error messages |
| In-memory auth broker | Pending vendor session state, final short-lived cookie | Cookies/tokens in tools, logs, crash bundles or Git |
| GET-only remote reader | Cookie from internal auth object | Arbitrary config/control, client-supplied cookies or URLs |

The service hosting the browser forms is trusted with the operator's real NCM credentials **during transit through that process**. This is a material risk: do not present the pattern as delegating authentication entirely to the vendor's hosted login screen.

## Essential controls

- Use a fixed trusted HTTPS login hostname and reject unexpected Host/Origin.
- Validate a cryptographic random nonce with short TTL; bind POSTs to distinct CSRF tokens and correct stages.
- Reject duplicate form fields, oversized bodies, invalid Unicode, wrong content types, expired state and replay.
- Never render user-entered passwords/MFA or token values in HTML or errors. Disable access/request tracing that logs bodies or full nonce-bearing paths.
- Limit vendor authentication egress and redirect/callback hosts. Reject external callback origins. No arbitrary HTTP method/url/body tool.
- Store session cookie only in process memory and fail closed on expiry, rejected login, MFA failure, missing cookie, or remote 401/403.
- Treat JWT `exp` decode as local expiry scheduling, **not** cryptographic verification.
- Require the Gateway to validate that *only an authorized READ tool from the expected service* can trigger URL elicitation.
- Use strictly bounded, serialized GET-only `status/...` reads. Don't permit status root crawling or configuration/control reads.
- Keep passwords, cookies, MFA codes, nonces, API keys, router identifiers, subscriber data, private domains and credentials out of source control.

## Critical limitation: shared session

The reference uses one auth object in one process. After a trusted operator logs in, **other clients that can reach the same backend may share that session**. No per-user principal binding is included. This is unacceptable for general multi-user or multi-tenant use without redesign: each user needs authenticated MCP identity, separately bound browser flow, separate vendor session, entitlement isolation, audit, revocation and concurrency testing.

Do not solve session loss across multiple workers by persisting a JWT unencrypted or by putting it in a common cache without appropriate identity binding and security review.

## Public URL / proxy risks

Opening an HTTPS portal on an Internet-facing reverse proxy increases exposure. A nonce does not replace access control, TLS, host verification, rate limiting and monitoring. Proxy path logs can disclose nonce-bearing login URLs even when query-string logging is disabled. Consider a private network or access gate that doesn't break the intended browser login flow.

## Vendor protocol caution

The authentication sequence may rely on NCM Accounts **internal** endpoints or release-specific behavior. This publication intentionally does not reproduce those endpoint bodies or claim an officially supported general OAuth integration. Verify vendor documentation, policy, approved hosts, redirects and required behavior for the actual NCM region and tenant. Prefer a supported integration where it satisfies the goal. Avoid automatically scraping or replaying browser flows to bypass MFA.

## Read-only does not mean harmless

A remote status path can still reveal sensitive device and network data (IP addresses, SIM identities, infrastructure topology, private hosts). Normalize and bound returned status, enforce NCM permissions, authenticate the requester, and avoid debug outputs containing secrets. Differentiate data observed in live NCOS from configuration claims inferred by Kiro.

## Repository permissions

If this guide is hosted in a public GitHub repo, strangers can **read** it and may clone/fork it. Unless given collaborator privileges, they cannot directly push changes to the owner's repo. For a reader-only publication, disable Issues, Discussions and Pull requests where available, and do not invite collaborators. This is about direct access to *your repo*, not restricting how public documentation may be copied under its license.

## Disclaimer

Security claims here are design requirements to test, not a third-party penetration test or vendor approval. Do not go live without approval and your own review of authentication exposure, user isolation and legal/vendor requirements.
