# NCM v1 Remote Status Authentication for MCP

**A practical architecture and Kiro integration guide** for safely reading live Cradlepoint NCOS `status/...` data through NetCloud Manager's legacy v1 remote interface.

This is a **documentation-only reference**, not a ready-to-run authentication server. It describes a human-in-the-loop browser login/MFA flow with short-lived, memory-only session state, a strictly GET-only NCOS status reader, and an optional MCP Gateway bridge that can prompt a client such as Kiro to open the login page.

**Intended audience:** MCP developers and infrastructure engineers building a *single-trusted-operator* integration. This guide does not grant any NCM access or bypass MFA. Do not paste credentials or real authentication URLs into AI conversations.

## Read the guide

| Chapter | Purpose |
|---|---|
| [01 — Architecture and flow](docs/01-ARCHITECTURE-AND-FLOW.md) | End-to-end sequence, session mechanics, and response contracts |
| [02 — Files and contracts](docs/02-FILE-MAP-AND-CONTRACTS.md) | Module responsibilities, interfaces, structure and design dependencies |
| [03 — Porting and deployment](docs/03-PORTING-AND-DEPLOYMENT.md) | Adapting to another MCP server, TLS/reverse proxy, staged rollout |
| [04 — Security and limitations](docs/04-SECURITY-AND-LIMITATIONS.md) | Threat model, isolation boundaries, credential protection |
| [05 — Kiro prompts](docs/05-KIRO-COPY-PASTE-PROMPTS.md) | Five prompts for discovery, implementation, review, deployment, acceptance |
| [06 — Acceptance and troubleshooting](docs/06-ACCEPTANCE-AND-TROUBLESHOOTING.md) | Detailed fake-based tests and safe live acceptance |

## System overview

```text
Kiro / MCP client
    |
    | execute_read("get_ncm_device_status_path", device_id, "status/...")
    v
MCP Gateway ---------------> NCM MCP backend
    ^                              |
    |                              | No valid browser session?
    |<----- authentication_required + short-lived login URL
    |
    | Present trusted HTTPS URL (native MCP URL elicitation, if supported)
    v
Human browser -> operator-controlled HTTPS login portal
                -> NCM Accounts login + MFA + session completion
                -> backend stores short-lived session cookie in RAM only
    |
    v
Gateway retries the SAME bounded GET status read
    -> NCM legacy remote status endpoint -> live NCOS status
```

NCM v2 API keys, v3 API bearer tokens, and the legacy v1 browser session are **different credential mechanisms**. The v1 remote path uses an interactive session; v2/v3 keys do not automatically replace it.

## Design boundaries

- **READ only:** one validated `status/...` path per request; no configuration, control, shell, or arbitrary HTTP passthrough.
- **Human login:** the browser page receives the operator's credentials and MFA transiently; the AI client does **not**. Only operate a portal you trust.
- **Short lived:** browser link expires, CSRF rotates, login cookie stays in backend RAM, and remote 401/403 invalidates it.
- **One trusted operator:** the reference uses one shared in-process session. It is **not** multi-user SSO or per-user delegation.
- **Vendor compatibility:** any internal authentication endpoints must be verified under applicable vendor terms and current documentation; this guide deliberately does not republish private source code or undocumented request bodies.

## Publication and contributions

This is a **reader-oriented reference**. Issues, discussions and pull requests may be disabled by the repository owner. Public readers do not receive write permission. Public GitHub content can nevertheless be viewed, cloned, and forked; repository visibility is not a copy-prevention system.

This sanitized handbook is a separate publication; it does **not** link into or depend upon the original private MCP repository and contains no real login links, router IDs, network addresses, credentials or session data.

## License

GPL-3.0; see [LICENSE](LICENSE). This retains the license family of the source documentation. It does not imply any license or support grant from Ericsson, Cradlepoint or the operator of an NCM account.
