<!--
SPDX-FileCopyrightText: 2026 Kai Ole Hartwig <mail@ole-hartwig.eu>
SPDX-License-Identifier: MIT
-->

# Threat model — koh Sylius agent-readiness family

Status: **initial model, written before the first public release.** Built from
the source of the packages listed below, not from a penetration test. It is refreshed before
every MAJOR release and whenever a package adds a trust boundary.

## Scope

One model for the family, because the packages share one request pipeline,
one context object and one response listener. It covers:

| Package | Surface |
|---|---|
| `koh/sylius-agent-contracts` | `AgentContext` (channel, locale, base URL), provider registries, the shared response-header listener, `koh:agent:check` |
| `koh/sylius-robots-txt` | `GET /robots.txt`, `Content-Signal` header |
| `koh/sylius-markdown` | `Accept: text/markdown` negotiation on storefront routes |
| `koh/sylius-api-catalog` | `GET /.well-known/api-catalog`, `Link` headers |
| `koh/sylius-mcp-oauth` | OAuth 2.1 authorization server for MCP clients, access-token authenticator |
| `koh/sylius-web-bot-auth` | `GET /.well-known/http-message-signatures-directory` |
| `koh/sylius-x402` | HTTP 402 payment gate on configured paths |
| `koh/sylius-agent-ready` | metapackage, no code of its own |

Out of scope: Sylius, Symfony, API Platform and Doctrine themselves; the
host application's firewall and admin user management; any payment verifier
or key provider the operator plugs in; sibling packages not listed above.

## Architecture and trust boundaries

```text
            Internet (anonymous crawlers, agents, browsers)
                 │  B1: HTTP request → Symfony kernel
   ┌─────────────┴───────────────────────────────────────────┐
   │  ingress / reverse proxy  (B2: Host, X-Forwarded-*)       │
   │  Symfony firewall (priority 8)                            │
   │    ├─ discovery documents (robots, api-catalog, key dir)  │
   │    ├─ markdown listener  (priority 4, after firewall)     │
   │    ├─ x402 listener      (priority 5, after firewall)     │
   │    ├─ OAuth endpoints    (register/token/revoke: public;  │
   │    │                      authorize: admin firewall)      │
   │    └─ /_mcp  (access-token authenticator)                 │
   │  shared response-header listener (Link, Vary, CORS)       │
   │  B3: shared HTTP cache / CDN in front of all of it        │
   │  B4: database (OAuth clients, codes, token hashes)        │
   │  B5: configuration and secrets (seeds, pay_to, TTLs)      │
   │  B6: operator-supplied extensions (verifiers, providers)  │
   └───────────────────────────────────────────────────────────┘
```

| ID | Boundary | Crossing |
|---|---|---|
| B1 | Client → application | Every request; clients are unauthenticated unless stated |
| B2 | Proxy → application | `Host` and forwarded headers used to build absolute URLs |
| B3 | Application → shared cache | Public responses (`max-age`, `stale-while-revalidate`) |
| B4 | Application → database | OAuth state written and read by `sylius-mcp-oauth` |
| B5 | Deployment → application | Configuration, environment secrets |
| B6 | Package → operator code | Tagged providers, verifiers and key providers |

## Assets

- Admin authority delegated to agents (OAuth access tokens, authorization codes).
- Signing identity of the shop's outbound bots (Web Bot Auth seed).
- Revenue and access control on paid resources (x402).
- Integrity of discovery documents agents act on (URLs, prices, policies).
- Storefront availability and cacheability.

## Attacker model

- **Anonymous remote client** — can send any request, header and body; can
  register OAuth clients (registration is open by protocol design).
- **Malicious agent or crawler** — ignores `robots.txt` and Content Signals.
- **Phishing third party** — can get a signed-in administrator to open a link.
- **Not assumed:** code execution on the host, database access, a compromised
  proxy, a malicious operator.

## STRIDE

