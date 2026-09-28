<!--
SPDX-FileCopyrightText: 2026 Kai Ole Hartwig <mail@ole-hartwig.eu>
SPDX-License-Identifier: MIT
-->

# koh/sylius-agent-ready

Metapackage. Pulls in the full agent-readiness surface for a Sylius shop:
discovery documents, `llms.txt`, agent authentication and the agentic commerce
protocols, each as its own independently installable package.

## Install

```bash
composer require koh/sylius-agent-ready
```

## What comes with it

| Package | Serves |
|---|---|
| `koh/sylius-agent-contracts` | the shared registries and `koh:agent:check` |
| `koh/sylius-robots-txt` | `/robots.txt` with AI crawler rules and Content Signals |
| `stefandoorn/sitemap-plugin` | `/sitemap.xml` |
| `koh/sylius-llms-txt` | `/llms.txt`, `/llms-full.txt` |
| `koh/sylius-markdown` | `Accept: text/markdown` negotiation |
| `koh/sylius-api-catalog` | `/.well-known/api-catalog` and the `Link` headers |
| `koh/sylius-agent-skills` | `/.well-known/agent-skills/index.json` |
| `koh/sylius-a2a` | `/.well-known/agent-card.json` |
| `koh/sylius-mcp-oauth` | the OAuth 2.1 server and its RFC 8414 / 9728 metadata |
| `koh/sylius-auth-md` | `/auth.md` |
| `koh/sylius-web-bot-auth` | `/.well-known/http-message-signatures-directory` |
| `koh/sylius-acp` | `/.well-known/acp.json` and the agent product feed |
| `koh/sylius-mpp` | `/openapi.json` with `x-payment-info` |
| `koh/sylius-x402` | HTTP 402 payment middleware (off by default) |

Suggested, not required: `koh/sylius-mcp` (MCP tools and server card),
`koh/sylius-webmcp` (in-browser agents), `koh/sylius-ucp` (UCP profile).

## Configure — installing is not enough

A metapackage cannot register bundles or import routes. Three things stay
manual, and the shop is not agent-ready until all three are done.

**1. Register the bundles** in `config/bundles.php`.

**2. Import the routes.** One file, one entry per installed bundle:

```yaml
# config/routes/koh_agent.yaml — must load before api_platform.yaml
koh_robots_txt:    { resource: '@KohSyliusRobotsTxtBundle/config/routes.php' }
koh_llms_txt:      { resource: '@KohSyliusLlmsTxtBundle/config/routes.php' }
koh_api_catalog:   { resource: '@KohSyliusApiCatalogBundle/config/routes.php' }
koh_agent_skills:  { resource: '@KohSyliusAgentSkillsBundle/config/routes.php' }
koh_a2a:           { resource: '@KohSyliusA2aBundle/config/routes.php' }
koh_mcp_oauth:     { resource: '@KohSyliusMcpOAuthBundle/config/routes.php' }
koh_auth_md:       { resource: '@KohSyliusAuthMdBundle/config/routes.php' }
koh_web_bot_auth:  { resource: '@KohSyliusWebBotAuthBundle/config/routes.php' }
koh_acp:           { resource: '@KohSyliusAcpBundle/config/routes.php' }
koh_mpp:           { resource: '@KohSyliusMppBundle/config/routes.php' }
```

There is no `ignore_errors` in the routing loader — that is a DI import option.
A resource naming an uninstalled bundle is a hard failure, so keep this file in
step with `config/bundles.php`.

**3. Exempt the paths from the shop firewall**, above the Sylius entries:

```yaml
security:
    firewalls:
        koh_agent_public:
            pattern: '^/(\.well-known/(?!oauth)|llms\.txt|llms-full\.txt|auth\.md|robots\.txt|sitemap|openapi\.json)'
            security: false
        # … then the Sylius firewalls
```

`sylius.security.shop_regex` is a catch-all that matches `/.well-known/…` and is
stateful. Without this, every crawler hit starts a session and the responses
carry `Set-Cookie` — uncacheable, for documents whose whole point is to be
served from a shared cache. An `access_control` entry does not help; it is
evaluated only after a firewall has already matched.

Also **delete `public/robots.txt`**. The web server answers it from disk before
Symfony runs and silently shadows the dynamic route.

## Verify

```bash
bin/console koh:agent:check
```

Sends a request to every announced endpoint and reports what came back. It
catches the failure none of the unit tests can: a bundle registers an endpoint,
the discovery documents advertise it, and the route was never imported — so
agents are pointed at a 404 while every document looks correct.

It also flags any discovery response that sets a cookie, which means the
firewall exemption is missing.

## Decommission

Decommission the member packages first — several hold state that outlives the
code: revoke outstanding MCP access tokens before removing
`koh/sylius-mcp-oauth`, revoke the Web Bot Auth seed after removing
`koh/sylius-web-bot-auth`, and switch x402 off before removing it. Each
package's README has its own section. Then remove the `composer require`, the
bundle entries, `config/routes/koh_agent.yaml` and the `koh_agent_public`
firewall. Restore a static `public/robots.txt` if you want one.

## Security

Report vulnerabilities privately — see [SECURITY.md](SECURITY.md). Advisories
are published as CSAF 2.0 at <https://ole-hartwig.eu/.well-known/csaf/>.

The threat model for the whole package family is in
[docs/threat-model.md](docs/threat-model.md).

## Licence

MIT — see [LICENSE](LICENSE).