| ID | Boundary | Category | Threat | Mitigation |
|---|---|---|---|---|
| THR-AGENT-001 | B2/B3 | Tampering | Forged `Host` header yields absolute URLs pointing elsewhere, then cached and served to others | Base URL comes from `Channel::getHostname()`; request host only with `trust_request_host: true` (requires `framework.trusted_proxies`). See THR-AGENT-013 for the fallback |
| THR-AGENT-002 | B3 | Info disclosure | One channel's document served to another channel | Cache keys only via `AgentContext::cacheKey()` (channel + locale) |
| THR-AGENT-003 | B3 | Info disclosure | Session cookie set on discovery responses, cached publicly | Documented firewall exemption; `koh:agent:check` flags any cookie on discovery responses |
| THR-AGENT-004 | B1 | Elevation | `Accept: text/markdown` bypasses access control on protected routes | Markdown listener runs after the firewall (priority 4 < 8) |
| THR-AGENT-005 | B3 | Info disclosure | Markdown and HTML variants cross-served by a shared cache | `Vary: Accept` on both variants of negotiable routes |
| THR-AGENT-006 | B1 | Denial of service | Markdown requests force a full render per hit | Listener short-circuits before the controller; responses carry ETag and `max-age` |
| THR-AGENT-007 | B1 | Spoofing | Authorization code intercepted and redeemed | PKCE S256 mandatory; codes single-use, 300 s TTL, bound to client and redirect URI; compared with `hash_equals` |
| THR-AGENT-008 | B4 | Info disclosure | Database read exposes usable tokens or codes | Only SHA-256 hashes of tokens and codes are stored; raw value shown once |
| THR-AGENT-009 | B1 | Tampering | CSRF or clickjacking on the consent form issues a code for the admin | Consent is `POST` with a CSRF token under the admin firewall; consent and error pages send `X-Frame-Options: DENY` and `frame-ancestors 'none'` (mcp-oauth v2.0.0) |
| THR-AGENT-010 | B1 | Tampering | Open redirect via a bad `client_id` / `redirect_uri` | Mismatches render an on-site error and are never redirected to |
| THR-AGENT-011 | B5 | Spoofing | Leaked `APP_SECRET` lets an attacker sign as the shop's bot | Web Bot Auth seed is a dedicated secret; `kid` is an RFC 7638 thumbprint, not configurable |
| THR-AGENT-012 | B1 | Elevation | Paid resource reached without payment | x402 off by default; default verifier never grants access; `HEAD` gated with `GET` unless methods are listed |
| THR-AGENT-013 | B2/B3 | Tampering | A channel **without a hostname** falls back to the request host even with `trust_request_host: false`; the result can be cached per channel | **Configuration requirement:** give every channel a hostname |
| THR-AGENT-014 | B1 | Spoofing | Consent phishing: a client registered with an attacker-controlled HTTPS redirect URI obtains a code once an admin approves | `allowed_redirect_uris` enforced at registration and at `/authorize` (exact match; loopback port-agnostic per RFC 8252 §7.3; empty list allows nothing); consent screen shows client and redirect target. **Mitigated in mcp-oauth v2.0.0** |
| THR-AGENT-015 | B1 | Repudiation | Agent actions not attributable | Every token is bound to an admin user and a client id; revocation endpoint |
| THR-AGENT-016 | B1 | Denial of service | Mass client registration fills the client table | **Operator:** rate-limit `/mcp/oauth/register` at the proxy |
| THR-AGENT-017 | B1 | Repudiation | Settlement failure after a paid response goes unnoticed | Settlement runs only on 2xx; failures are logged, never turned into an error for the payer |
| THR-AGENT-018 | B6 | Elevation | A `PaymentBypassVoterInterface` implementation that trusts request data lets anyone skip payment | Voters run after the firewall; a voter must base its decision on the security token, not on headers |
| THR-AGENT-019 | B1 | Tampering | 402 challenge echoes a `Host`-derived resource URL | Response is `no-store` and goes back only to the sender; accepted residual |
| THR-AGENT-020 | B1 | Elevation | A leaked access token stays usable far longer than configured: OAuth tokens lived 90 days, CLI tokens never expired | OAuth tokens live `access_token_ttl` (default 3600 s); CLI tokens expire after 90 days unless `--no-expiry` is given explicitly. **Mitigated in mcp-oauth v2.0.0** |
| THR-AGENT-021 | B1 | Denial of service | Consent template missing from the bundle: a fresh install fails at `/authorize`, and hosts write their own form without the CSRF field | Bundled, overridable consent template with CSRF field; translatable strings (`koh_mcp_oauth` domain). **Mitigated in mcp-oauth v2.0.0** |

## Agent-specific considerations

- **Prompt injection via discovery documents.** `llms.txt`, Markdown and
  `auth.md` carry shop-authored text that agents feed to models. The packages
  do not sanitise merchant content for model consumption; content authors are
  trusted (B5/B6).
- **Excessive agency.** An OAuth token carries the full authority of the
  administrator who approved it; the only scope is `mcp`. Least privilege is
  the operator's job: approve agents with a dedicated, restricted admin user.
- **Policy is advisory.** `robots.txt` and Content Signals state a policy; they
  do not enforce it against a crawler that ignores them.

## Out of scope by design

- Inbound bot authentication: `sylius-web-bot-auth` publishes keys for
  requests the shop *sends*; it verifies nothing that arrives.
- On-chain verification and settlement for x402 — the operator plugs in a
  verifier; the bundled one refuses every payment.
- Rate limiting and WAF rules — expected at the proxy.

## Findings carried to remediation

| ID | Package | Finding | Proposed resolution |
|---|---|---|---|
| THR-AGENT-013 | agent-contracts | Request-host fallback for channels without a hostname | Document as a hard requirement, or refuse to build URLs without a hostname unless `trust_request_host` is on |
| THR-AGENT-014 | mcp-oauth | `allowed_redirect_uris` is accepted as configuration but not enforced: any HTTPS host and any private-use scheme is accepted at registration | **Mitigated in v2.0.0:** allow-list enforced at registration and at `/authorize`; empty list allows nothing |
| THR-AGENT-020 | mcp-oauth | `access_token_ttl` (default 3600 s) is accepted but not used; tokens issued through the OAuth flow live 90 days, CLI tokens never expire unless `--ttl-days` is given | **Mitigated in v2.0.0:** OAuth tokens honour `access_token_ttl`; CLI tokens default to 90 days, `--no-expiry` is an explicit opt-in. Residual: tokens minted by 1.x keep their stored expiry — operators review `expires_at IS NULL` rows (README, "Upgrading to 2.0") |
| THR-AGENT-021 | mcp-oauth | The consent template `mcp/oauth/authorize.html.twig` is not shipped; the host application must provide it | **Mitigated in v2.0.0:** template shipped as `@KohSyliusMcpOAuth/oauth/authorize.html.twig`, English and German catalogues |

## Document control

| Field | Value |
|---|---|
| Owner | Kai Ole Hartwig |
| Version | 0.2 |
| Last review | 2026-09-28 |
| Next review | before the next MAJOR release of any package in scope, or when a trust boundary is added |
| Report a gap | see [SECURITY.md](../SECURITY.md) |
